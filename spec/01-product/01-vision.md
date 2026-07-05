# Vision

## What Orbit Is Solving

The problem with every Discord alternative is that they try to out-Discord Discord. They build custom protocols, custom servers, custom permission systems, and then spend years catching up to feature parity while Discord moves further ahead. None of them work everywhere. None of them are easy to self-host. None of them make it trivial to write a bot or integrate with external tooling.

Discord won for one reason above all others: it works everywhere. Browser tab, desktop app, mobile, embedded on a website. TeamSpeak is better audio. Mumble is lighter. IRC has more history and more tooling. None of them run in a browser tab without friction. Discord does, and that's why communities migrated.

Orbit closes that accessibility gap, on a foundation communities can actually own.

## The Foundation

IRC has been running communities reliably for over thirty years. The protocol is open, well documented, and owned by no one. Every language has mature IRC libraries, bot frameworks have existed for decades, and the server software is battle-tested at scale on minimal hardware.

The missing piece is the client. Orbit is that client: a desktop application, a web app, and an embedded client any website can drop in. All of them are backed by the same IRC server you could have been running for years. The protocol and its ecosystem are unchanged. What changes is that a non-technical user can now join a community from their browser, and a website owner can embed a live community chat without standing up anything new.

Orbit doesn't fork the IRC server and isn't in the business of reimplementing IRC. How Orbit relates to the protocol, its gaps, and the no-fork rule is covered in [Protocol Posture](../02-architecture/02-protocol-posture.md).

## Where Orbit's Value Lives

Orbit isn't trying to beat Discord. It's trying to make defection cheap, pleasant, and reliable, so that when people are ready to leave, the door is already open.

Everything your friend group needs from Discord, privately, on a $5 VPS. `docker compose up`. For less than $10 a month you get voice, text, files, and identity, all self-hosted, all yours. Every alternative that died tried to out-Discord Discord and failed because it wasn't decoupled, scaled poorly on cost, and was horrible to maintain. Orbit survives by not owning what it doesn't have to.

The way Orbit wins on the client side is UX. People stay on Discord because it's easy. The bar: make leaving take an afternoon instead of a research project. Anonymous access, invite a friend to a call without them making an account, works in the browser, works on a phone. The onboarding is gone; the product demonstrates itself.

IRC already does the "decentralized but nobody cares" thing naturally. An Orbit client can connect to any existing IRC network, so a community can adopt Orbit without a migration. Orbit's value is the whole experience: a polished client layer together with Satellite (voice and video) and Depot (file storage), orchestrated into one product. The work is UX and integration, making a thirty-year-old protocol feel modern and effortless.

Orbit is not a product to sell. It's infrastructure and a cultural push: resistance to surveillance and platform lock-in. If Orbit ever provides hosted instances, that's a small convenience cut for people who don't want to operate their own box, not a SaaS play.

## Portability Over Permanence

IRC moderation at scale isn't hypothetical. Freenode ran roughly 90,000 concurrent users for twenty years with channel ops, services, bots, and network bans, and Libera continues that model today. What actually killed Freenode wasn't a moderation failure. It was a governance and ownership failure, the 2021 hostile takeover, and everyone migrated to Libera in a week.

That cuts in Orbit's favor. When your platform is built on a standard protocol with no lock-in, what would be a death for a centralized service becomes a one-week migration. Portability over permanence is a stronger defense than moderation features alone. How moderation itself is layered is covered in [Messaging](../02-architecture/10-messaging.md).

## Reading On

- [Experience](02-experience.md) covers what using Orbit feels like, surface by surface.
- [Scope](03-scope.md) covers what Orbit deliberately is not.
- [Platform Comparison](04-platform-comparison.md) answers "why not just use Discord, Slack, or Teams."
- [Architecture Overview](../02-architecture/01-overview.md) is where the how begins.
