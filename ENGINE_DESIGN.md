# NexDPI engine design

## Current packet path

`Android app -> VpnService TUN -> IPv4/IPv6 parser and fragment reassembler -> packet dispatcher -> DNS / TCP / UDP local engine -> protected direct sockets OR selected proxy transport -> packet encoder -> TUN -> Android app`

The VPN installs both IPv4 and IPv6 default routes. Supported TCP and UDP flows are terminated locally and relayed through sockets protected with `VpnService.protect()`, preventing those sockets from re-entering the VPN. With no proxy profile, sockets connect directly. A selected SOCKS5 profile uses SOCKS5 CONNECT for TCP and a per-flow UDP ASSOCIATE for UDP, including optional RFC 1929 username/password authentication. A selected HTTP CONNECT profile uses HTTP CONNECT plus optional Basic authentication for TCP and rejects UDP to avoid silent direct fallback. MTProxy profiles are exported to Telegram through the native `tg://proxy` deep link; MTProxy is not a general egress transport. DNS/UDP destination port 53 is handled separately and forwarded to DoH. NexDPI itself is excluded from the VPN so the DoH upstream can use the underlying network; DoH currently does not use the selected proxy. Whitelisted apps are excluded before the TUN is established.

## Live modules

- `vpn/NexDpiVpnService`: foreground VPN lifecycle, Android consent, IPv4/IPv6 default routes, app exclusions, bounded DNS/TCP/UDP executors, sequential UDP submission, session-generation checks and synchronized TUN writes.
- `vpn/NexDpiPacketDispatcher`: validates and classifies complete IP packets. UDP/53 is handled as DNS; other UDP goes to the UDP relay; TCP/53 is rejected rather than leaked as plaintext; other TCP goes to the TCP relay. Unsupported protocols fail closed.
- `vpn/IpFragmentReassembler`: bounded, timeout-limited IPv4/IPv6 input-fragment reassembly. Overlaps, invalid headers and malformed sets are discarded.
- `vpn/DnsPacketCodec`: parses IPv4/IPv6 UDP DNS, builds checksum-correct replies, fragments large replies to TUN MTU, validates upstream transaction ID and DNS response flags.
- `vpn/DnsOverHttpsResolver`: RFC 8484 DoH using HTTPS, Cloudflare/Google/Quad9 failover, bounded timeouts, upstream response validation and sticky provider selection.
- `vpn/DohBootstrapDns`: fixed bootstrap IP addresses for supported DoH hostnames; normal TLS hostname verification remains enabled.
- `vpn/UdpPacketCodec`: validates IPv4/IPv6 UDP packets and checksums; handles common IPv6 extension headers and response fragmentation.
- `vpn/UdpFlowRelay`: bounded five-tuple UDP flow table (24 concurrent flows by default), protected direct datagram sockets or SOCKS5 UDP associations, idle expiry, and reverse packet injection. When an HTTP CONNECT profile is selected, UDP is rejected instead of sent directly. This carries ordinary UDP and QUIC datagrams when the selected transport supports them; it does not make blocked destination IPs reachable.
- `vpn/TcpPacketCodec`: parses IPv4/IPv6 TCP and builds checksummed packets, including reset packets for TCP DNS.
- `vpn/TcpSlidingWindow` and `vpn/TcpStreamReassembler`: bounded sequence-aware buffering, reordering, retransmit handling and wraparound checks.
- `vpn/TcpFlowRelay`: local userspace TCP endpoint (32 concurrent flows by default) with protected direct sockets, SOCKS5 CONNECT or HTTP CONNECT setup, handshake, sequence/acknowledgement tracking, bounded windows, retransmission and FIN/RST lifecycle.
- `vpn/TlsClientHelloParser` and `vpn/TlsClientHelloGate`: bounded ClientHello parsing, ECH-extension detection, SNI extraction and deferred initial forwarding. The engine explores three split candidates per SNI and then uses a bounded UCB-style selector based on whether the remote endpoint sends data after ClientHello. This is only a reachability heuristic, not proof of TLS success or DPI circumvention. Socket write boundaries are not guaranteed to become separate wire segments.
- `proxy/TelegramProxyLinkParser`: imports Telegram `t.me` / `tg://` SOCKS5 and MTProxy links, plus `socks5://`, `socks5h://` and HTTP CONNECT URIs. The transparent tunnel currently receives resolved IP destinations, so `socks5h://` is imported as SOCKS5 but does not preserve remote-DNS semantics. Validates basic, `dd` random-padding and `ee` FakeTLS MTProxy secret encodings.
- `proxy/ProxyProfileStore`: encrypts profile credentials and secrets using AES-GCM with a key held in Android Keystore.
- `proxy/Socks5Client` / `Socks5UdpAssociation`: RFC 1928/1929 TCP CONNECT, optional username/password and UDP ASSOCIATE framing.
- `proxy/HttpConnectClient`: HTTP/1.1 CONNECT setup and Basic proxy authentication for TCP; plain HTTP proxy mode without CONNECT, Digest/NTLM and TLS-to-proxy are not implemented.
- `data/WhitelistRepository`: local per-app VPN exclusions.
- `MainActivity`: original Jetpack Compose interface with one connection button, whitelist, multi-profile proxy manager and native MTProxy handoff to Telegram.

## Explicit limitations

- Only TCP and UDP are forwarded by the current default-route datapath. ICMP, ESP and other IP protocols are not proxied and are dropped. This can affect ping and software relying on unsupported protocols.
- DNS-over-TCP on port 53 is currently rejected with RST; the DNS module only handles UDP/53. Applications excluded by the whitelist bypass all NexDPI packet handling.
- Apps using their own DoH/DoT do not send those DNS queries to the virtual DNS address. System UDP DNS queries are sent to the selected third-party DoH resolver (Cloudflare, Google or Quad9). Their transport is passed through TCP/UDP relay, but encrypted DNS is not inspected.
- No remote VPN gateway exists, by design. A direct socket cannot evade destination-IP blocking, geographic routing restrictions or an upstream outage affecting the destination.
- Proxy support: SOCKS5 can carry TCP and UDP only if the server supports UDP ASSOCIATE; HTTP CONNECT carries TCP only and fails closed for UDP; MTProxy is delegated to Telegram and is not a system-wide proxy. HTTP-only proxy mode, Digest/NTLM auth and TLS-wrapped HTTPS proxy transport are not implemented.
- DNS-over-HTTPS requests generated by NexDPI itself currently use the underlying network directly, even when a proxy profile is selected. DNS proxy routing is not yet implemented.
- When an ECH extension is present (even if the outer ClientHello exposes a public name), or no SNI is present, the TLS gate skips splitting and forwards the stream unchanged. TLS split selection is best-effort. The selector records whether the remote endpoint sent data after ClientHello, but this is not proof that TLS succeeded or DPI was bypassed. Socket write boundaries do not guarantee separate on-wire TCP segments.
- The Android service has been syntax-compiled with local API stubs. A real Gradle build and device-level network validation remain required before any release.

## Safety invariants

1. The engine does not claim that an unsupported IP protocol has been forwarded.
2. TCP/UDP upstream sockets must be protected from routing back into the VPN.
3. DoH connections retain TLS hostname verification and are not permitted to recurse into the TUN.
4. A malformed UDP/TCP DNS packet must not be sent upstream as plaintext.
5. Packet queues, flow tables, fragment buffers and stream windows are bounded.
6. A failed startup, VPN revocation or user stop invalidates the session generation and closes TUN and relay resources.
7. Whitelist changes take effect only after controlled VPN restart.
8. No app inventory, DNS history or diagnostics are uploaded.

## Android platform constraints

The user must approve a VPN through `VpnService.prepare()` before first use. Android allows one active VPN service per user/profile. Per-app allow/disallow rules are set before `Builder.establish()` and require rebuilding the VPN interface to apply changes. See Android's [VPN guide](https://developer.android.com/develop/connectivity/vpn) and [`VpnService.Builder`](https://developer.android.com/reference/android/net/VpnService.Builder).
