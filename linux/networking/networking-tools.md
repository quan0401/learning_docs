---
title: "Networking Tools — ip, ss, iptables/nftables, tcpdump, DNS, netns"
date: 2026-05-07
updated: 2026-05-07
tags: [linux, networking, iproute2, nftables, tcpdump, dns]
---

# Networking Tools — ip, ss, iptables/nftables, tcpdump, DNS, netns

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `linux` `networking` `iproute2` `nftables` `tcpdump` `dns`

---

## Table of Contents

1. [ip and ss — The iproute2 Replacements for ifconfig/netstat](#1-ip-and-ss--the-iproute2-replacements-for-ifconfignetstat)
2. [iptables and nftables — Packet Filtering and NAT](#2-iptables-and-nftables--packet-filtering-and-nat)
3. [tcpdump and Wireshark — Capturing and Reading Traffic](#3-tcpdump-and-wireshark--capturing-and-reading-traffic)
4. [DNS Tools — dig, drill, getent, /etc/nsswitch.conf](#4-dns-tools--dig-drill-getent-etcnsswitchconf)
5. [Network Namespaces — ip netns and the Container Networking Primitive](#5-network-namespaces--ip-netns-and-the-container-networking-primitive)
6. [Operator Walkthrough](#6-operator-walkthrough)

## Summary

The Linux operator's network toolbox is split between **iproute2** (configuration: `ip`, `ss`), **netfilter** (packet handling: `nftables`/`iptables`), and **observation** (`tcpdump`, `dig`, `getent`). This doc is the on-call complement to the protocol-theory side of the curriculum: the goal is to walk into a "service can't reach the database" page and know the exact sequence of commands that converts vague symptoms into a verified root cause — listening sockets, routing, conntrack state, name resolution, and namespace boundaries.

---

## 1. ip and ss — The iproute2 Replacements for ifconfig/netstat

### 1.1 Why net-tools is gone

`ifconfig`, `route`, `netstat`, `arp`, and `nis-tools` come from the **net-tools** package. The kernel grew capabilities (multiple IPs per interface, policy routing, IPv6 features, namespaces, advanced queueing disciplines) that net-tools could not represent. The replacement is **iproute2**, whose man page describes it as "utilities for controlling TCP/IP networking and traffic" via a single object-oriented `ip` command. Most distributions either ship net-tools as a deprecated compatibility package or omit it entirely on minimal images.

The mental model: `ip OBJECT COMMAND [OPTIONS]`. The object names are stable and map directly to kernel concepts.

| Object | What it manages |
|--------|-----------------|
| `link` | L2 devices (NICs, veth, bond, bridge, vlan) |
| `address` (`addr`) | L3 addresses bound to a link |
| `route` | Routing table entries |
| `neigh` | ARP / NDISC neighbor cache |
| `rule` | Routing policy database (multiple routing tables) |
| `netns` | Network namespaces |

Source: `man 8 ip` lists each as a top-level object.

### 1.2 ip link — devices

```bash
# List all links (interfaces) with state and MAC
ip link show

# Bring an interface up/down
ip link set dev eth0 up
ip link set dev eth0 down

# Set MTU
ip link set dev eth0 mtu 1400
```

The `state` field for each link is the most useful operator signal: `UP` plus `LOWER_UP` means the kernel has the interface enabled and the physical/virtual carrier is present. `NO-CARRIER` means cable unplugged (or for veth, peer is down).

### 1.3 ip addr — L3 addresses

```bash
# Show all addresses
ip addr show

# Show one interface
ip addr show dev eth0

# Add / remove an address
ip addr add 10.0.0.5/24 dev eth0
ip addr del 10.0.0.5/24 dev eth0
```

A single link can hold many addresses; this is what `ifconfig`'s alias syntax (`eth0:0`) was poorly emulating.

### 1.4 ip route — routing table

```bash
# Show the main table
ip route

# Show a specific route's resolution
ip route get 1.1.1.1

# Add a static route
ip route add 10.20.0.0/16 via 10.0.0.1 dev eth0

# Show all tables (policy routing)
ip route show table all
```

`ip route get` is gold for debugging: it answers "if I send a packet to this address right now, which interface and gateway does the kernel pick?" It also shows the source address the kernel would select.

### 1.5 ip neigh, ip rule

```bash
# ARP / neighbor cache (replaces 'arp -n')
ip neigh

# Routing policy DB (which packets use which routing table)
ip rule
```

Stuck at `STALE` or `FAILED` neighbor entries point to L2 problems before ever invoking IP routing. `ip rule` matters when systems use multiple routing tables (VRFs, mark-based policy routing).

### 1.6 ss — socket statistics

`ss` replaces `netstat` and pulls data directly from the kernel via netlink (`AF_NETLINK`/`SOCK_DIAG`), which is why it is faster and more detailed than `netstat`'s `/proc` parsing. `man 8 ss` describes it as showing "more TCP and state information than other tools."

| Flag | Meaning |
|------|---------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-a` | All sockets (listening + non-listening) |
| `-n` | Numeric (no name resolution) |
| `-p` | Show owning process (PID + program) |
| `-e` | Extended (uid, inode, cookie) |
| `-o` | Timer information for TCP |
| `-x` | Unix domain sockets |

The canonical "what's listening on this box" command:

```bash
ss -tlnp
```

`-t` TCP, `-l` listening, `-n` numeric, `-p` show process. Compare with `netstat -tlnp` which produces similar output but reads `/proc/net/tcp` line by line.

### 1.7 ss filter syntax

`ss` accepts both **state filters** and **address/port predicates**.

State filter keywords (per `man 8 ss`): `established`, `syn-sent`, `syn-recv`, `fin-wait-1`, `fin-wait-2`, `time-wait`, `closed`, `close-wait`, `last-ack`, `listening`, `closing`. Plus aggregates: `all`, `connected` (everything except listening and closed), `synchronized`, `bucket`.

```bash
# All connections to/from port 443
ss -tn '( dport = :443 or sport = :443 )'

# Only TIME-WAIT TCP sockets
ss -tn state time-wait

# Established SSH connections with timer info
ss -tno state established '( dport = :ssh or sport = :ssh )'
```

### 1.8 What LISTEN actually means in the kernel

A socket in `LISTEN` state has had `listen(2)` called on it and the kernel has set up two queues: the **SYN queue** (incomplete handshakes, awaiting ACK) and the **accept queue** (completed handshakes, awaiting `accept(2)` from userspace). `ss -lt` shows the accept queue length under `Recv-Q` for listening sockets. If `Recv-Q` is climbing, the application is not calling `accept()` fast enough — the listen backlog is filling.

For non-listening sockets, `Recv-Q` and `Send-Q` are bytes in the kernel's receive/send buffers. Non-zero `Send-Q` plus a stuck connection often means the peer is not ACKing.

### 1.9 Socket state cheat sheet

| State | What it means | When you see lots of them |
|-------|---------------|---------------------------|
| `LISTEN` | Server socket awaiting connections | Normal for daemons |
| `SYN-SENT` | Client sent SYN, waiting for SYN-ACK | Outbound connect attempts in flight |
| `SYN-RECV` | Server sent SYN-ACK, awaiting client ACK | SYN flood or slow clients |
| `ESTABLISHED` | Three-way handshake done, data flowing | Steady-state connections |
| `FIN-WAIT-1`/`FIN-WAIT-2` | Local side initiated close | Fine in transit |
| `TIME-WAIT` | Local side closed; waiting 2*MSL for stray packets | High counts on connection-churning clients |
| `CLOSE-WAIT` | Remote side closed; **local app has not called close()** | Application leak — almost always a bug |
| `LAST-ACK` | Local close after remote close | In-flight |
| `CLOSED` | No state | Should not stay long |

`CLOSE-WAIT` accumulating is the single most common operator finding from `ss`: it means a userspace process is leaking connections.

---

## 2. iptables and nftables — Packet Filtering and NAT

### 2.1 Netfilter hook points

The kernel's packet path exposes five hooks where rule sets can run. From the netfilter HOWTO:

```text
                     ┌────────────────┐
incoming packet ───► │ PRE_ROUTING    │ ──► routing decision
                     └────────────────┘             │
                                ┌───────────────────┴────────────────┐
                                ▼                                    ▼
                     ┌────────────────┐                    ┌────────────────┐
                     │ LOCAL_IN       │                    │ FORWARD        │
                     └────────────────┘                    └────────────────┘
                                │                                    │
                              process                                ▼
                                │                          ┌────────────────┐
                                ▼                          │ POST_ROUTING   │ ──► out
                     ┌────────────────┐                    └────────────────┘
                     │ LOCAL_OUT      │ ──► routing decision ───────▲
                     └────────────────┘
```

Quoted from the netfilter HOWTO: incoming packets pass `NF_IP_PRE_ROUTING` first; if destined for the local host they then hit `NF_IP_LOCAL_IN` before reaching the process; otherwise they hit `NF_IP_FORWARD`. Locally generated packets traverse `NF_IP_LOCAL_OUT`. Everything heading out the wire passes `NF_IP_POST_ROUTING`.

This is the geometry every iptables/nftables ruleset is filling in.

### 2.2 iptables: tables and chains

`iptables` (the legacy tool, still widely installed) organizes rules into **tables**, each containing **chains** that attach to specific hooks. From `man 8 iptables`:

| Table | Chains | Purpose |
|-------|--------|---------|
| `filter` (default) | INPUT, FORWARD, OUTPUT | Allow/deny |
| `nat` | PREROUTING, INPUT, OUTPUT, POSTROUTING | Address translation |
| `mangle` | PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING | Header modification |
| `raw` | PREROUTING, OUTPUT | Connection-tracking exemption |
| `security` | INPUT, OUTPUT, FORWARD | Mandatory access control |

Rule operations (also from the man page):

| Flag | Action |
|------|--------|
| `-A chain` | Append rule |
| `-I chain [pos]` | Insert at position (default 1) |
| `-D chain` | Delete |
| `-L chain` | List |
| `-F chain` | Flush all rules |
| `-N name` | Create user-defined chain |
| `-P chain target` | Set chain policy |
| `-v` | Verbose (interface + counters) |
| `-n` | Numeric output |
| `--line-numbers` | Show rule numbers |

Targets (rule actions): `ACCEPT`, `DROP`, `RETURN`, plus extension-provided `LOG`, `REJECT`, and user-defined chain jumps.

### 2.3 conntrack and ESTABLISHED,RELATED

The `conntrack` subsystem tracks every flow the kernel has seen and assigns it a state. The classic stateful firewall idiom is:

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

This says "accept any packet that belongs to a flow we already authorized, or a related flow such as an FTP data connection." `man 8 iptables` references conntrack via match extensions (`-m conntrack`). The valid states are `NEW`, `ESTABLISHED`, `RELATED`, `INVALID` (and `UNTRACKED` if the `raw` table marks the flow with `NOTRACK`).

Without this rule, every reply packet would have to be matched on its own, and you would write mirrored rules for both directions — error-prone and stateless.

### 2.4 Why nftables replaces iptables

nftables, the netfilter project's own successor (per netfilter.org), addresses the structural problems of iptables:

- **One tool, one syntax** for IPv4, IPv6, ARP, bridge, and netdev — replaces `iptables`, `ip6tables`, `arptables`, `ebtables`.
- **Atomic rule replacement** — `nft -f rules.nft` swaps the entire ruleset in a single transaction. iptables-restore is similar but iptables itself runs rule-at-a-time.
- **Native named sets and maps** — first-class data structures for "IP in this set ⇒ accept", instead of generating one rule per element with `ipset` glue.
- **No hardcoded tables/chains** — you create your own tables and attach base chains to whichever hook and priority you want.

A minimal inet filter ruleset (verified syntax from the nftables wiki quick reference):

```bash
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0\; policy drop\; }
nft add chain inet filter output { type filter hook output priority 0\; policy accept\; }
nft add chain inet filter forward { type filter hook forward priority 0\; policy drop\; }

nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept
```

The `inet` family covers both IPv4 and IPv6 with a single set of rules.

### 2.5 Reading a default-deny ruleset

The standard pattern:

```text
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state established,related accept
        ct state invalid drop
        iif lo accept
        ip protocol icmp accept
        tcp dport { 22, 80, 443 } accept
    }
}
```

Read top to bottom: established/related flows pass; invalid packets dropped; loopback always allowed; ICMP allowed; SSH/HTTP/HTTPS allowed; everything else hits the policy `drop`. Named anonymous sets `{ 22, 80, 443 }` compile to a single hash lookup in the kernel — they are not three rules.

### 2.6 Counters and tracing

Counters attach inline to any rule. Per the wiki, `tcp dport 22 counter accept` records packets and bytes hitting that rule. View with `nft list ruleset`:

```bash
nft list ruleset
```

Each `counter` line shows the running total, which is the operator's primary "is this rule actually matching?" signal. For deeper debugging, base chains support `meta nftrace set 1` to mark packets and `nft monitor trace` to print the rule path each marked packet takes.

---

## 3. tcpdump and Wireshark — Capturing and Reading Traffic

### 3.1 Anatomy of a tcpdump invocation

From the tcpdump man page, the tool "prints out a description of the contents of packets on a network interface that match the Boolean expression". The command surface that matters in practice:

| Flag | Meaning |
|------|---------|
| `-i iface` | Capture interface; `-i any` captures from all regular interfaces |
| `-n` | Don't resolve addresses to names |
| `-nn` | Also don't resolve port numbers to service names |
| `-w file` | Write raw pcap to file |
| `-r file` | Read from saved pcap |
| `-s snaplen` | Bytes per packet (default 262144) |
| `-c N` | Stop after N packets |
| `-G N` | Rotate output file every N seconds |
| `-W N` | Limit rotated file count (ring buffer) |
| `-C size` | Roll file at size (millions of bytes) |
| `-v` / `-vv` | Verbose detail |

Three operator habits that pay off:

1. Always `-nn` interactively — DNS resolution while capturing is slow and itself shows up in the capture.
2. Always `-w file.pcap` for anything past 30 seconds — printed output drops packets under load.
3. Always pair `-G` with `-W` — otherwise you fill the disk.

### 3.2 BPF / pcap filter expressions

`man 7 pcap-filter` defines the filter grammar. The primitives:

| Primitive | Matches |
|-----------|---------|
| `host H` | Source or destination address H |
| `src host H` / `dst host H` | Direction-qualified |
| `net N/M` | CIDR network |
| `port P` | TCP/UDP port |
| `portrange A-B` | Port range |
| `tcp` / `udp` / `icmp` | Protocol |
| `ether`, `ip`, `ip6`, `arp` | Family |

Combinators: `and` (`&&`), `or` (`||`), `not` (`!`).

### 3.3 Common patterns

```bash
# All traffic to/from a host
tcpdump -nn -i any host 10.0.0.5

# DNS queries
tcpdump -nn -i any 'udp port 53'

# Just the SYNs (connection openings)
tcpdump -nn -i any 'tcp[tcpflags] & tcp-syn != 0'

# RST + ACK (connection rejects)
tcpdump -nn -i any 'tcp[tcpflags] & (tcp-rst|tcp-ack) == (tcp-rst|tcp-ack)'

# Web traffic from outside our network
tcpdump -nn -i any 'tcp dst port 80 and not src net 192.168.0.0/16'
```

The `tcp[tcpflags]` byte-offset trick is from `man 7 pcap-filter`; the constants `tcp-syn`, `tcp-fin`, `tcp-rst`, `tcp-ack` are exposed by the filter compiler.

### 3.4 Reading TCP output

A typical line:

```text
12:14:33.456 IP 10.0.0.5.41982 > 10.0.0.6.5432: Flags [S], seq 12345, win 64240, options [...], length 0
12:14:33.457 IP 10.0.0.6.5432 > 10.0.0.5.41982: Flags [S.], seq 99887, ack 12346, win 65535, length 0
12:14:33.457 IP 10.0.0.5.41982 > 10.0.0.6.5432: Flags [.], ack 99888, win 64240, length 0
```

`Flags [S]` = SYN, `[S.]` = SYN-ACK, `[.]` = ACK alone, `[F.]` = FIN-ACK, `[R]` = RST, `[P.]` = PSH+ACK (data). Sequence numbers are absolute on the first packet and relative thereafter (unless you pass `-S`).

### 3.5 Wireshark interop

`tcpdump -w file.pcap` writes the standard pcap format Wireshark reads. The standard operator workflow on a remote host:

```bash
# On the server
tcpdump -nn -i any -w /tmp/issue.pcap -G 60 -W 10 'host 10.0.0.6 and port 5432'

# Pull and open locally
scp host:/tmp/issue.pcap .
wireshark issue.pcap
```

For TLS traffic decryption, the pcap alone is not enough; you need the session keys exported via `SSLKEYLOGFILE` (a Chrome/Firefox/curl/openssl-ecosystem convention) and configured in Wireshark under TLS protocol preferences. Capturing without keys still shows handshake metadata: SNI, certificate chain, cipher suite negotiation.

---

## 4. DNS Tools — dig, drill, getent, /etc/nsswitch.conf

### 4.1 dig

`dig` is "a flexible tool for interrogating DNS name servers" (per `man 1 dig`, the BIND9 utility). It is the operator's truth source — it bypasses the system NSS path and talks directly to a resolver.

| Form | Meaning |
|------|---------|
| `dig example.com` | A record via system resolver |
| `dig @1.1.1.1 example.com` | Query a specific resolver |
| `dig example.com AAAA` | IPv6 record |
| `dig example.com MX` | Mail exchangers |
| `dig example.com NS` | Authoritative nameservers |
| `dig example.com SOA` | Zone start-of-authority |
| `dig example.com TXT` | TXT records (SPF, DKIM, verification) |
| `dig example.com CAA` | Certificate authority authorization |
| `dig +short example.com` | Trim output to answers only |
| `dig +trace example.com` | Walk from root → TLD → authoritative |
| `dig +tcp example.com` | Force TCP transport |

`+trace` is what you reach for when the answer "looks wrong": it queries the root, then the TLD nameservers, then the zone's authoritatives, showing each delegation. If `+trace` agrees with `dig @1.1.1.1` but disagrees with `dig` (no `@`), the system resolver is lying — likely a stale cache or a stub resolver redirect.

### 4.2 Authoritative vs recursive

Every dig response carries flags. The two that matter:

- `aa` — **a**uthoritative **a**nswer. The server you queried owns the zone.
- `ra` — **r**ecursion **a**vailable. The server is willing to chase the answer for you.

Public resolvers (`1.1.1.1`, `8.8.8.8`) return `ra` set, `aa` clear. Authoritative servers for a zone return `aa` set for queries within their zone. `dig +trace` ends at servers that return `aa`.

### 4.3 getent vs dig — the NSS path

`man 1 getent` describes it as displaying "entries from databases supported by the Name Service Switch libraries, which are configured in `/etc/nsswitch.conf`". This is the crucial difference: **`getent hosts foo` follows the same path your application does**; `dig foo` does not.

Order of operations for `getent hosts example.com`:

1. Read `/etc/nsswitch.conf` to find the source order for the `hosts` database.
2. Try each source in order: `files` (`/etc/hosts`), `dns`, `mdns4_minimal`, `myhostname`, `resolve` (systemd-resolved), depending on what is configured.
3. Honor `[NOTFOUND=return]` and similar action specifiers.

If `dig example.com` returns the right answer but `getent hosts example.com` returns the wrong one, the bug is in `/etc/hosts` or the NSS chain — not in DNS.

### 4.4 /etc/nsswitch.conf

Per `man 5 nsswitch.conf`, the file configures lookup sources for databases including `passwd`, `group`, `shadow`, `hosts`, `networks`, `services`, `protocols`, `rpc`, `aliases`, `ethers`, `netgroup`, `initgroups`, `publickey`. A typical hosts line from the man page:

```text
hosts:          dns [!UNAVAIL=return] files
```

Action specifiers follow `[STATUS=action]` syntax — `STATUS` ∈ {`success`, `notfound`, `unavail`, `tryagain`}, `action` ∈ {`return`, `continue`, `merge`}.

### 4.5 /etc/hosts and /etc/resolv.conf

`/etc/hosts` is the static hostname-to-IP override consulted by the `files` NSS source.

`/etc/resolv.conf` (`man 5 resolv.conf`) configures the **glibc stub resolver** with directives:

| Directive | Meaning |
|-----------|---------|
| `nameserver IP` | Server to query (up to MAXNS, default 3) |
| `search DOMAIN [...]` | Search list for unqualified names |
| `domain DOMAIN` | Obsolete single-domain form |
| `options timeout:N` | Per-server timeout in seconds (default 5, cap 30) |
| `options attempts:N` | Retries before failing (default 2, cap 5) |
| `options ndots:N` | Dots threshold for absolute lookup (default 1, cap 15) |
| `options rotate` | Round-robin nameservers |

`ndots` in particular bites Kubernetes deployments where the default `ndots:5` causes every external lookup to first try every search-domain suffix.

### 4.6 systemd-resolved and resolvectl

On systemd distributions, `/etc/resolv.conf` is often a symlink to `/run/systemd/resolve/resolv.conf` and points at the **stub listener** on `127.0.0.53:53`. The actual resolution is done by `systemd-resolved`. `man 1 resolvectl` describes resolvectl as a tool that "may be used to resolve domain names, IPv4 and IPv6 addresses, DNS resource records and services with the systemd-resolved.service(8) resolver service."

```bash
resolvectl status              # Show per-link DNS configuration
resolvectl query example.com   # Resolve via systemd-resolved
resolvectl flush-caches        # Clear the cache
```

Per the man page, "/etc/resolv.conf will only be updated with servers added with this command when /etc/resolv.conf is a symlink to /run/systemd/resolve/resolv.conf, and not a static file." If you edit `/etc/resolv.conf` directly on a systemd-resolved system, your edits may be silently ignored.

---

## 5. Network Namespaces — ip netns and the Container Networking Primitive

### 5.1 What a netns is

`man 8 ip-netns` describes a network namespace as "logically another copy of the network stack, with its own routes, firewall rules, and network devices." Each namespace has its own:

- Interface table (so `lo` exists independently in each)
- Routing tables and policy rules
- Netfilter rule sets (and conntrack tables)
- Socket table (a socket bound in one namespace is invisible in another)
- ARP / neighbor cache

This is the kernel primitive containers compose with. A Docker container or a Kubernetes pod is, fundamentally, a process group running inside a network namespace plus a few other namespaces (mount, PID, UTS, IPC, user).

### 5.2 ip netns operations

```bash
# Create a named namespace (registered at /var/run/netns/NAME)
sudo ip netns add red

# List
ip netns list

# Run a command inside it
sudo ip netns exec red ip addr
sudo ip netns exec red ping -c1 127.0.0.1

# Delete the name (namespace persists if processes remain)
sudo ip netns del red
```

A freshly-created namespace has only `lo` and even that is `DOWN` until you bring it up.

### 5.3 veth pairs — connecting namespaces

A **veth** is a pair of virtual ethernet interfaces wired together. Whatever enters one comes out the other. Each end can live in a different namespace.

```bash
# Create the pair in the current (host) namespace
sudo ip link add veth-red type veth peer name veth-host

# Move one end into the red namespace
sudo ip link set veth-red netns red

# Bring up host side and address it
sudo ip addr add 10.10.0.1/24 dev veth-host
sudo ip link set veth-host up

# Bring up namespace side and address it
sudo ip netns exec red ip addr add 10.10.0.2/24 dev veth-red
sudo ip netns exec red ip link set veth-red up
sudo ip netns exec red ip link set lo up
```

Now `ping 10.10.0.2` from the host reaches the namespace.

### 5.4 Bridging multiple namespaces

For more than two endpoints, use a **Linux bridge** (`type bridge`) and attach each veth's host side to the bridge as a port. This is the same construction Docker uses for its default `bridge` network:

```bash
sudo ip link add br0 type bridge
sudo ip link set br0 up
sudo ip link set veth-host master br0
```

Each namespace plugs into the bridge via its host-side veth; the bridge handles L2 switching between them.

### 5.5 Containers and Kubernetes pods

- **Docker bridge networking**: Each container gets a netns. A veth pair connects the container's netns to the host's `docker0` bridge. NAT happens via netfilter rules in the host's namespace.
- **Kubernetes pods**: All containers in a pod share a single netns ("pause" container creates and holds it). This is why containers in the same pod talk over `localhost` and share port allocations.
- **CNI plugins**: Plug into this same `ip netns` + veth machinery; what differs is whether they bridge to a host bridge, route via the host's main table, or encapsulate (VXLAN, Geneve) for cross-node traffic.

See `kubernetes/networking/services-and-discovery.md` and `network-policies-and-dns.md` for how this composes at cluster scale.

### 5.6 nsenter — entering a container's netns from the host

`man 1 nsenter`'s synopsis: `nsenter [options] [program [arguments]]`. The `-n` flag enters the network namespace; `-t PID` selects the target process via `/proc/PID/ns/*`.

```bash
# Find a container's main PID (via Docker)
PID=$(docker inspect --format '{{.State.Pid}}' my-container)

# Run ss inside the container's netns (without tools needing to exist in the container)
sudo nsenter -t "$PID" -n ss -tlnp
sudo nsenter -t "$PID" -n ip addr
sudo nsenter -t "$PID" -n tcpdump -nn -i any -c 20
```

This is the single most useful technique for debugging container networking when the container image is distroless or otherwise lacks `ip`, `ss`, `tcpdump`. The host's tools work; you just borrow the container's namespace.

---

## 6. Operator Walkthrough

A reproducible scenario on a throwaway Linux VM (Ubuntu/Debian/Fedora — adapt package names). Run as root or with `sudo`. None of these operations leave persistent state if you delete the namespace at the end.

### 6.1 Find what is listening and the owning PID

```bash
# Start something to look at
python3 -m http.server 8080 &

# Find it
ss -tlnp | grep 8080
# LISTEN  0   5  *:8080  *:*  users:(("python3",pid=12345,fd=3))
```

`Recv-Q` is the current accept-queue depth; `Send-Q` for a listener is the configured backlog. The `users:` field gives PID and file descriptor.

### 6.2 Capture a single TCP handshake on loopback

```bash
# Terminal A — capture
sudo tcpdump -nn -i lo -c 6 'tcp port 8080'

# Terminal B — generate one connection
curl -s http://127.0.0.1:8080 > /dev/null
```

Expect six packets: SYN, SYN-ACK, ACK (handshake), then PSH-ACK with HTTP request, ACK, PSH-ACK with HTTP response (and possibly more for FIN exchange — adjust `-c`).

### 6.3 Allow a port with nftables and verify with counters

This requires nftables installed (`apt install nftables` / `dnf install nftables`). Skip on systems still on iptables-only.

```bash
# Create a test table
sudo nft add table inet test
sudo nft add chain inet test input { type filter hook input priority 0\; policy accept\; }
sudo nft add rule inet test input tcp dport 8080 counter accept

# Generate traffic
curl -s http://127.0.0.1:8080 > /dev/null

# Inspect
sudo nft list ruleset
# table inet test {
#     chain input {
#         type filter hook input priority 0; policy accept;
#         tcp dport 8080 counter packets 5 bytes 320 accept
#     }
# }

# Cleanup
sudo nft delete table inet test
```

The `counter packets N bytes M` line proves the rule matched. Counters that stay at zero while traffic flows mean the rule is unreachable — earlier rule already accepted/dropped, or the chain is on the wrong hook.

### 6.4 Create a network namespace and ping across a veth pair

```bash
# Setup
sudo ip netns add lab
sudo ip link add v-host type veth peer name v-lab
sudo ip link set v-lab netns lab

sudo ip addr add 10.99.0.1/24 dev v-host
sudo ip link set v-host up

sudo ip netns exec lab ip addr add 10.99.0.2/24 dev v-lab
sudo ip netns exec lab ip link set v-lab up
sudo ip netns exec lab ip link set lo up

# Test from host into namespace
ping -c2 10.99.0.2

# Test from namespace
sudo ip netns exec lab ping -c2 10.99.0.1

# Inspect listening sockets in the namespace (none, by design)
sudo ip netns exec lab ss -tlnp

# Cleanup
sudo ip link del v-host         # deletes both ends of the veth
sudo ip netns del lab
```

This reproduces in 10 lines what container runtimes do at startup.

### 6.5 Authoritative DNS query for a public domain

```bash
# Find authoritative nameservers for a zone
dig +short NS kernel.org

# Query one of them directly (bypassing recursive resolvers)
dig @ns1.kernel.org kernel.org A +norecurse

# Walk the delegation
dig +trace kernel.org

# Compare with the system NSS path
getent hosts kernel.org
```

If `dig @ns1.kernel.org` and `dig +trace` agree but `dig kernel.org` (no `@`) disagrees, your local resolver is stale — `resolvectl flush-caches` (systemd-resolved) or restart the local resolver. If `dig` and `getent hosts` disagree, check `/etc/nsswitch.conf` and `/etc/hosts` — something on the NSS path is overriding DNS.

---

## Related

- [../../networking/transport/tcp-deep-dive.md](../../networking/transport/tcp-deep-dive.md) — protocol-level theory for the states `ss` and `tcpdump` show
- [../../networking/transport/socket-programming.md](../../networking/transport/socket-programming.md) — what `LISTEN`, accept queues, and the `SO_*` knobs mean from the API side
- [../../networking/application-layer/dns-internals.md](../../networking/application-layer/dns-internals.md) — DNS resolution internals complementing the operator tooling here
- [../../networking/application-layer/tls-and-certificates.md](../../networking/application-layer/tls-and-certificates.md) — what tcpdump shows you (and doesn't) for TLS handshakes
- [../../networking/infrastructure/firewalls-and-security.md](../../networking/infrastructure/firewalls-and-security.md) — firewall design that the nftables rules in section 2 implement
- [../../kubernetes/networking/services-and-discovery.md](../../kubernetes/networking/services-and-discovery.md) — how pod networking composes namespaces, veths, and iptables/nftables
- [../../kubernetes/networking/network-policies-and-dns.md](../../kubernetes/networking/network-policies-and-dns.md) — netfilter-backed policy enforcement and cluster DNS
- [../../operating-systems/fundamentals/06-file-descriptors-and-ulimits.md](../../operating-systems/fundamentals/06-file-descriptors-and-ulimits.md) — sockets are file descriptors; `ss -p` ties them back to processes
- [../../operating-systems/fundamentals/07-epoll-kqueue-iouring.md](../../operating-systems/fundamentals/07-epoll-kqueue-iouring.md) — what a server actually does between `accept()` calls

## References

- `ip(8)` — iproute2 main page. <https://man7.org/linux/man-pages/man8/ip.8.html>
- `ip-netns(8)` — network namespace management. <https://man7.org/linux/man-pages/man8/ip-netns.8.html>
- `ss(8)` — socket statistics. <https://man7.org/linux/man-pages/man8/ss.8.html>
- `iptables(8)` — packet filtering tables, chains, targets. <https://man7.org/linux/man-pages/man8/iptables.8.html>
- nftables wiki main page — overview of families, hooks, atomic replacement, named sets. <https://wiki.nftables.org/wiki-nftables/index.php/Main_Page>
- nftables wiki "Quick reference - nftables in 10 minutes" — verified syntax for tables, chains, counters, sets. <https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes>
- `tcpdump(1)` — capture flags and basic filter intro. <https://www.tcpdump.org/manpages/tcpdump.1.html>
- `pcap-filter(7)` — BPF filter expression grammar. <https://www.tcpdump.org/manpages/pcap-filter.7.html>
- `dig(1)` — BIND9 DNS lookup utility. <https://man7.org/linux/man-pages/man1/dig.1.html>
- `getent(1)` — NSS database query utility. <https://man7.org/linux/man-pages/man1/getent.1.html>
- `nsswitch.conf(5)` — Name Service Switch configuration. <https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html>
- `resolv.conf(5)` — glibc stub resolver configuration. <https://man7.org/linux/man-pages/man5/resolv.conf.5.html>
- `resolvectl(1)` — systemd-resolved client. <https://man7.org/linux/man-pages/man1/resolvectl.1.html>
- `nsenter(1)` — execute in namespaces of another process. <https://man7.org/linux/man-pages/man1/nsenter.1.html>
- Netfilter Hacking HOWTO §3 — packet flow through PRE_ROUTING, LOCAL_IN, FORWARD, LOCAL_OUT, POST_ROUTING hooks. <https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html>
