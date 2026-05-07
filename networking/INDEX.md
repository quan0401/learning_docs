# Networking Documentation Index — Learning Path

A progressive path from network fundamentals through transport and application protocols to modern infrastructure and observability. Practical and backend-oriented — connects protocol theory to what you actually debug and deploy as a TypeScript/Node or Java/Spring developer.

Cross-references to the [TypeScript learning path](../typescript/INDEX.md), the [Java learning path](../java/INDEX.md), the [Go learning path](../golang/INDEX.md) (`net` and `net/http` as a hands-on lens on TCP, TLS, and HTTP/2), the [Database learning path](../database/INDEX.md), the [Kubernetes learning path](../kubernetes/INDEX.md), the [Low-Level Design learning path](../low-level-design/INDEX.md), the [System Design learning path](../system-design/INDEX.md), the [Operating Systems learning path](../operating-systems/INDEX.md) for the kernel side of sockets/epoll/io_uring, the [Observability learning path](../observability/INDEX.md) for tracing and metrics across the wire, the [Performance Engineering learning path](../performance/INDEX.md) for latency/throughput analysis, the [Security learning path](../security/INDEX.md) for TLS, OAuth/OIDC, and zero-trust application of these protocols, the [Search learning path](../search/INDEX.md), the [Linux learning path](../linux/INDEX.md), the [Git Internals learning path](../git-internals/INDEX.md), and the [Web Scraping learning path](../web-scraping/INDEX.md) (DNS recon, TLS/JA3 fingerprinting, HTTP/2 fingerprint as practiced offensively against crawlers).

**Markers:** **★** = core must-learn (everyday backend protocol work, common in interviews and production debugging). **○** = supporting deep-dive (more ops/network-engineer territory or specialized advanced topics). Internalize all ★ before going deep on ○.

---

## Tier 1 — Foundations: How Networks Work

The mental model everything else builds on. Layers, addressing, and how bits travel between machines.

1. [★ OSI & TCP/IP Models — Layers, Encapsulation, and Protocol Mapping](fundamentals/osi-and-tcp-ip-models.md) — 7-layer OSI vs 4-layer TCP/IP, encapsulation/decapsulation, PDUs at each layer, where HTTP/TCP/IP/Ethernet live _(2026-04-23)_
2. [★ IP Addressing & Subnetting — IPv4, IPv6, CIDR, and NAT](fundamentals/ip-addressing-and-subnetting.md) — address structure, subnet masks, CIDR notation, private ranges, NAT/PAT, IPv6 addressing and transition _(2026-04-23)_
3. [○ Ethernet & Data Link — MAC Addresses, ARP, Switches, and VLANs](fundamentals/ethernet-and-data-link.md) — frame structure, MAC addressing, ARP resolution, switch forwarding, VLANs, broadcast domains _(2026-04-23)_

---

## Tier 2 — Transport Layer: Reliable Delivery

How data gets from process to process. TCP reliability, UDP speed, and the socket abstraction your code sits on top of.

4. [★ TCP Deep Dive — Handshakes, Flow Control, and Congestion Control](transport/tcp-deep-dive.md) — 3-way handshake, sequence/ACK numbers, sliding window, slow start, AIMD, TIME_WAIT, Nagle's algorithm, TCP Fast Open _(2026-04-23)_
5. [★ UDP & When to Use It — Connectionless Transport and QUIC](transport/udp-and-quic.md) — UDP datagram model, use cases (DNS, video, gaming), checksum, multicast, QUIC as UDP+TLS+streams _(2026-04-23)_
6. [★ Socket Programming Fundamentals — Berkeley Sockets and I/O Models](transport/socket-programming.md) — socket(), bind(), listen(), accept(), connect(), blocking vs non-blocking, select/poll intro _(2026-04-23)_

---

## Tier 3 — Application Layer: What Backend Devs Touch Daily

The protocols your code speaks directly. DNS resolution, HTTP evolution, TLS handshakes, real-time channels.

7. [★ DNS Internals — Resolution, Caching, and Record Types](application-layer/dns-internals.md) — recursive vs iterative resolution, caching hierarchy, TTL, A/AAAA/CNAME/MX/SRV/TXT records, DNS-over-HTTPS _(2026-04-23)_
8. [★ HTTP/1.1 → HTTP/2 → HTTP/3 — The Evolution of Web Transport](application-layer/http-evolution.md) — persistent connections, pipelining, multiplexing, HPACK/QPACK, server push, QUIC transport, connection coalescing _(2026-04-23)_
9. [★ TLS/SSL Handshake & Certificates — Encryption in Transit](application-layer/tls-and-certificates.md) — symmetric vs asymmetric crypto, TLS 1.3 handshake, certificate chain, ALPN, mTLS, certificate pinning, OCSP _(2026-04-23)_
10. [★ WebSocket & Server-Sent Events — Real-Time Communication](application-layer/websocket-and-sse.md) — HTTP upgrade handshake, WebSocket framing, heartbeats, SSE protocol, scaling, backpressure _(2026-04-23)_
11. [★ gRPC & Protocol Buffers — High-Performance RPC](application-layer/grpc-and-protobuf.md) — HTTP/2 transport, proto schema, streaming modes, deadlines/cancellation, vs REST/GraphQL _(2026-04-23)_

---

## Tier 4 — Infrastructure: Between Client and Server

The network layer you deploy into. Load balancers, proxies, CDNs, and security boundaries.

12. [★ Load Balancing — L4 vs L7, Algorithms, and Health Checks](infrastructure/load-balancing.md) — transport vs application LB, round-robin, least-conn, consistent hashing, sticky sessions, health probes _(2026-04-23)_
13. [★ Reverse Proxies & API Gateways — Nginx, Envoy, and Traffic Management](infrastructure/reverse-proxies-and-gateways.md) — proxy vs gateway, rate limiting, circuit breaking, request routing, TLS termination _(2026-04-23)_
14. [★ CDN & Edge Networking — Caching at the Edge](infrastructure/cdn-and-edge.md) — cache-control directives, cache invalidation, origin shielding, edge compute, anycast routing _(2026-04-23)_
15. [○ Firewalls & Network Security — Defense in Depth](infrastructure/firewalls-and-security.md) — iptables/nftables, security groups, WAF rules, DDoS mitigation, network segmentation _(2026-04-23)_

---

## Tier 5 — Network Programming in Practice

Applying protocol knowledge in code. Async I/O, connection management, and the tools to debug what goes wrong.

16. [★ Async I/O Models — select, epoll, kqueue, and Event-Driven Networking](network-programming/async-io-models.md) — blocking vs non-blocking, I/O multiplexing, reactor pattern, how Node.js (libuv) & Netty use epoll/kqueue _(2026-04-23)_
17. [★ Connection Pooling & Keep-Alive — Managing Persistent Connections](network-programming/connection-pooling.md) — HTTP keep-alive, database pool sizing (Little's Law), pool lifecycle, HikariCP, undici, pgBouncer link _(2026-04-23)_
18. [★ Network Debugging — tcpdump, Wireshark, and Diagnostic Tools](network-programming/network-debugging.md) — packet capture, Wireshark filters, netstat/ss, traceroute/mtr, curl deep flags, DNS troubleshooting _(2026-04-23)_

---

## Tier 6 — Advanced & Modern Networking

Service mesh, container networking, observability, and zero-trust — the modern production network stack.

19. [○ Service Mesh — Sidecar Proxies, mTLS, and Traffic Management](advanced/service-mesh.md) — sidecar pattern, Istio/Linkerd architecture, automatic mTLS, traffic splitting, observability _(2026-04-23)_
20. [○ Container Networking — Docker, Kubernetes CNI, and Service Discovery](advanced/container-networking.md) — bridge/host/overlay networks, CNI plugins, pod-to-pod, ClusterIP/NodePort/LoadBalancer, Ingress _(2026-04-23)_
21. [○ Network Observability — Tracing, Metrics, and eBPF](advanced/network-observability.md) — distributed tracing, latency histograms, error rates, eBPF for kernel-level visibility, RED method _(2026-04-23)_
22. [○ Zero-Trust Networking — BeyondCorp and Identity-Based Access](advanced/zero-trust.md) — BeyondCorp model, identity-aware proxies, microsegmentation, mutual TLS, policy engines _(2026-04-23)_

---

## Quick Reference by Topic

### Fundamentals

- [OSI & TCP/IP Models](fundamentals/osi-and-tcp-ip-models.md)
- [IP Addressing & Subnetting](fundamentals/ip-addressing-and-subnetting.md)
- [Ethernet & Data Link](fundamentals/ethernet-and-data-link.md)

### Transport

- [TCP Deep Dive](transport/tcp-deep-dive.md)
- [UDP & QUIC](transport/udp-and-quic.md)
- [Socket Programming](transport/socket-programming.md)

### Application Layer

- [DNS Internals](application-layer/dns-internals.md)
- [HTTP Evolution](application-layer/http-evolution.md)
- [TLS & Certificates](application-layer/tls-and-certificates.md)
- [WebSocket & SSE](application-layer/websocket-and-sse.md)
- [gRPC & Protocol Buffers](application-layer/grpc-and-protobuf.md)

### Infrastructure

- [Load Balancing](infrastructure/load-balancing.md)
- [Reverse Proxies & Gateways](infrastructure/reverse-proxies-and-gateways.md)
- [CDN & Edge](infrastructure/cdn-and-edge.md)
- [Firewalls & Security](infrastructure/firewalls-and-security.md)

### Network Programming

- [Async I/O Models](network-programming/async-io-models.md)
- [Connection Pooling](network-programming/connection-pooling.md)
- [Network Debugging](network-programming/network-debugging.md)

### Advanced

- [Service Mesh](advanced/service-mesh.md)
- [Container Networking](advanced/container-networking.md)
- [Network Observability](advanced/network-observability.md)
- [Zero-Trust Networking](advanced/zero-trust.md)

---

## Bug Spotting

Active-recall practice docs. Each presents 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), with one-line `<details>` hints inline and full root-cause + fix in a Solutions section. Every bug cites a real reference (RFC, CVE, official-doc gotcha, postmortem, library issue). Use these to pressure-test concept knowledge after working through the tiers above.

- [TCP — Bug Spotting](transport/tcp-bug-spotting.md) ★ — _(2026-05-03)_
