# IRC Compatibility Profile

This page documents what standard IRC clients experience when connecting to an Uplink server (Ergo in the MVP, the Uplink fork in the next step). The goal is to set honest expectations: standard IRC clients are fully supported and get a great experience, but some Orbit-specific features are invisible to them.

## Connection and Authentication

Standard IRC clients connect identically to how they connect to any IRCv3 server. SASL PLAIN and SCRAM-SHA-256 are supported. SASL ANONYMOUS is supported for guest access. No Orbit-specific handshake is required.

## Feature Compatibility Table

| Feature | Orbit Client | IRC Client (modern, IRCv3) | IRC Client (basic) |
|---|---|---|---|
| Text chat | ✓ Full | ✓ Full | ✓ Full |
| Message history (`chathistory`) | ✓ Full | ✓ Full | Limited (no `chathistory` support) |
| Message retractions | ✓ Tombstone rendered | ✓ Message removed (via `draft/message-redaction`) | Sees `*** alice retracted a message ***` NOTICE |
| Message editing | Post-Uplink | Post-Uplink | Post-Uplink |
| Threads | ✓ Thread panel UI | Can `/join` thread channel manually | Can `/join` thread channel manually |
| Voice / video | ✓ Full | Not available | Not available |
| File uploads | ✓ Inline preview | Sees plain URL | Sees plain URL |
| User avatars | ✓ Rendered (via Metadata) | ✓ Accessible (via `METADATA GET`) | Not available |
| Display names | ✓ Rendered (via Metadata) | ✓ Accessible (via `METADATA GET`) | Not available |
| Presence (online/away) | ✓ Rich (via Metadata + away-notify) | ✓ Basic (AWAY status) | ✓ Basic (AWAY status) |
| Read markers | ✓ Synced across devices | ✓ If client supports `draft/read-marker` | Not available |
| DMs | ✓ Full history, E2E optional (post-MVP) | ✓ Standard IRC PRIVMSG with history | ✓ Standard IRC PRIVMSG |
| Group DMs | ✓ Invite-only private channel | ✓ Standard IRC private channel | ✓ Standard IRC private channel |
| Satellite invites | ✓ Rendered as join button | Sees TAGMSG (invisible if no tag support) | Not visible |
| P2P call invites | ✓ Rendered as call prompt | Sees TAGMSG (invisible if no tag support) | Not visible |

## What IRC Clients Cannot Do

- **Initiate voice/video sessions** - Satellite is WebRTC-based. IRC clients can be invited to a session via a `satellite://` link shared in the channel, which they can open in a browser.
- **See inline file previews** - they see a plain URL, which is fully functional to open.
- **Render thread UI** - they can join thread channels directly (`/join #dev/t-abc123`) and read/reply normally.
- **Edit messages** - post-Uplink feature.

## What IRC Clients Can Do That Orbit Clients Do

Everything IRC does well: text chat, channel modes, SASL auth, DMs, bots, ZNC/bouncer support, scripting. An IRC client connecting to Uplink is a first-class participant in the text layer. The spec does not treat IRC clients as second-class.

## Bots

Any IRC bot - written in any language, using any IRC library - connects to Uplink and works immediately. Bots are first-class citizens. They can moderate channels, post reminders, respond to commands, and trigger Satellite session announcements. No Orbit-specific SDK is needed.
