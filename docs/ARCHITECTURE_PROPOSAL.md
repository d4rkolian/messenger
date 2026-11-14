# Architecture proposal: block-resistant retro messenger

Status: **draft for team discussion.** It proposes changes to `PLAN.md` and does not replace it.

Goals: a messenger for desktop (Windows, Linux, macOS), mobile (iOS, Android, native) and web. It must be resilient to blocking (like Telegram in Russia) and have a late-90s / early-00s vibe inspired by ICQ.

Caveat: knowledge of current Russian blocking techniques may be out of date. Verify it with a research pass before committing to details.

---

## 1. Where `PLAN.md` and the repo disagree

- `PLAN.md` describes a P2P network (go-libp2p, DHT), a Tauri desktop app with a local Go sidecar, and Tauri/Capacitor mobile builds.
- The repo contains a central setup: Postgres, Mosquitto (MQTT) and nginx on `icq.works`.
- Native mobile conflicts with the Tauri/Capacitor mobile plan.
- Nothing is a hard requirement, so the architecture is open.

## 2. Opinion on P2P and MQTT

- **Drop pure P2P for the MVP.** Mobile OSes kill background sockets. NAT traversal, DHT churn and offline delivery are open-ended problems. Offline delivery needs store-and-forward anyway, and Sprint 4 of `PLAN.md` already concedes this with the "blind push gateway" and "P2P-lite" modes.
- **Do not make MQTT the core.** It lacks message history, per-conversation delivery guarantees and read receipts. It would be a second system to secure.
- **Fix the infra first.** Mosquitto has `allow_anonymous true` and port 1883 is published to the internet. nginx serves HTTP only (no TLS).
- **Assign UINs on the server (or from keys), not by hashing.** A 6-7 digit hash collides quickly (birthday bound at roughly 4,000 users for 10M values).

## 3. Threat model

Blocking works at these layers:

1. **IP and domain blocks.** A single domain on a single server dies on day one.
2. **DPI (the TSPU boxes at ISPs).** They fingerprint and throttle protocols such as WireGuard, OpenVPN and QUIC, and sometimes any unknown TLS pattern.
3. **App store removal.**
4. **Mobile "whitelist" shutdowns.** Only approved domains and IPs work. This is the hardest case.
5. **Phone-number identity.** It gives the state leverage over users.
6. **Server seizure or legal pressure.**

No design beats all of these. The goal is to make blocking expensive and whack-a-mole, and to fail gracefully.

## 4. Architecture for resilience

### 4.1 Identity is a key, not a server or phone number
- A UIN is a number bound to a public key. No phone number or email.
- Identity is portable: if a server dies, you re-home your UIN to another server and contacts follow.

### 4.2 Many small relays
- Relays are dumb stores of encrypted blobs. Anyone can run one (XMPP/Matrix-like).
- Clients keep a list of relays and fail over between them. Some relays are public and some are invite-only, like Tor bridges.
- Start with a small trusted relay set run by us. Open federation brings spam and abuse problems, so defer it.

### 4.3 Pluggable transports in the client
- The client dials through several disguises and picks whatever works:
  - WebSocket over TLS behind a CDN.
  - TLS camouflage that mimics a real site (REALITY/XTLS style).
  - A browser-like TLS fingerprint (uTLS).
  - QUIC where it works, plus a fallback chain.
- Go fits well. Xray, sing-box, uTLS, Tor's Lyrebird/Snowflake and Psiphon are Go projects. Reuse proven tech instead of inventing protocols.

### 4.4 Bootstrapping (how a new client finds relays)
Use several channels at once:
- A built-in relay list.
- Signed configs fetched from several places (DNS TXT/DoH, GitHub, cloud buckets, a channel in another messenger).
- Invite links and QR codes that carry a relay address.
- Updates signed with a pinned key, so a hijacked source cannot poison the list.

### 4.5 End-to-end encryption from day one
- Relays must not see content, and seizing one must not expose anything.
- Use a vetted protocol (Signal double ratchet via libsignal, or MLS). Do not roll our own.
- Minimize metadata (for example sealed sender).

### 4.6 Distribution without app stores
- Android: direct APK, F-Droid, mirrors.
- Web/PWA: cannot be removed from a store and can be mirrored on many domains. On iOS it is the most robust option, with TestFlight and alternative stores as extras.
- Desktop: plain downloads.

### 4.7 Push
- Android: persistent socket (foreground service) and UnifiedPush as the main path.
- iOS needs APNs. Accept the limitation and keep payloads blind.

### 4.8 Last resort (post-MVP)
- Local mesh over Bluetooth and Wi-Fi Direct (like Briar or Bridgefy).
- Relays behind infrastructure that stays whitelisted.
- The architecture should not make these impossible.

## 5. Client and server stack

```
Clients (iOS / Android / Web / Tauri desktop)
   |  native UI shells; shared Go core on mobile and desktop (gomobile)
   |  transports (4.3), E2EE, sync, local storage
   v
Go relay (single binary to start)
   |- accounts/keys, UIN registry
   |- WebSocket gateway, presence
   |- encrypted offline queue (per-conversation seq, ack)
   '- Postgres or SQLite
```

- **iOS:** Swift + SwiftUI. **Android:** Kotlin + Jetpack Compose. **Web:** a small TypeScript SPA. **Desktop:** Tauri wrapping the web app.
- **Shared Go core via gomobile.** It removes the cost of writing sync, storage, transports and crypto three times. Trade-offs: gomobile binding ergonomics, binary size, battery.
- **Crypto spike needed.** libsignal is Rust, and the Go ports are less battle-tested. Decide between libsignal via FFI and MLS before committing.
- **Delivery model.** The relay assigns a monotonically increasing per-conversation `seq`. Clients ack and can re-sync with "everything after seq N". This covers offline, reconnects and multiple devices.
- **Protocol.** Define it once in a schema (Protobuf or a documented JSON spec) and generate types for Go, Swift, Kotlin and TypeScript.
- **Go libraries (suggestion).** `net/http` + `coder/websocket`, `pgx` + `sqlc`, `golang-migrate`. Avoid heavy frameworks.

## 6. Retro vibe

The vibe and the resilience reinforce each other:

- **UIN with no phone number.** Historically accurate and also protects users.
- **ICQ statuses:** Online, Away, N/A, Occupied, DND, Free for Chat, Invisible. "Invisible to specific users" is a real privacy feature.
- **Authorization requests** ("X wants to add you"), which also work as an anti-spam gate.
- **Contact list** with groups, offline messages, away messages, user info pages.
- **Sounds:** the typewriter sound, sound schemes, an "uh-oh" style alert. Use our own samples (see section 7).
- **Skinning** in the Winamp/QIP spirit. For web and Tauri, `98.css` or `XP.css` gets far. Mobile needs a custom retro theme in SwiftUI and Compose.
- **Late-90s web touches:** 88x31 buttons, guestbook-style profiles, opt-in "random chat".

## 7. Risks and caveats

- **Name and trademark.** "ICQ" is a VK trademark, and VK is a Russian company. Use ICQ as inspiration, but choose our own name, logo and sounds. Do not reuse the original "Uh-oh!" sample or the flower logo. Do not give anyone an easy legal lever.
- **User and developer risk.** Russian law restricts promoting circumvention tools. Consider where developers live and where servers and money sit. Keep hosting and payments outside Russia.
- **Push infrastructure.** FCM/APNs may be throttled or blocked. This is why the Android persistent socket and UnifiedPush are the main path.
- **Discovery is the weak spot.** If bootstrapping fails, the app is useless. It needs the most testing.

## 8. Proposed roadmap

| Phase | Goal |
|---|---|
| **0** | Threat model, protocol and identity spec, a name. Spikes: crypto choice (libsignal FFI vs MLS) and gomobile. Fix current infra (TLS, close MQTT, no anonymous access). CI on Linux only. |
| **1** | Go relay + Go core with a CLI client. Keys and UIN, E2EE 1:1 with offline queue. Two transports (plain WSS plus one camouflage). |
| **2** | Web client with the retro UI: contact list, statuses, authorization requests, sounds. Multi-relay failover. |
| **3** | Android native (persistent socket, UnifiedPush), then iOS native. Signed bootstrap config, invite links. |
| **4** | Tauri desktop, more transports, groups, file transfer, and a real test against a DPI-like environment (ideally with users inside Russia). |
| **Later** | Local mesh, whitelist-mode tricks, community relay program. |

Web comes before mobile on purpose. It is the fastest usable product and it validates the protocol before it is implemented three times.

## 9. Open questions for the team

1. **Name.** Are we open to dropping "ICQ" / `icq.works`?
2. **Federation.** Trusted relay set first (recommended) or open federation from the start?
3. **Shared Go core on mobile.** OK, or must networking and crypto be written natively in Swift and Kotlin?
4. **Who knows Swift and Kotlin?** This decides whether the mobile phases run in parallel or in sequence.
5. **First step.** Research current blocking techniques and transport libraries (a few hours of reading) and then rewrite `PLAN.md`, or go straight to the Phase 0 spikes?
