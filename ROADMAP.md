# NexDPI roadmap — no VPS, no ByeDPI

## Milestone 0 — independent Android shell
- [x] Standalone Android project and original Jetpack Compose interface.
- [x] Local whitelist and installed-app picker.
- [x] GitHub Actions workflow for unit tests and Android debug build.

## Milestone 1 — local encrypted DNS
- [x] Foreground `VpnService` with Android consent flow.
- [x] Virtual IPv4/IPv6 DNS addresses and DNS packet parsing/response encoding.
- [x] DNS-over-HTTPS with Cloudflare, Google and Quad9 failover.
- [x] Fixed DoH bootstrap addresses with TLS hostname verification.
- [x] DNS response validation and SERVFAIL fallback.
- [x] Bounded IPv4/IPv6 fragment reassembly.

## Milestone 2 — local TCP/UDP datapath
- [x] IPv4/IPv6 UDP codec, checksums and response fragmentation.
- [x] Protected local UDP relay with flow limits and idle expiry.
- [x] IPv4/IPv6 TCP codec, checksums and reset handling.
- [x] Bounded TCP stream reassembly and sliding window.
- [x] Local userspace TCP relay with SYN/SYN-ACK, ACK, data, retransmission and close handling.
- [x] TLS ClientHello parser/gate with SNI extraction and best-effort split candidates.
- [x] Packet dispatcher for DNS, generic UDP and TCP.
- [x] Wire TCP/UDP relays into IPv4/IPv6 default-route TUN mode.
- [x] Bounded parallel transport and DNS workers; per-flow synchronization; defaults capped at 32 TCP and 24 UDP flows.
- [ ] Implement or deliberately bypass safely unsupported ICMP/other IP protocols.
- [ ] Verify per-flow ordering under high concurrency and network handoff.
- [x] Add bounded per-SNI strategy exploration and remote-response feedback heuristic.
- [ ] Validate the feedback signal and strategy ranking on real operator networks.

## Milestone 3 — optional proxy support
- [x] Multi-profile proxy manager with encrypted Android Keystore storage.
- [x] Import `t.me/socks` / `tg://socks` SOCKS5 links and authenticated `socks5://` / `socks5h://` URIs.
- [x] SOCKS5 TCP CONNECT and UDP ASSOCIATE transport integration with RFC 1929 username/password authentication.
- [x] Import MTProxy `t.me/proxy` / `tg://proxy` links; validate basic, `dd` random-padding and `ee` FakeTLS secrets; hand off to Telegram's native client.
- [x] HTTP CONNECT URI import and TCP transport integration with optional Basic auth (no plain HTTP-only, Digest/NTLM or TLS-to-proxy mode).
- [x] Fail closed for UDP when the selected HTTP CONNECT profile cannot carry UDP.
- [ ] Route NexDPI's own DoH upstream requests through the selected proxy profile.
- [ ] Optional automatic failover between saved SOCKS5/HTTP CONNECT profiles.
- [ ] Physical-device tests against real SOCKS5, HTTP CONNECT and MTProxy servers.

## Milestone 4 — reliability and validation
- [x] 96 local Kotlin tests passed through a JUnit-compatible runner, including SOCKS5 auth/CONNECT, SOCKS5 UDP ASSOCIATE, HTTP CONNECT and Telegram proxy-link parsing. Full Gradle test task has not been run in this environment.
- [ ] Run actual Gradle unit-test task in CI and inspect build artifact.
- [ ] Instrumentation tests for connect/disconnect/revoke/restart and whitelist changes.
- [ ] Device tests on Wi-Fi and mobile data, IPv4-only, IPv6 and dual-stack networks.
- [ ] Verify Android Private DNS automatic/off/strict behavior.
- [ ] Measure DNS latency, throughput, battery and memory under realistic app load.
- [ ] Review TCP/UDP flow state, queue limits, malformed packet handling and logs.
- [ ] Signed release build after device validation.

## Non-negotiable constraints
- No ByeDPI/ByeDPIAndroid code, dependency, wrapper or copied UI.
- No VPS and no mandatory remote relay.
- No claim of universal DPI bypass.
- User-facing controls are limited to whitelist and optional proxy profiles; proxy use remains off until selected.
- Do not label a network strategy successful until measured on the actual network.

- [ ] Route NexDPI DoH upstream through the selected SOCKS5/HTTP CONNECT profile to avoid direct DNS bootstrap outside the proxy.
- [ ] Preserve original DNS hostnames for `socks5h://` remote-DNS semantics where possible.
