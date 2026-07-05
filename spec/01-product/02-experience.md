# Experience

What using Orbit feels like. Orbit runs on the desktop, in a browser tab, on phones, and embedded inside other websites, and the experience is the same product everywhere with each surface playing to its strengths.

## Joining

Entry is frictionless. A friend gets a link, opens it in a browser, picks a name, and is in the call. No account, no install, nothing beyond the browser's microphone prompt. Anonymous guests can chat and join voice; the operator decides how much further a guest can go (file uploads, for example, are off for guests by default).

Accounts are for people who stay. One login through the community's identity provider carries across chat, voice, and files, and operators choose the login experience (password, SSO, passkeys, MFA). Verified users are visually distinct from guests, so everyone can tell who's who at a glance.

Because Orbit is decentralized, there's no central invite service. A link to a server or channel *is* the invite; access control stays with the community.

## Text Chat

Channels are lightweight and communities organize them into collapsible folders just by naming them with slashes (`#dev/frontend` renders as `frontend` inside a `dev` group). History is there when you come back, and read positions sync across devices, so the phone knows what you read on the desktop.

Messages support replies with an inline excerpt of the original, retraction (removed messages leave a tombstone, never the content), reactions, link previews, image thumbnails, emoji, and basic Markdown. In-place editing is the one gap: IRC hasn't standardized it yet, so Orbit waits for the standard, with client-side edit tags as the fallback if none lands in time. Unread indicators and mention highlights work as you'd expect, and messages composed while disconnected are held and sent when the connection returns.

### Threads

A message that grows a conversation gets a thread indicator beneath it. Clicking opens a thread panel: the original message at the top, replies in order, and a compose box. The user sees a reply input, never the mechanics underneath. People on plain IRC clients can read and reply to the same thread from their own client, so threading never locks out the rest of the ecosystem.

## Voice and Video

Joining voice means picking a session in the current channel and clicking in. You see who's already there before you join. Inside a session: mute and deafen, per-user volume, voice activity indicators, push-to-talk, webcam toggles, screen sharing, and a grid layout for small groups or a speaker-focused layout for larger ones. A session also has its own throwaway chat for quick callouts and links, which disappears when the session ends.

Whoever starts a session runs it. The creator can mute and kick, set a password, lock the room, restrict who can join, and delegate moderation to others. Someone turned away at the door can knock; the host admits or ignores. Multiple sessions can coexist in one channel, so sub-groups can split off without ceremony. If you don't like how a room is run, make your own.

Voice doesn't require the community's infrastructure at all:

- **Quick calls from a link.** A session link shared over text or email is enough. The recipient opens it and joins, no chat server and no account involved.
- **Bring your own voice server.** If a community hasn't set up voice, anyone can point the client at their own server and invite the channel to it. Those sessions carry a "Community" label instead of the verified badge, and the client asks for confirmation before connecting to an unfamiliar one.
- **1:1 calls** connect the two people directly, and can escalate on the fly: voice to video to screen share to sending a file.

## Files and Media

Drop a file into chat and it appears inline: images, audio, and video preview directly in the conversation, while people on plain IRC clients see an ordinary URL. Uploads from signed-in users are attributed and removable, with per-user quotas under the operator's control. Power users can mint an upload key and share screenshots straight from command-line or capture tools.

## Where Orbit Runs

### Desktop

The full experience. System tray with unread and mention badges, native notifications, `orbit://` deep links that open straight into a channel or call, fine-grained audio device control, and the largest on-device message archive, bounded only by disk. This is the client for the person who lives in the app.

### Web App and PWA

The same application in a browser tab, and the front door for most people. It installs as a PWA: its own window without browser chrome, an app shell that loads instantly on repeat visits, and messages queued offline that send when connectivity returns. The installed PWA and the desktop client coexist; both are supported paths.

What works where:

| Capability | Desktop | Web / PWA | Embedded |
|---|---|---|---|
| Text chat and history | Yes | Yes | Yes, recent messages |
| On-device message archive | Yes, disk-bounded | Yes, browser-managed and best-effort | No, session only |
| Message retractions | Yes | Yes | Yes |
| Message editing | Not yet: no IRC standard | Not yet | Not yet |
| Rich rendering | Yes | Yes | Yes |
| File uploads | Yes | Yes | No (guest surface; operator-configurable) |
| Voice and video | Yes | Yes | Yes |
| Offline message outbox | Yes | Yes | No |
| Notifications | Native OS | Browser notifications | No |
| Tray and badges | Native tray | Tab title and favicon | Not applicable |
| Deep links | Opens `orbit://` links | Link hand-off to desktop | "Open in Orbit" button |
| Installable | Native app | PWA install | Not applicable |
| Settings and server browser | Yes | Yes | Hidden |
| Audio device selection | Yes, fine control | Yes, device picker | Browser default |

### Account and Identity Surface

Identity is a product surface, not a settings page of arcana. The client never shows raw IRC service commands; it surfaces intent:

- An account without a verified recovery email gets a non-blocking prompt and an in-app flow to claim it.
- Whether you stay reachable for DMs while offline is shown in plain language, with a warning when you're not.
- A single toolbar indicator flags actionable identity state, so you're nudged once, not nagged.

### Mobile

People expect to get messages and join calls on their phone; without that, a communication platform doesn't stick past early adopters. Today the web app covers mobile: it installs as a PWA on Android and iOS, and chat, voice, and video work. A native mobile app built from the same codebase is designed for the things a PWA can't reach, app store presence, reliable notifications, and voice that survives backgrounding, but it isn't built yet. Mobile also needs its own interaction design (bottom navigation, gestures) rather than a shrunken desktop layout; that design work hasn't happened, which is why this section is thinner than the desktop one. The client design lives in [Clients](../02-architecture/11-clients.md).

### Embedded

A community's chat can live inside any website. The embedded client is a compact single-channel view in an iframe: no sidebar, no settings, just the conversation, with an "Open in Orbit" button as the escape hatch to the full app. Guests can join and speak in voice by default, and the operator can restrict that. Because it's the same application as the web app, embedded users get every fix and improvement automatically.

Embedders can match their site with a small set of theme options: light or dark mode, accent color, corner radius. Deeper customization means self-hosting the web app. The embed snippet and theming parameters live in [Clients (implementation)](../03-implementation/08-clients.md).

## Finding Communities

The client ships with a Browse Communities view backed by the public directory at `orbit.directory`. Users search public communities by name, description, tags, region, and language, and connect directly from the listing. A listing shows the community's name and description, its domain, an approximate member count, which services it runs (chat, voice, files), and operator-provided region and language tags. Listing is self-service for operators and strictly opt-in; a community that never registers works exactly the same. The directory model lives in [Infrastructure](../02-architecture/12-infrastructure.md).

## Privacy Expectations

Channels and standard DMs are readable by the operator of the server you joined, the same trust model IRC has always had: choose your server accordingly. DMs travel in plaintext until end-to-end encryption is built; 1:1 calls are already end-to-end encrypted by construction. The full threat model and the E2E boundary live in [Messaging](../02-architecture/10-messaging.md).

## Your Message History

Every client keeps an on-device copy of the messages it has seen, so switching channels is instant, scrollback works offline, and history outlives the server's retention window on the devices that saw it. Users see and control this in Settings, under Storage: per-conversation size and date range, eviction, a JSON export for a user-owned archive, and a configurable size cap. The mechanics live in [Local Cache](../03-implementation/07-local-cache.md).
