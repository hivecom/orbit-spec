# Satellite Gateway

## Problem

The Satellite is a single node - one LiveKit instance and one token service. This works well for small and mid-sized communities, but a single node has a hard ceiling: one server's bandwidth and CPU. When a community outgrows that ceiling, the operator needs to add nodes. The moment there are multiple nodes, a new problem emerges: **routing**.

A client hitting `/session/create` needs to land on a node with available capacity. A client hitting `/session/join` needs to land on the node that actually hosts the target room. The `+orbit/sat-invite` tag embeds the Satellite URL, and in a single-node deployment the Satellite *is* that node - so routing is trivial. With multiple nodes, something needs to sit in front and make routing decisions. That something is the **gateway**.

## Design

A thin routing layer - the Satellite gateway - sits in front of a pool of LiveKit nodes and exposes the same API surface as a single-node Satellite. Clients see no difference. The gateway absorbs the token service role and adds room-to-node routing.

### Responsibilities

| Concern | How |
|---|---|
| **`/info`** | Aggregates room and participant data from all nodes. Returns a unified view - clients never see individual nodes. |
| **`/session/create`** | Picks the least-loaded node, creates the room there, records the room-to-node mapping, returns a token. The token contains the specific node's WebRTC endpoint so the client connects directly to the right node for media. |
| **`/session/join`** | Looks up which node hosts the target room, issues a token pointing at that node. |
| **`/session/knock`**, **`/session/lock`**, etc. | Proxied to the correct node based on the room-to-node mapping. |
| **Health checking** | Periodically polls each node. Unhealthy nodes stop receiving new rooms. If a node becomes unresponsive, its rooms are marked as lost - participants are disconnected and the client shows "Voice session ended unexpectedly." |
| **Drain coordination** | On scale-down, the gateway stops routing new sessions to a target node. Once all rooms on that node have naturally ended, the node is retired. |

### Room-to-Node Mapping

The gateway tracks which rooms live on which nodes. This is a small, fast-changing dataset (room ID → node URL). Options:

- **In-memory hash map** - simplest. Lost on gateway restart, but rooms are ephemeral - a gateway restart means all routing state is rebuilt by querying each node's LiveKit API for active rooms.
- **Redis / shared store** - needed only if the gateway itself is horizontally scaled (multiple gateway instances). Likely overkill for most deployments.
- **LiveKit's native room list API** - the gateway queries each node's LiveKit API on startup and periodically to rebuild its routing table. This makes the gateway stateless - it derives routing from the nodes themselves.

The stateless approach (derive from nodes) is preferred. The gateway becomes a pure routing function with no state of its own - if it restarts, it re-queries nodes and is back in seconds.

### Load Balancing

"Least-loaded" for `/session/create` needs a metric. Options:

| Metric | Pros | Cons |
|---|---|---|
| Participant count | Simple, easy to query from LiveKit API | Doesn't account for video vs. audio-only load |
| Room count | Even simpler | A room with 2 people and a room with 50 people are not equivalent |
| Bandwidth utilization | Most accurate reflection of actual load | Requires node-level metrics export, more complex |
| CPU utilization | Good proxy for encryption/forwarding load | Requires node-level metrics, OS-dependent |

The initial implementation uses **participant count** - it's available directly from LiveKit's API with no extra instrumentation. This can be refined to bandwidth- or CPU-aware routing if participant count proves too coarse.

### Client Transparency

The gateway MUST be invisible to clients. The API contract is identical to a single-node Satellite:

- Same endpoints: `/info`, `/session/create`, `/session/join`, `/session/knock`, `/session/admit`, `/session/lock`
- Same response shapes
- Same `satellite://` links - the link points at the gateway, not at individual nodes

The only new element in the response is the WebRTC endpoint URL embedded in the LiveKit JWT, which points at the specific node hosting the room. The client connects to this URL for media - but this is standard LiveKit behavior, not a gateway-specific concern.

## Scaling Model

```text
Scale up:
  Load increases -> gateway detects nodes near capacity ->
  operator (or K8s autoscaler) adds a new node ->
  gateway discovers it (DNS, config reload, or K8s service discovery) ->
  new /session/create requests route to the new node

Scale down:
  Load decreases -> gateway stops routing new sessions to the least-busy node ->
  existing rooms on that node drain naturally (participants leave) ->
  once the node has zero rooms -> node is retired (pod terminated, container stopped) ->
  at least one node always remains active

Node failure:
  Gateway health check detects unresponsive node ->
  node is removed from routing pool ->
  rooms on that node are lost - no migration, no failover ->
  clients see "Voice session ended unexpectedly" ->
  users create a new session (routed to a healthy node)
```

Room migration (moving an active room from one node to another) is explicitly out of scope. LiveKit sessions are stateful - WebRTC connections are tied to a specific server. Moving a room would require disconnecting and reconnecting every participant, which is indistinguishable from the room crashing. If a node needs to be retired while rooms are still active, the operator waits for them to drain.

## Kubernetes Integration

K8s is the natural deployment model for multi-node Satellites:

- **Nodes as pods** in a StatefulSet or Deployment. Each pod runs LiveKit + a lightweight health endpoint.
- **Gateway as a Deployment** (1–2 replicas) with a Service fronting it. The DNS SRV record points at the gateway Service.
- **Autoscaling** via HPA (Horizontal Pod Autoscaler) based on aggregate participant count or custom metrics.
- **Drain on scale-down** via `preStop` hooks - the gateway stops routing to the pod, waits for rooms to drain, then the pod terminates.
- **STUNner** for NAT traversal - integrates with K8s networking for TURN relay.

K8s is not required. The gateway works equally well with a static set of nodes defined in a config file - Docker Compose with 3 LiveKit containers and 1 gateway container is a valid multi-node deployment.

## Risks

- **Gateway as single point of failure.** If the gateway goes down, no new sessions can be created or joined. Active sessions continue (media flows directly between clients and nodes), but no new participants can join. Mitigation: run 2 gateway replicas behind a load balancer. Since the gateway is stateless (derives routing from nodes), replicas don't need to coordinate.
- **Latency on `/session/join`.** The gateway adds one extra hop - client → gateway → node. For the HTTP token exchange this is negligible (sub-millisecond if co-located). Media is unaffected - clients connect directly to nodes for WebRTC.
- **LiveKit Cloud overlap.** LiveKit offers a hosted, managed multi-node solution. Operators willing to use LiveKit Cloud instead of self-hosting get routing for free - LiveKit Cloud handles it internally. The gateway serves fully self-hosted multi-node deployments where operators want to keep all infrastructure under their own control.

## Validation Criteria

1. A gateway routing between 2–3 LiveKit instances correctly places and locates rooms.
2. Client transparency - an Orbit client connected to the gateway behaves identically to one connected to a single node. No client changes required.
3. Scale-up: add a node while sessions are active, verify new sessions route to it.
4. Drain: stop routing to a node, verify existing sessions continue, verify the node is retired once empty.
5. Failure: kill a node, verify the gateway removes it from routing and clients on that node see a clean error.
6. Overhead: gateway adds <5 ms to `/session/create` and `/session/join` latency.

## Dependencies

- Stable single-node Satellite
- `/info` endpoint returning room-level data (specified in [Satellite - Architecture](../02-components/02-satellite.md#architecture))
- LiveKit's room list API for stateless routing table derivation

## Timeline

The gateway is built when communities outgrow single-node capacity. The single-node Satellite ships first and establishes the API surface that the gateway preserves. Once operators need horizontal scaling, the gateway is the path forward - no client changes, no protocol changes, just a new layer in front of the same nodes.
