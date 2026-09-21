# Channels

**A privacy-first anonymous forum and real-time virtual world.**
Built solo over roughly four months. Launched 5 July 2026 at `channels.zone`. Shut down August 2026.

**Status:** Archived · **Live page:** https://phoque52.github.io/channels-demo · **Walkthrough:** https://youtu.be/6gTw8FKWJUM

---

> **About this repository**
>
> This repo contains the preserved marketing site only — `index.html`, its stylesheets, fonts and media.
> The application source (Express server, WebSocket hub, SQLite layer, client engine) is **not published**.
> Everything below documents a system that is no longer online.

---

## Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Data model](#data-model)
- [Identity and authentication](#identity-and-authentication)
- [The forum](#the-forum)
- [Real-time hubs](#real-time-hubs)
- [Moderation and safety](#moderation-and-safety)
- [Security engineering](#security-engineering)
- [Privacy model](#privacy-model)
- [Release history](#release-history)
- [Why it ended](#why-it-ended)
- [What I would do differently](#what-i-would-do-differently)
- [Technology](#technology)

---

## Overview

Most forum software is threads and replies. Channels was a forum attached to a world you could
walk through — persistent 2D rooms rendered in the browser, with synchronized audio, video and
drawing shared live between everyone present.

Four principles were fixed from the start and never relaxed: **no ads, no algorithm, no real
name, no investors.** Every design decision downstream followed from them, including the ones
that eventually made the project unsustainable.

**By the numbers**

| | |
|---|---|
| Server-side code | ~2,650 lines across 12 modules |
| Client-side code | ~9,500 lines (rendering, audio, networking) |
| Database tables | 13, SQLite in WAL mode |
| WebSocket message types | 48 |
| Server tick rate | 30 Hz (33 ms) |
| Claimable homes | 1,800 across 150 cities in 10 countries |
| Official channels | 15, plus unlimited user-created |
| Development time | ~4 months, solo |
| Time live | 6 weeks |

---

## Architecture

A single Node process served everything: HTTP, static assets and the WebSocket world state.
No microservices, no message broker, no external cache. For the load this site realistically
faced, one process and one embedded database was the correct call — and it kept the monthly
cost to a single small VPS.

```
                     ┌──────────────────────────────┐
   Browser ───TLS───▶ │  Caddy (reverse proxy, TLS)  │
                     └──────────────┬───────────────┘
                                    │
                     ┌──────────────▼───────────────┐
                     │      Node / Express 5        │
                     │                              │
                     │  compression → helmet (CSP)  │
                     │  → body parse → session      │
                     │  → CSRF origin check         │
                     │  → rate limiters → guards    │
                     │  → static → routers          │
                     └──────┬────────────────┬──────┘
                            │                │
                   ┌────────▼─────┐   ┌──────▼────────┐
                   │  HTTP routes │   │  ws server    │
                   │  (6 routers) │   │  30 Hz tick   │
                   └────────┬─────┘   └──────┬────────┘
                            │                │
                     ┌──────▼────────────────▼──────┐
                     │  better-sqlite3 (WAL mode)   │
                     │  app data + session store    │
                     └──────────────────────────────┘
```

**Middleware order matters and is deliberate.** Session initialisation precedes every route
guard so `req.session` is always populated. Page guards are registered *before*
`express.static`, because static middleware would otherwise serve protected HTML directly and
the guard would never run — a subtle failure mode that is easy to introduce and hard to notice.

**Synchronous database access.** `better-sqlite3` is synchronous, which in a single-threaded
runtime sounds alarming. In practice queries against an embedded SQLite file complete in
microseconds, and the absence of callback or promise overhead made the real-time loop simpler
and more predictable than an async driver would have.

---

## Data model

Thirteen tables, created idempotently at boot. Migrations are applied as guarded `ALTER TABLE`
statements wrapped in `try/catch`, so the schema converges on startup regardless of which
version the database file was last written by.

| Table | Purpose |
|---|---|
| `users` | Username, Argon2id hashes, avatar JSON, bio, mod flag, ban timestamp |
| `channels` | Official (`c/`) and user-created (`uc/`) channels, soft-deleted |
| `threads` | Thread bodies, soft-deleted |
| `replies` | Replies, cascade-deleted with their thread |
| `mod_actions` | Append-only audit log of every moderator action |
| `notifications` | System and activity notifications |
| `notification_reads` | Per-user read state |
| `neighborhood_homes` | 1,800 home slots, ownership, expiry |
| `home_access` | Key-holder access lists per home |
| `home_furnishings` | Placed objects inside home interiors |
| `paintings` | Per-easel canvas images |
| `paintit_pixels` | The persistent shared canvas |
| `bug_reports` | In-app reports with status tracking |

Content is **soft-deleted** throughout — `deleted_at` rather than `DELETE` — so moderation is
reversible and the audit log stays meaningful.

The official channel list is **upserted from code on every boot**, so editing a channel's name
or motto in the source propagates to the database without a migration step.

---

## Identity and authentication

There is no email address, no phone number, no OAuth provider, and no password reset. An
account is three generated artifacts:

1. **A username** — adjective + noun + four digits, checked for collisions against the database
   with a bounded retry.
2. **A passphrase** — 16 bytes from `crypto.randomBytes`, base64url-encoded.
3. **Eleven passitems** — words drawn without replacement from a 180-word list.

All three are displayed exactly once, during signup. The passphrase and the joined passitem
string are hashed with **Argon2id** and the raw values are dropped from the server the moment
the final signup step completes. Sign-in requires the username, the passphrase *and* all eleven
passitems.

The consequence is absolute and was stated plainly on the signup page: **lose the credentials
and the account is gone.** There is no recovery path because there is no second factor of
identity to recover against.

**Timing-attack resistance.** When a username does not exist, sign-in still performs an
`argon2.verify` against a hardcoded dummy hash before failing. Without it, a non-existent user
would return measurably faster than a wrong password and the endpoint would leak which
usernames are registered.

**Sessions** are server-side, stored in the same SQLite file, four-hour rolling expiry, cookie
flagged `httpOnly` / `sameSite=lax` / `secure` in production. Expired sessions are swept every
15 minutes.

**Ban enforcement** is checked against the database on *every* authenticated request rather than
trusted from the session, so a ban applied mid-session takes effect on the user's next action
instead of whenever their cookie happens to expire.

---

## The forum

Two namespaces: `c/` for the fifteen official channels, `uc/` for user-created ones. Any account
could create two user channels per day.

Threads and replies support `@mention` autocomplete, backed by a prefix-match endpoint that
escapes `%` and `_` before interpolating into a `LIKE` pattern — otherwise a username containing
a wildcard would match far more than intended.

Posting is rate-limited per action type. Threads, replies and channel creation each carry their
own limiter, so a user who hits the thread limit can still reply.

---

## Real-time hubs

The hubs are the part of this project I am most satisfied with. A single `ws` server shares the
HTTP server's port, maintains authoritative state for every room in memory, and broadcasts
deltas on a **30 Hz tick**.

Rooms are created lazily and keyed by id. Each holds a `Map` of connected users, per-slot
instrument assignments, and a bounded chat history. Movement is server-authoritative: the client
sends an intent, the server clamps it against room geometry — including stage boundaries and
staircase corridors — and the resulting position is what propagates to everyone else.

The protocol carries **48 distinct message types**, covering position and avatar sync, chat and
emotes, instrument note on/off, cinema playback state, canvas strokes and pixel placement, room
object updates, NPC dialogue, gateways between rooms, and home access control.

### `h/music`

Five stages, six instruments — piano, guitar, bass, violin, saxophone and drums — with a
**capacity of three simultaneous performers** per stage. Note events broadcast to every listener
in the room.

Instruments are **synthesized in the browser with the Web Audio API**, not sampled: oscillators
routed through gain envelopes and biquad filters. No audio files ship to the client, so an
instrument costs nothing to load and its timbre is tunable in code.

### `h/cinema`

Five rooms. One participant hosts; the server holds the authoritative playback state and the
YouTube iframe on every other client seeks to the host's timestamp. Late arrivals join at the
correct position rather than from the beginning.

### `h/paintit`

A persistent 500×500 shared canvas. One pixel per user per 60 seconds, sixteen colors, cooldowns
tracked in server memory. Pixels persist to the database; new arrivals receive a snapshot and
then live deltas.

### `h/neighborhood`

Ten countries of fifteen cities each, twelve homes per city — **1,800 claimable homes**. Homes
are addressed by a page id that maps deterministically to a country and city name, so the address
space is generated rather than stored.

A home is yours while you keep returning. An **hourly expiry job** releases homes whose owners
have gone inactive, clears their access lists, notifies anyone standing inside at that moment,
and opens the next page of homes when the current one fills.

Interiors are furnishable with working objects — instruments, a canvas, a television — and access
is key-based: the owner grants named users entry, and everyone else is refused at the door.

---

## Moderation and safety

Moderators are a flag on the user row, verified against the database on every privileged request.
Every moderator action writes to an append-only `mod_actions` table recording who acted, what
they acted on, and an optional note.

Users file bug reports from inside the app across eight categories, rate-limited to five per hour
**keyed by user id rather than IP** — so users behind a shared address do not exhaust each
other's quota. Reporters receive a notification when a moderator resolves their report.

---

## Security engineering

| Control | Implementation |
|---|---|
| Password hashing | Argon2id, raw values never persisted |
| Username enumeration | Dummy-hash verification on unknown users |
| Session storage | Server-side, SQLite-backed, 4h rolling, auto-swept |
| CSRF | Origin/host comparison on all state-changing methods |
| CSP | Strict `helmet` policy; YouTube explicitly allowlisted |
| SQL injection | Prepared statements throughout; `LIKE` wildcards escaped |
| Rate limiting | Per-endpoint: sign-in 10/15 min, signup 10/h, reports 5/h |
| Ban enforcement | Re-checked from the database on every request |
| Startup safety | Process refuses to boot without `SESSION_SECRET` |
| Transport | TLS terminated at Caddy; `trust proxy` scoped to one hop |

The CSRF check deliberately allows requests with **no** `Origin` header or an opaque `null`
origin, since browsers send those for legitimate redirect chains and sandboxed contexts, while
rejecting any origin whose hostname does not match the request host.

---

## Privacy model

What the server stored: a generated username, two Argon2id hashes, content the user created,
an avatar configuration, and a server-side session token. Nothing that connects to a real
identity.

**IP addresses were used only for rate limiting.** The counters lived in server memory and were
never written to an account record. There was no analytics package, no ad network, no tracking
cookie, and no third-party script beyond the YouTube embed required by the cinema hub. Fonts
were self-hosted specifically to avoid leaking visitor requests to a font CDN.

This was stated on the front page in the same detail it is stated here, including its limits.

---

## Release history

| Version | Date | Highlights |
|---|---|---|
| **v1.2** | never shipped | Home interiors rebuild, in progress when development stopped |
| **v1.1** | 13 Aug 2026 | Housing system, estate office, notification system |
| **v1.0** | 5 Jul 2026 | Mobile support, neighborhood, transportation, border guards, instrument mechanics and designs |
| **v0.5** | 30 Jun 2026 | User profiles, hub lobbies, NPCs, spam prevention, free shared canvas |
| **v0.1** | 21 Jun 2026 | Initial release |

---

## Why it ended

**No revenue was possible by design.** The site's own pillars were no ads, no algorithm, no
investors, donations only. A product whose charter forbids revenue cannot be a business, and
cannot be taken to investors without breaking the promise on its own front page. It ran for six
weeks on a VPS paid out of pocket.

**Zero organic users after launch.** This was not a marketing failure. It was structural, and
there were three causes:

1. **Nothing was visible from outside.** All threads, hubs and neighborhood content sat behind
   the signup wall. Nothing was indexable, shareable, or evaluable before committing.

2. **Signup was maximum-friction.** Generated credentials, eleven words to write down on paper,
   no recovery ever — asked of a visitor who had not yet seen a single post.

3. **Real-time social features need density.** Empty music stages and 1,800 empty homes are
   worse than not having the feature at all.

Each decision was defensible in isolation. Together they made growth close to impossible.

---

## What I would do differently

**Make something public.** A read-only view of `c/` channels, indexable and shareable, would
have cost little privacy and given search engines and link-sharers something to point at. The
signup wall protected users who never arrived.

**Stage the friction.** Let someone read first, create an account second, and defer the
eleven-passitem ceremony until they have a reason to protect the account. The security model
was sound; presenting all of it before any value was the error.

**Seed density, or cut it.** Real-time features should have launched in one room at one
announced time, rather than across five stages and ten countries that were always empty.

**Use a CSPRNG everywhere.** The passphrase correctly uses `crypto.randomBytes`, but username
and passitem selection use `Math.random()`. Passitems are a credential factor and deserve the
same treatment — a real flaw I would fix before this system ever held anything valuable.

**Decide the business model before the charter.** "No revenue" is a coherent position for a
hobby. Writing it onto the front page of something intended to last made it unfixable later
without breaking a public promise.

---

## Technology

**Runtime** — Node.js, Express 5
**Database** — SQLite via `better-sqlite3`, WAL journaling
**Real-time** — `ws`, sharing the HTTP server port
**Auth** — `argon2` (Argon2id), `express-session` with a SQLite store
**Security** — `helmet`, `express-rate-limit`, custom origin-based CSRF
**Client** — Vanilla JavaScript, Canvas rendering, Web Audio API. No framework, no build step.
**Fonts** — Instrument Serif, Space Grotesk, self-hosted
**Infrastructure** — Single VPS behind Caddy for TLS and reverse proxying

There is no bundler and no dependency on a front-end framework. The client is plain ES, served
directly, and the entire application ships as static files plus one Node process.

---

## Contact

Questions about how any part of this was built are welcome — **channelssupport@proton.me**

---

<sub>Channels ran from 5 July to August 2026. Archived and documented for reference.</sub>
