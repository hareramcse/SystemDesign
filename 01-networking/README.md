# 1. Networking

> Status: **Documented** — master reference

[<- Back to master index](../README.md)

## Sub-topics

| # | Sub-topic | Status |
|---|-----------|--------|
| 1.1 | [OSI Model](#11-osi-model) | Done |
| 1.2 | [TCP/IP](#12-tcpip) | Done |
| 1.3 | [TCP Handshake](#13-tcp-handshake) | Done |
| 1.4 | [UDP](#14-udp) | Done |
| 1.5 | [MTU](#15-mtu) | Done |
| 1.6 | [IP Addressing/Subnetting](#16-ip-addressingsubnetting) | Done |
| 1.7 | [CIDR](#17-cidr) | Done |
| 1.8 | [DNS](#18-dns) | Done |
| 1.9 | [DNS Resolution](#19-dns-resolution) | Done |
| 1.10 | [HTTP/HTTPS](#110-httphttps) | Done |
| 1.11 | [SSL/TLS](#111-ssltls) | Done |
| 1.12 | [HTTP/2 & HTTP/3](#112-http2-http3) | Done |
| 1.13 | [QUIC](#113-quic) | Done |
| 1.14 | [Keep-Alive Connections](#114-keep-alive-connections) | Done |
| 1.15 | [Forward & Reverse Proxy](#115-forward-reverse-proxy) | Done |
| 1.16 | [NAT](#116-nat) | Done |
| 1.17 | [VPN](#117-vpn) | Done |
| 1.18 | [Anycast/Multicast/Broadcast](#118-anycastmulticastbroadcast) | Done |
| 1.19 | [CDN](#119-cdn) | Done |
| 1.20 | [Load Balancer](#120-load-balancer) | Done |
| 1.21 | [Load Balancer Algorithm](#121-load-balancer-algorithm) | Done |
| 1.22 | [SSE, Polling & WebSockets](#122-sse-polling--websockets) | Done |

---

## 1.1 OSI Model

The **OSI (Open Systems Interconnection) Model** is a conceptual framework that explains how data travels from one computer to another over a network.

Think of it as a package delivery process. When you send a message from your laptop or mobile phone, many things happen behind the scenes before that message reaches the destination. The OSI model divides these responsibilities into **7 layers** so that each layer has a specific job.

Data moves from **Layer 7 down to Layer 1** on the sender side and then from **Layer 1 up to Layer 7** on the receiver side.

---

### Layer 7 — Application Layer

**Purpose:**  
This is the layer closest to the end user. It provides network services directly to applications.

**Examples:** HTTP, HTTPS, DNS, SMTP, FTP, gRPC

**What happens here?**

When you open a browser and type:

```text
https://google.com
```

The browser creates an HTTP request.

When you send an email, the email client creates an SMTP request.

When you send a WhatsApp message, the application creates a request that needs to be sent over the network.

This layer only cares about: *"What information do I want to send?"*

It does not care about routing, IP addresses, cables, or packet delivery.

**Real-world analogy:** You write a letter and decide what message should be written inside it.

---

### Layer 6 — Presentation Layer

**Purpose:**  
Convert data into a format that both systems can understand.

**Responsibilities:** Data formatting, serialization, encryption, decryption, compression, decompression

**Examples:** JSON, XML, Protocol Buffers (Protobuf), TLS encryption, GZIP compression

**What happens here?**

Suppose your application wants to send:

```text
Hello World
```

The presentation layer may:

- Convert text into bytes
- Compress the data
- Encrypt the data

After encryption, anybody intercepting the traffic cannot easily read the message.

**Real-world analogy:** Before sending a letter, you may translate it into another language or lock it inside a secure box.

---

### Layer 5 — Session Layer

**Purpose:**  
Manage communication sessions between two systems.

**Responsibilities:** Create connection sessions, maintain active sessions, resume interrupted sessions, terminate sessions

**Examples:** WebSocket session, TLS session, gRPC stream

**What happens here?**

Suppose two systems are continuously exchanging messages. The session layer manages:

- Connection start
- Connection maintenance
- Connection recovery
- Connection termination

**Real-world analogy:** A phone call — someone starts the call, both people keep talking, the call remains active, eventually the call ends.

---

### Layer 4 — Transport Layer

**Purpose:**  
Ensure data reaches the correct application and reaches reliably when required.

**Protocols:** TCP, UDP

**Responsibilities:** Reliable delivery, error handling, flow control, port management, packet sequencing

#### TCP (Transmission Control Protocol)

TCP focuses on **reliability**.

**Features:**

- Guaranteed delivery
- Ordered delivery
- Retransmission of lost packets
- Error detection

**Example:**

```text
Packet 1 → Delivered
Packet 2 → Lost
TCP detects the loss and sends Packet 2 again.
```

**Used by:** HTTP, HTTPS, database connections, banking applications, payment systems

**Real-world analogy:** A courier service that requires a delivery confirmation.

See also: [1.3 TCP Handshake](#13-tcp-handshake)

#### UDP (User Datagram Protocol)

UDP focuses on **speed**.

**Features:**

- No delivery guarantee
- No retransmission
- No ordering guarantee
- Lower overhead

**Used by:** Video streaming, online gaming, voice calls, DNS queries

**Real-world analogy:** Making an announcement through a loudspeaker — you simply broadcast the message and do not wait for confirmation.

See also: [1.4 UDP](#14-udp)

#### Ports

Ports identify which application should receive the data.

| Port | Application |
|------|-------------|
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |

**IP address** identifies the machine. **Port** identifies the application running on that machine.

**Example:**

```text
192.168.1.10:3306

Machine:      192.168.1.10
Application:  MySQL
```

---

### Layer 3 — Network Layer

**Purpose:**  
Find the destination system and determine the path to reach it.

**Main protocol:** IP (Internet Protocol)

**Responsibilities:** Logical addressing, routing, path selection

**Examples:** `192.168.1.10`, `10.0.0.5`, `8.8.8.8`

**What happens here?**

When data needs to travel from Bangalore to Mumbai — or even India to the USA — routers determine the best route.

Routers only care about:

- Source IP
- Destination IP

They do not care whether the data contains an HTTP request, email, image, or video. Their only job is: *"Where should I send this packet next?"*

**Real-world analogy:** Courier company deciding which city or warehouse the package should move through.

---

### Layer 2 — Data Link Layer

**Purpose:**  
Deliver data within the local network.

**Technologies:** Ethernet, Wi-Fi

**Main identifier:** MAC address (e.g. `AA:BB:CC:DD:EE:FF`)

**Responsibilities:** Local communication, frame creation, MAC addressing, error detection within local network

**Devices:** Network switches

**What happens here?**

Suppose multiple devices are connected to the same office network. The switch checks: *Which device owns this MAC address?* Then forwards the frame to the correct device.

**Real-world analogy:**

- **IP address** → apartment address
- **MAC address** → specific person living in that apartment

---

### Layer 1 — Physical Layer

**Purpose:**  
Physically transmit data.

**Examples:** Network cables, fiber optics, wireless radio signals, network cards

**Responsibilities:** Transmit bits; convert bits into electrical signals, light signals, or radio signals

At this layer there is no HTTP, TCP, UDP, IP, or JSON. Everything becomes **0 and 1**.

**Example:**

```text
01001010
10101011
11100001
```

**Real-world analogy:** Roads, cables, and transportation infrastructure used to physically move packages.

---

### How a Google request travels

| Step | What happens |
|------|--------------|
| 1 | User enters `https://google.com` |
| 2 | **Application layer** creates HTTP request |
| 3 | **Presentation layer** encrypts request using TLS |
| 4 | **Session layer** manages the communication session |
| 5 | **Transport layer** uses TCP and breaks data into segments |
| 6 | **Network layer** adds source and destination IP addresses |
| 7 | **Data link layer** adds source and destination MAC addresses |
| 8 | **Physical layer** converts everything into electrical / light / radio signals |
| 9 | Data travels through switches, routers, ISPs, and internet backbone networks |
| 10 | Google receives the data |
| 11 | The process happens in **reverse order** until Google's application receives the HTTP request |

---


## 1.2 TCP/IP


### What is it?

The **TCP/IP model** is the practical four-layer stack underlying the Internet: **Link**, **Internet (IP)**, **Transport (TCP/UDP)**, and **Application**. It is the implementation counterpart to the OSI reference model.

### Why it matters

Every web request, API call, and database connection over a network uses TCP/IP. Understanding encapsulation, IP routing, and TCP reliability is mandatory for backend and infrastructure engineers.

### How it works

1. Application generates payload (HTTP request).
2. TCP segments data, adds ports, sequence numbers, checksums.
3. IP adds source/destination addresses; routers forward hop-by-hop.
4. Link layer frames packets with MAC addresses on local segment.
5. Reverse on receive; demultiplexing by port delivers to correct socket.

```mermaid
flowchart LR
    HTTP --> TCP
    TCP --> IP
    IP --> ETH[Ethernet]
```

### Key details

- **IPv4:** 32-bit addresses, NAT common, header 20+ bytes
- **IPv6:** 128-bit addresses, no NAT needed ideally, simplified header
- **ICMP:** control messages (ping, unreachable)
- **Ports:** 0 - 65535; well-known 0 - 1023, ephemeral high ports for clients

### When to use

- Foundation for all subsequent networking topics
- Choosing TCP vs. UDP for a service
- Configuring firewalls (layer 3 vs. layer 4 rules)

### Trade-offs / Pitfalls

- IPv4 exhaustion mitigated by NAT but complicates P2P and logging
- IPv6 adoption still incomplete in some corporate networks
- "TCP/IP" colloquially means entire internet stack including app protocols
- TLS/HTTPS detail lives in [1.11 SSL/TLS](#111-ssltls) — TCP/IP section covers layers; TLS is application-adjacent security on top of TCP

### References

- [TCP/IP — networking fundamentals video](https://www.youtube.com/watch?v=2QGgEk20RXM)

---


## 1.3 TCP Handshake


### What is it?

The **TCP three-way handshake** is the connection-establishment ritual that must complete before any application bytes flow. It exchanges three segments in order: **SYN → SYN-ACK → ACK**. Each side announces an **initial sequence number (ISN)**—a 32-bit counter used to order bytes, detect duplicates, and acknowledge receipt.

Beyond sequence numbers, the handshake negotiates **TCP options** embedded in the SYN segments: maximum segment size (MSS), window scaling, selective ACK (SACK), timestamps, and (optionally) TCP Fast Open cookies. Until both sides agree the connection is open, the socket state machine remains in `SYN_SENT` / `SYN_RECEIVED`; only after the final ACK does it reach **ESTABLISHED**.

**Worked example (sequence numbers):**

| Step | Segment | Client ISN | Server ISN | Meaning |
|------|---------|------------|------------|---------|
| 1 | Client → Server: SYN, seq=1000 | 1000 | — | "I want to connect; my first byte will be seq 1000." |
| 2 | Server → Client: SYN-ACK, seq=5000, ack=1001 | 1000 | 5000 | "Accepted; my ISN is 5000; I expect your next byte at 1001." |
| 3 | Client → Server: ACK, ack=5001 | 1000 | 5000 | "I expect your next byte at 5001." Connection open. |

Teardown is a separate **four-way FIN/ACK** dance (or an abrupt **RST**). Handshake cost is paid once per new TCP connection unless connections are reused.

### Why it matters

Every **new** TCP connection pays at least **one full round-trip time (RTT)** before the server can accept application data—and often more:

| Stack layer | Typical extra RTTs (cold connection) |
|-------------|--------------------------------------|
| TCP 3-way handshake | 1 RTT |
| TLS 1.2 (full handshake) | +2 RTTs |
| TLS 1.3 (1-RTT mode) | +1 RTT |
| **Total before first HTTP byte** | **2–4 RTTs** |

On a 100 ms cross-region link, 3 RTTs = **300 ms** of pure wait before your API handler runs. That is why **HTTP keep-alive**, **connection pooling**, and **HTTP/2 multiplexing** exist: they amortize handshake + TLS cost across many requests.

**Interview point:** "Why is my API slow on the first request but fast after?" → cold TCP + TLS setup. "Why do microservices behind LBs see connection storms?" → each pod may open thousands of short-lived TCP connections per second.

### How it works

**Connection establishment (3-way):**

1. **Client → SYN:** `SYN=1`, `seq=client_ISN`, optional TCP options (MSS, window scale, SACK, timestamps).
2. **Server → SYN-ACK:** `SYN=1`, `ACK=1`, `seq=server_ISN`, `ack=client_ISN+1`. Server moves to `SYN_RECEIVED`; may queue socket in **accept backlog** if `listen()` backlog is configured.
3. **Client → ACK:** `ACK=1`, `ack=server_ISN+1`. Both sides enter **ESTABLISHED**. Application `send()`/`write()` may now place data in the send buffer.

**Connection teardown (4-way, graceful):**

1. Side A sends **FIN** (no more data from A).
2. Side B **ACK**s the FIN (B may still send data—half-close).
3. Side B sends **FIN** when done.
4. Side A **ACK**s; A enters **TIME_WAIT** (typically 2× MSL ≈ 60–120 s on Linux).

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    Note over C: CLOSED -> SYN_SENT
    C->>S: SYN seq=1000
    Note over S: LISTEN -> SYN_RECEIVED
    S->>C: SYN-ACK seq=5000 ack=1001
    Note over C: SYN_SENT -> ESTABLISHED
    C->>S: ACK ack=5001
    Note over S: SYN_RECEIVED -> ESTABLISHED
    Note over C,S: Application data flows both ways
    C->>S: FIN
    S->>C: ACK
    S->>C: FIN
    C->>S: ACK
    Note over C: TIME_WAIT (2×MSL)
```

**TCP state machine (simplified):**

```text
CLOSED --[active open, send SYN]--> SYN_SENT --[recv SYN-ACK, send ACK]--> ESTABLISHED
LISTEN --[recv SYN, send SYN-ACK]--> SYN_RECEIVED --[recv ACK]--> ESTABLISHED
ESTABLISHED --[send FIN]--> FIN_WAIT_1 --> ... --> TIME_WAIT --> CLOSED
```

**Half-open connections:** If the client sends SYN but never completes the handshake (network drop, attacker), the server holds `SYN_RECEIVED` state until **SYN backlog timeout** (often 60–180 s). `netstat` shows many `SYN_RECV` entries under SYN flood.

### Key details

**SYN flood attack**

Attacker floods SYN packets with **spoofed source IPs**. Server allocates connection table entries and sends SYN-ACKs to victims that never reply. The **accept queue** and **half-open connection table** fill; legitimate clients get dropped or time out.

**Mitigations:**

| Technique | How it works |
|-----------|--------------|
| **SYN cookies** | Server encodes connection state in the SYN-ACK sequence number; no state stored until valid ACK returns |
| **SYN proxy** (LB/firewall) | Edge completes handshake with client; separate backend handshake |
| **Rate limiting** | Drop excess SYNs per source / globally |
| **Increase `tcp_max_syn_backlog`** | More half-open slots (does not stop attack, buys time) |

```text
# Linux tuning (illustrative)
net.ipv4.tcp_syncookies = 1          # enable SYN cookies under pressure
net.core.somaxconn = 4096            # upper bound for listen() backlog
net.ipv4.tcp_max_syn_backlog = 8192  # half-open queue size
```

**TIME_WAIT**

After the side that **actively closes** sends the final ACK, it enters **TIME_WAIT** for **2 × MSL** (Maximum Segment Lifetime, often 60 s → TIME_WAIT ≈ 60–120 s). Purpose: (1) ensure the final ACK reaches the peer if lost; (2) let delayed duplicate segments from the old connection expire so they cannot corrupt a **new** connection with the same 4-tuple (src IP, src port, dst IP, dst port).

**Symptoms:** High-traffic clients (load generators, proxies) exhaust **ephemeral ports** because thousands of sockets sit in TIME_WAIT.

**Mitigations:** `SO_REUSEADDR`, connection pooling (fewer unique 4-tuples), tune `ip_local_port_range`, let the **server** initiate close where possible, or (controversial) `tcp_tw_reuse` on Linux for outgoing connections.

**Worked example — port exhaustion:**

```text
Ephemeral port range: 32768–60999 → ~28,000 ports
Each short-lived client connection: 1 socket in TIME_WAIT for ~60 s
Sustained rate: 28,000 / 60 ≈ 467 new connections/sec before "Cannot assign requested address"
```

**Listen backlog**

`listen(backlog)` creates two conceptual queues on Linux:

| Queue | Holds | Overflow behavior |
|-------|-------|-------------------|
| **SYN queue** (half-open) | Handshakes in progress | SYN cookies or drop |
| **Accept queue** (full-open) | ESTABLISHED sockets waiting for `accept()` | Client may think connected; server app slow to `accept()` → timeouts |

**TCP Fast Open (TFO):** Server issues a cookie; on repeat visits client may send **data in the SYN**, saving ~1 RTT. Limited adoption (middleboxes sometimes drop SYNs with payload).

**Interview point:** Distinguish **SYN queue** vs **accept queue**. "Backlog full" often means the application is not calling `accept()` fast enough, not that SYN flood is occurring.

### When to use

- **Performance analysis:** Attribute first-byte latency to TCP + TLS RTTs; justify keep-alive and regional edge placement.
- **Capacity planning:** Estimate max new connections/sec given TIME_WAIT duration and ephemeral port range.
- **Incident response:** `ss -s`, `netstat -an | grep SYN_RECV`, `TIME_WAIT` counts during connection storms or DDoS.
- **Load balancer design:** Decide SYN proxy vs pass-through; tune health checks that open new TCP per probe.
- **Kernel tuning:** Adjust `somaxconn`, `tcp_max_syn_backlog`, syncookies before high-traffic launches.

### Trade-offs / Pitfalls

| Pitfall | What goes wrong | Mitigation |
|---------|-----------------|------------|
| New TCP per HTTP request | 1+ RTT + TLS per request | Keep-alive, HTTP/2, connection pools |
| TLS 1.2 on high-latency links | +2 RTTs on top of TCP | TLS 1.3, session resumption (PSK/tickets) |
| LB SYN pass-through under attack | Backend half-open table saturated | SYN proxy at edge |
| NAT gateway connection limits | Each TCP through NAT consumes mapping entry | Reuse connections; raise limits |
| Missing ACK after SYN-ACK | Half-open socket until timeout | Shorter timeouts; syncookies |
| Aggressive `TIME_WAIT` reuse | Rare duplicate segment corruption | Prefer pooling over disabling TIME_WAIT |
| Health check opens new TCP every 5 s | Thousands of wasted handshakes | Less frequent checks or reuse where supported |

**Latency budget example (SF client → Virginia server, ~70 ms RTT):**

```text
TCP handshake:     1 × 70 ms =  70 ms
TLS 1.2 handshake: 2 × 70 ms = 140 ms
HTTP request/response: 1 × 70 ms =  70 ms (minimum)
─────────────────────────────────────────
Cold first request: ~280 ms before JSON arrives
Warm keep-alive:    ~70 ms (one RTT for HTTP only)
```

### References

- [TCP Handshake — TCP/IP video](https://www.youtube.com/watch?v=2QGgEk20RXM)

---

## 1.4 UDP


### What is it?

**User Datagram Protocol (UDP)** is a connectionless transport protocol sending independent datagrams without guaranteed delivery, ordering, or congestion control. Minimal 8-byte header: ports + length + checksum.

### Why it matters

UDP's simplicity enables low-latency use cases where application handles loss: DNS, VoIP, gaming, QUIC/HTTP3, video streaming. No handshake means faster first packet.

### How it works

1. Application sends datagram with source/dest ports.
2. IP routes packet; no connection state in network.
3. Receiver may get duplicates, out-of-order, or nothing.
4. Application implements retry, ordering, or tolerates loss.
5. Optional checksum validates integrity (often offloaded to hardware).

```mermaid
flowchart LR
    App1[App] -->|datagram| UDP
    UDP --> IP
    IP --> App2[App]
```

### Key details

| Property | TCP | UDP |
|----------|-----|-----|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordering | Best-effort |
| Congestion control | Built-in | None (app must implement) |
| Header size | 20+ bytes | 8 bytes |
| First packet latency | 1+ RTT (handshake) | Immediate |
| Typical uses | HTTP, gRPC, databases | DNS, QUIC, VoIP, gaming |

- Max practical payload ~65 KB; often limited to avoid fragmentation (~1200–1400 bytes safe)
- **QUIC** builds reliable streams on UDP in userspace — see [1.13 QUIC](#113-quic)
- Firewalls often block UDP except DNS — HTTP/3 needs UDP 443 open
- Broadcast/multicast built on UDP at IP layer

**Interview point:** "Use TCP unless you need UDP's simplicity or can build reliability on top (QUIC)."

### When to use

- Real-time media where late data is useless
- DNS queries (single request-response)
- Custom protocols with application-level reliability (QUIC)
- High-frequency metrics where sampling OK

### Trade-offs / Pitfalls

- No congestion control can starve TCP traffic (fairness concern)
- Application must implement reliability if needed
- NAT binding timeouts differ from TCP (often shorter for UDP)
- Fragmentation causes loss amplification - stay under path MTU

### References

- [UDP — TCP/IP and UDP video](https://www.youtube.com/watch?v=2QGgEk20RXM)

---


## 1.5 MTU
## 1.5 MTU


### What is it?

**Maximum Transmission Unit (MTU)** is the largest IP packet payload size a link can carry without fragmentation. Ethernet standard MTU is **1500 bytes**; loopback often 65536. **Path MTU** is the minimum MTU along a route.

### Why it matters

MTU mismatches cause fragmentation (performance hit) or **PMTUD black holes** (packets dropped silently when DF bit set). VPN overlays reduce effective MTU (e.g., 1400 bytes).

### How it works

1. Sender creates packet up to interface MTU.
2. If packet exceeds next hop MTU and **Don't Fragment (DF)** set, router drops and sends ICMP "fragmentation needed."
3. Sender reduces size (TCP MSS negotiation) and retransmits.
4. If ICMP blocked, **PMTUD fails** - connection hangs or times out.
5. **MSS clamping** on routers fixes TCP MSS for VPN clients.

```mermaid
flowchart LR
    Send[1500 byte packet] --> Router{MTU 1400?}
    Router -->|DF set| Drop[Drop + ICMP]
    Router -->|DF clear| Frag[Fragment]
```

### Key details

- **TCP MSS** = MTU - IP header - TCP header (typically 1460 for 1500 MTU)
- **Jumbo frames:** 9000 MTU in datacenter (lower CPU overhead)
- Cloud VPC MTU often 9001 (enhanced networking) or 1500
- QUIC/UDP apps must handle PMTUD themselves

### When to use

- Debugging mysterious VPN or cross-cloud connectivity failures
- Tuning database replication over WAN
- Configuring Docker/Kubernetes CNI overlay networks

### Trade-offs / Pitfalls

- ICMP filtering breaks PMTUD - common misconfiguration
- Fragmentation increases loss probability (lose one fragment = whole datagram lost)
- Mixed MTU paths hard to diagnose without `tracepath`
- Oversized UDP -> silent black hole if no fragmentation

### References

- [MTU and path MTU discovery — video](https://www.youtube.com/watch?v=XMcYwr-yJGA)

---


## 1.6 IP Addressing/Subnetting


### What is it?

**IP addressing** assigns unique identifiers to hosts on IP networks. **Subnetting** divides a network into smaller broadcast domains using a subnet mask - separating network prefix from host portion.

### Why it matters

Correct subnet design enables routing, security zones (public/private subnets), and capacity planning in VPCs. Mis-subnetting causes routing black holes and exhausted address space.

### How it works

1. IPv4 address: four octets (e.g., 192.168.1.10).
2. Subnet mask defines network bits vs. host bits (255.255.255.0 = /24).
3. Hosts in same subnet communicate via L2; cross-subnet via gateway (router).
4. Private ranges (RFC 1918): 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16.
5. Gateway IP typically first usable host (.1 in /24).

```mermaid
flowchart TB
    Net[Network 192.168.1.0/24] --> H1[.1 Gateway]
    Net --> H2[.10 Server]
    Net --> H3[.20 Client]
```

### Key details

- /24 = 256 addresses, 254 usable hosts (minus network/broadcast in classical model)
- **Loopback:** 127.0.0.1
- **Link-local:** 169.254.x.x (APIPA)
- Plan subnets for growth: app tier, DB tier, management per AZ

### When to use

- VPC and cloud network design
- Firewall rule scoping (CIDR blocks)
- On-prem datacenter VLAN planning

### Trade-offs / Pitfalls

- IPv4 /24 exhaustion in large flat networks - need smaller subnets or IPv6
- Overlapping CIDRs break VPN peering
- Broadcast domain size affects ARP storms (less issue in modern L3 fabrics)
- Forgetting reserved addresses (AWS reserves 5 per subnet)

### References

- [IP Addressing and Subnetting — video](https://www.youtube.com/watch?v=eWb35_xIKho)

---


## 1.7 CIDR


### What is it?

**Classless Inter-Domain Routing (CIDR)** notation expresses IP addresses and routing prefixes as `address/prefix-length` (e.g., 10.0.0.0/16). Replaced classful A/B/C addressing with flexible prefix lengths.

### Why it matters

CIDR enables efficient address allocation and aggregation in internet routing tables. Cloud security groups, NACLs, and k8s network policies all use CIDR notation.

### How it works

1. `/prefix` indicates number of leading network bits.
2. `/32` = single host; `/0` = default route.
3. Longest prefix match in routers determines route.
4. Supernetting aggregates multiple networks into one route announcement.
5. Calculate host count: 2^(32-prefix) - reserved addresses.

```mermaid
flowchart LR
    R[Router] -->|10.0.0.0/8| Cloud
    R -->|192.168.0.0/16| Office
```

### CIDR vs subnet mask

CIDR notation (`10.0.0.0/16`) is the modern way to express what subnet masks once did (`255.255.0.0`). See [1.6 IP Addressing/Subnetting](#16-ip-addressingsubnetting) for design context; CIDR is the notation used in cloud rules and routing.

### Key details

| CIDR | Hosts (approx) | Common use |
|------|----------------|------------|
| /32 | 1 | Single host rule |
| /24 | 254 | Small subnet |
| /16 | 65K | VPC |
| /8 | 16M | Large private network |

- **Aggregate routes** reduce BGP table size
- `0.0.0.0/0` means all traffic (default route)

### When to use

- Writing firewall and security group rules
- IPAM (IP address management) planning
- Understanding BGP route advertisements

### Trade-offs / Pitfalls

- Off-by-one prefix errors open huge address ranges
- Overly broad rules (/8) expose too much
- IPv6 CIDR with /64 standard for subnets - different scaling intuition

### References

- [CIDR notation — video](https://www.youtube.com/watch?v=7u0XnqS-5xs)

---


## 1.8 DNS

For the client → resolver → authoritative **resolution chain** and per-layer caches, see [1.9 DNS Resolution](#19-dns-resolution).

### What is it?

The **Domain Name System (DNS)** is a hierarchical, distributed database that maps human-readable names (`api.example.com`) to machine-usable records—primarily IP addresses, but also mail routes, aliases, text blobs, and service locations. It is the internet's **naming and directory service**: decentralized, cached at every layer, and queried billions of times per day.

**Namespace hierarchy (right-to-left):**

```text
api.shop.example.com.
│   │    │       │    └── root (implicit dot)
│   │    │       └─────── TLD (.com)
│   │    └─────────────── registered domain (example.com)
│   └──────────────────── subdomain (shop)
└──────────────────────── host label (api)
```

**Two roles every engineer should distinguish:**

| Role | Who | Responsibility |
|------|-----|----------------|
| **Authoritative nameserver** | Domain owner (Route 53, Cloudflare, registrar) | **Source of truth** for a zone (`example.com` and below) |
| **Recursive resolver** | ISP, `8.8.8.8`, `1.1.1.1`, corporate DNS | Queries on behalf of clients; **caches** answers until **TTL (Time To Live)** expires |

### Why it matters

**Every** network interaction that uses a hostname starts with DNS—often multiple lookups (page HTML, **CDN (Content Delivery Network)**, API, analytics). DNS is on the **critical path** for availability:

- Wrong A record → entire service unreachable
- Stale cache after migration → split traffic to old and new IPs
- NXDOMAIN → hard failure before TCP even starts
- Slow resolver → adds 20–200 ms to every **new** domain contact

DNS also enables **traffic management** without app changes: weighted records, geo-routing, failover health checks, and internal service discovery (Kubernetes CoreDNS, Consul).

**Interview point:** "DNS is eventually consistent by design." TTL controls how long the world believes your answer. You cannot flip traffic instantly unless TTL was lowered days in advance.

### How it works

1. **Zone delegation:** Parent zone (`.com`) holds **NS records** pointing to authoritative servers for `example.com`.
2. **Authoritative server** answers queries for names in its zone from a **zone file** or API (Route 53, etc.).
3. Every positive answer includes **TTL (Time To Live)** (seconds)—how long downstream caches may reuse the record; see table below. *Who* walks the tree on a cache miss is covered in [1.9](#19-dns-resolution).
4. **UDP port 53** by default; responses > 512 bytes (classic) or > EDNS0 size trigger **TCP fallback** (extra RTT).

**Common record types (what to know for interviews):**

| Type | Stores | Example use |
|------|--------|-------------|
| **A** | IPv4 address | `api.example.com → 203.0.113.10` |
| **AAAA** | IPv6 address | Dual-stack services |
| **CNAME** | Alias to another name | `www` → `cdn.provider.com` (cannot coexist with other records on same name) |
| **MX** | Mail server hostname + priority | Email delivery (`10 mail.example.com`) |
| **TXT** | Arbitrary text | SPF, DKIM, domain verification, ACME challenges |
| **NS** | Delegates subdomain to other nameservers | `sub.example.com` → separate zone |
| **SRV** | Service location (port, host, priority) | `_https._tcp.example.com` |
| **CAA** | Which CAs may issue certs | Security hardening |

**TTL (Time To Live)**

TTL is a **per-record** hint: "you may cache this answer for N seconds."

```text
api.example.com.  300  IN  A  203.0.113.10
                  ^^^
                  TTL = 300 s → resolvers worldwide may serve this IP for 5 min
```

| TTL strategy | Failover speed | Resolver load | When |
|--------------|----------------|---------------|------|
| High (3600–86400) | Slow (up to 1 h stale) | Low | Stable infra |
| Low (30–60) | Fast propagation | High query volume | Migrations, blue/green |
| Pre-migration | Lower TTL **days before** cutover | — | Planned IP change |

You **configure** authoritative DNS when you own a domain. You **consume** recursive DNS as a client (or run your own resolver internally).

### Key details

- **Glue records:** When NS targets live **inside** the zone they delegate (e.g., `ns1.example.com` for `example.com`), parent TLD must publish **glue A/AAAA** alongside NS or resolution loops.
- **DNSSEC:** Cryptographic chain of trust (DS → DNSKEY → RRSIG). Prevents cache poisoning; does not encrypt queries (use DoH/DoT for privacy).
- **Split-horizon / split DNS:** Same name, different answer inside corp network vs public internet (`db.internal` → `10.0.1.5` internally, NXDOMAIN externally).
- **Wildcard:** `*.example.com` matches one label level only—not `a.b.example.com`.
- **CNAME at apex (`example.com`):** Standard DNS forbids CNAME at zone apex; providers offer **ALIAS/ANAME** (resolve at query time, return A).
- **Managed DNS (Route 53, Cloudflare):** Health-checked failover, latency routing, weighted round-robin at DNS layer.
- **Negative caching:** NXDOMAIN cached per SOA **MINIMUM** field—reduces hammering on typos.

**Interview point:** CNAME vs A—CNAME adds an extra lookup hop and cannot be used at apex; A is direct but you must update IP manually on migration.

| Ops topic | Guidance |
|-----------|----------|
| **TTL migration** | T−7d: lower TTL to 60–300 s on records that will change; T0: cutover; T+2×old TTL: soak; T+7d: raise TTL on stable records. Never lower TTL mid-incident and expect instant failover. |
| **Failover** | Active/passive (health-checked swap), weighted canary, latency routing—speed always bounded by cached TTL. DNS-only failover is slower than **LB (load balancer)** drain. |
| **Sizing** | TTL too low → resolver QPS spike; keep stable CDN CNAMEs high, migration A records low; prefer ALIAS/ANAME at apex over long CNAME chains. |

### When to use

- **Public services:** A/AAAA for app servers; CNAME for CDN/vendor aliases; MX/TXT for email and verification.
- **Internal platforms:** Private zones (Route 53 Private, Azure Private DNS) for service discovery.
- **Kubernetes:** CoreDNS resolves `*.svc.cluster.local` inside the cluster.
- **Traffic shifting:** Weighted/latency records before deploying new region.
- **Debugging:** `dig`, `nslookup`; for full delegation path use `dig +trace` ([1.9](#19-dns-resolution))

### Trade-offs / Pitfalls

| Pitfall | Consequence | Mitigation |
|---------|-------------|------------|
| Long TTL during incident | Traffic keeps hitting failed IP | Pre-lower TTL; use short TTL on failover records |
| Short TTL everywhere | Higher latency, resolver load, cost | TTL per record (stable CDN CNAME high, migration A low) |
| CNAME chains | Multiple lookups, failure amplification | Flatten to A/ALIAS where possible |
| Wrong NS delegation | Domain entirely broken | Verify at registrar + authoritative |
| Split DNS mismatch | "Works in office, fails on VPN" | Document which resolver each environment uses |
| UDP truncation → TCP | Extra RTT on large responses (DNSSEC, many records) | EDNS0 buffer size; keep responses lean |
| Cache poisoning (historical) | Wrong IP served | DNSSEC, random source ports, QNAME minimization |
| TTL lowered mid-incident | Expect instant failover; traffic still hits dead IP for cached TTL | Pre-lower TTL days ahead; combine with LB drain |
| Failover record same TTL as stable | Standby promotion slow | Separate failover A with TTL=60; primary can stay high |
| Health check too aggressive | Flapping between primary/secondary at DNS layer | Match app readiness; use calculated health (2/3 regions) |

### References

- [DNS — how DNS works video](https://www.youtube.com/watch?v=vhfRArT11jc)

---


## 1.9 DNS Resolution

For record types, **TTL (Time To Live)** strategy, and failover runbooks, see [1.8 DNS](#18-dns).

### What is it?

**DNS (Domain Name System) resolution** is the end-to-end process of turning a hostname into one or more IP addresses (and other record data)—from the application's `getaddrinfo("api.example.com")` call through browser/OS caches, stub resolver, recursive resolver, and the authoritative chain (root → TLD → zone).

Resolution is **not** a single lookup: CNAME chains add hops; A + AAAA mean two logical questions; negative answers (NXDOMAIN) are also cached.

### Why it matters

DNS resolution sits on the **critical path** for cold connections:

| Cache layer hit? | Typical added latency |
|------------------|----------------------|
| Browser cache | ~0 ms |
| OS resolver cache | ~0–1 ms |
| Recursive resolver cache | ~1–20 ms (LAN) |
| Full iterative lookup | ~20–150+ ms (depends on RTT to authoritative) |

A slow or blocking `getaddrinfo()` can **stall an entire event loop** in Node.js or Python if called synchronously on the hot path. Understanding the chain explains:

- Why `dns-prefetch` and connection pooling help
- Why NXDOMAIN still "feels" slow (negative cache TTL)
- Why `/etc/hosts` and `127.0.0.1` overrides work immediately (short-circuit before network)

**Interview point:** Resolution is separate from connection. DNS can succeed and TCP can still fail (firewall, wrong port).

### How it works

**Full resolution chain (cache miss on `api.example.com`):**

```mermaid
sequenceDiagram
    participant Browser
    participant OS as OS stub resolver
    participant Rec as "Recursive resolver (8.8.8.8)"
    participant Root as Root DNS
    participant TLD as .com TLD
    participant Auth as "Authoritative NS (example.com)"

    Browser->>Browser: Check browser DNS cache
    Browser->>OS: getaddrinfo(api.example.com)
    OS->>OS: Check /etc/hosts, nscd, OS cache
    OS->>Rec: UDP query A api.example.com
    Rec->>Rec: Check recursive cache (miss)
    Rec->>Root: NS query for .com
    Root-->>Rec: .com TLD nameservers
    Rec->>TLD: NS query for example.com
    TLD-->>Rec: example.com authoritative NS + glue
    Rec->>Auth: A api.example.com
    Auth-->>Rec: A 203.0.113.10 TTL=300
    Rec-->>OS: Answer + TTL (cached at recursive)
    OS-->>Browser: 203.0.113.10 (cached at OS)
    Browser->>Browser: TCP connect to 203.0.113.10:443
```

**Step-by-step (numbered):**

1. **Application** calls resolver API (`getaddrinfo`, Java `InetAddress`, Go `net.LookupHost`).
2. **Browser cache** (Chrome, etc.) may answer first for navigations; separate from OS cache.
3. **OS stub resolver** checks **`/etc/hosts`** (static overrides), then local cache (`systemd-resolved`, Windows DNS Client).
4. **Stub forwards** to configured **recursive resolver** (DHCP-provided ISP DNS, `8.8.8.8`, corporate internal resolver).
5. **Recursive resolver** on cache miss performs **iterative** queries:
   - Root hints → TLD nameservers for `.com`
   - TLD → authoritative NS for `example.com`
   - Authoritative → final A/AAAA (following CNAMEs if present)
6. Answer returns with **TTL**; cached at recursive, then OS; app receives IP list (often happy-eyeballs between A and AAAA).

**CNAME chase example:**

```text
Query: www.example.com
Authoritative: CNAME www.example.com → cdn.cloudfront.net
Recursive: new query A cdn.cloudfront.net → 52.84.x.x
Total: 2 authoritative round-trips on full miss (more if chain longer)
```

**Pseudo-code (recursive resolver logic, simplified):**

```text
function resolve(name, type):
    if cached(name, type) and not expired:
        return cache[name, type]

    if name is CNAME:
        target = query_authoritative(name, CNAME)
        return resolve(target, type)   // chase

    // walk delegation
    servers = root_hints
    for each label from TLD down:
        servers = query(s servers, NS for zone)
    answer = query(s servers, type for name)
    cache[name, type] = answer with TTL
    return answer
```

### Key details

| Layer / concern | Behavior |
|-----------------|----------|
| **Browser cache** | Per-tab; Chrome ~60 s minimum; bypass with hard refresh |
| **OS stub** (`systemd-resolved`, Windows DNS Client) | Machine-wide; TTL from record; flush with `ipconfig /flushdns` or `systemd-resolve --flush-caches` |
| **`/etc/hosts`** | Static override (∞ TTL until edited) — short-circuits before network |
| **Recursive resolver** (8.8.8.8, corp DNS) | Iterative walk on miss; cannot flush remotely — wait TTL or change resolver |
| **Positive / negative caching** | A/AAAA/CNAME until TTL; NXDOMAIN per SOA MINIMUM |
| **DNS prefetch / preconnect** | `<link rel="dns-prefetch">` warms browser; preconnect adds TCP + **TLS (Transport Layer Security)** |
| **Happy eyeballs (RFC 8305)** | Races AAAA vs A; broken IPv6 adds **300 ms+** before IPv4 wins — fix routing or remove AAAA |
| **Hot path** | Never block event loop on sync `getaddrinfo`; pool connections to avoid repeat lookups |

**Worked example — cache layers after first lookup:**

```text
T+0s:   Full resolution → 80 ms (recursive miss)
T+10s:  Same host again → 0 ms (OS cache hit)
T+400s: TTL=300 expired → 20 ms (recursive still cached? depends on when recursive TTL started)
```

**Runbook — "DNS updated but clients still see old IP":** `dig @authoritative-ns` → `dig @8.8.8.8` → `dig` (local stub); flush OS cache if layer (3) stale. Effective staleness = max of independent layer TTLs.

**Debugging commands:**

```bash
dig api.example.com                    # single query via configured resolver
dig +trace api.example.com             # show full delegation path
dig @8.8.8.8 api.example.com           # specific recursive
nslookup -type=A api.example.com 8.8.8.8
```

**Interview point:** `dig +trace` shows iterative resolution; `dig` alone shows what **your stub resolver** returns (may be cached).

### When to use

- **Incident debugging:** Is it DNS or TCP/TLS/app? (`dig` vs `curl -v` vs `traceroute`)
- **Migration planning:** Follow [1.8](#18-dns) TTL runbook; verify propagation (`dig @multiple resolvers`)
- **Performance:** Prefetch/preconnect for critical third-party origins; pool connections to avoid repeat lookups
- **Split-horizon issues:** Compare answers from corporate resolver vs `8.8.8.8`
- **Local dev:** `/etc/hosts` or `dnsmasq` to point `api.local` → `127.0.0.1`

### Trade-offs / Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Circular CNAME chain | SERVFAIL / resolution failure | Audit zone file |
| Long CNAME chain | Extra latency per hop | Flatten or ALIAS |
| Resolver timeout (5 s default) | Slow app startup | Faster resolver; async DNS; cache |
| Wrong corporate resolver | Internal IP leaked externally or vice versa | Split DNS policies |
| Large DNSSEC response | UDP truncate → TCP retry | EDNS0; TCP from start |
| Stale OS cache after change | "I updated DNS but laptop still old" | Flush cache; wait TTL |
| IPv6 AAAA published but broken | Happy eyeballs delay | Fix AAAA or remove |
| Assuming DNS load balances | DNS round-robin is crude | Use LB IP or short TTL + health checks |
| Ignoring stacked caches | "TTL is 60s" but users stale for minutes | Flush OS cache; check browser; verify recursive separately |
| nscd / systemd-resolved stuck | Local cache ignores authoritative change | Restart resolver or flush caches |
| DoH bypasses corp split-DNS | Internal names fail on laptop | Policy DoH off on corp devices or internal DoH forwarder |

**Happy eyeballs (brief):** Modern stacks try IPv6 and IPv4 in parallel with short timeout; broken AAAA can add **300 ms+** delay before falling back to IPv4 — see Key details table above.

### References

- [DNS Resolution — step-by-step video](https://www.youtube.com/watch?v=BZISxpdl4lQ)

---


## 1.10 HTTP/HTTPS


### What is it?

**HTTP (Hypertext Transfer Protocol)** is an **application-layer**, **request/response** protocol. A client sends a **request** (method, path, headers, optional body); a server returns a **response** (status code, headers, body). HTTP is deliberately **stateless**: each request is independent unless the application adds state (cookies, tokens, server-side sessions).

**HTTPS** is HTTP over **TLS** (Transport Layer Security). TLS provides **confidentiality** (encryption), **integrity** (tamper detection), and **authentication** (certificate proves server identity). Browsers require HTTPS for modern features (HTTP/2 in practice, geolocation, secure cookies).

**Request structure (HTTP/1.1):**

```http
GET /api/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbG...
Connection: keep-alive

(body empty for GET)
```

**Response structure:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private, max-age=60
Content-Length: 87

{"id":42,"name":"Ada"}
```

### Why it matters

HTTP semantics drive **REST API design**, **CDN caching**, **browser security**, and **observability** (status codes, headers in logs). Misusing methods or cache headers causes subtle production bugs: double POST charges, stale data served from CDN, credentials leaked over plaintext.

HTTPS is baseline for security: prevents credential sniffing, enables integrity, and is a ranking signal. Termination point (LB vs app) affects certificate management, client IP visibility, and mTLS options.

**Interview point:** HTTP is stateless; "session" is application-layer convention (cookie + server store or JWT).

### How it works

**Typical HTTPS request lifecycle:**

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Note over Client,Server: 1. TCP 3-way handshake
    Note over Client,Server: 2. TLS handshake (cert verify, key exchange)
    Client->>Server: GET /api/users HTTP/1.1 / Host: api.example.com
    Server->>Server: Route, auth, business logic
    Server-->>Client: HTTP/1.1 200 OK / Content-Type: application/json
    Note over Client,Server: 3. Connection kept alive (reuse for next request)
```

1. **DNS** resolves hostname → IP.
2. **TCP** connection established (3-way handshake).
3. **TLS handshake** (HTTPS): negotiate cipher suite, verify server certificate chain, derive session keys.
4. Client sends **HTTP request** over encrypted channel.
5. Server maps path to handler; may read body (POST/PUT).
6. Server returns **status + headers + body**.
7. **Connection: keep-alive** (default HTTP/1.1) reuses TCP+TLS for subsequent requests on same host.

**HTTP methods (semantics matter for APIs and caches):**

| Method | Safe? | Idempotent? | Typical use | Body? |
|--------|-------|-------------|-------------|-------|
| **GET** | Yes | Yes | Read resource | No |
| **HEAD** | Yes | Yes | Metadata only (no body) | No |
| **POST** | No | No | Create, actions, non-idempotent ops | Yes |
| **PUT** | No | Yes | Replace entire resource | Yes |
| **PATCH** | No | No* | Partial update | Yes |
| **DELETE** | No | Yes | Remove resource | Optional |

*PATCH idempotency depends on patch document design.

**Status code classes:**

| Class | Meaning | Examples |
|-------|---------|----------|
| **1xx** | Informational | `100 Continue` |
| **2xx** | Success | `200 OK`, `201 Created`, `204 No Content` |
| **3xx** | Redirection | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| **4xx** | Client error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests` |
| **5xx** | Server error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

**Worked example — conditional GET (caching):**

```http
# First request
GET /logo.png HTTP/1.1
→ 200 OK, ETag: "abc123", body: (image bytes)

# Second request
GET /logo.png HTTP/1.1
If-None-Match: "abc123"
→ 304 Not Modified (no body — browser uses cache)
```

**HTTPS / TLS overview** — full walkthrough (CA certificate issuance, handshake steps, session key derivation, who uses which key): see **[1.11 SSL/TLS](#111-ssltls)**.

| Phase | Purpose |
|-------|---------|
| ClientHello | Client proposes TLS version, cipher suites, SNI (hostname) |
| ServerHello + Certificate | Server picks params; sends cert chain for `api.example.com` |
| Key exchange | ECDHE → shared secret (forward secrecy); session key never sent on wire |
| Finished | Both sides derive symmetric keys; HTTP now encrypted |

TLS 1.3 completes in **1 RTT** (vs 2 for TLS 1.2). **Session resumption** (tickets/PSK) can reduce repeat connection cost to **0 RTT** (with replay trade-offs).

**Stateless model:**

```text
Request 1: POST /login → 200 + Set-Cookie: session=xyz
Request 2: GET /profile + Cookie: session=xyz → server looks up session store

HTTP itself forgot Request 1; the COOKIE carries identity.
```

### Key details

- **Host header:** Required in HTTP/1.1 for virtual hosting (one IP, many sites).
- **Content-Type / Accept:** Negotiate representation (`application/json` vs `text/html`).
- **Cache-Control:** `public`, `private`, `max-age`, `no-store` — drives CDN and browser behavior.
- **Authorization:** `Bearer` JWT, Basic (rare), custom schemes — not the same as `401` vs `403`.
- **Cookies:** `Set-Cookie` with `HttpOnly`, `Secure`, `SameSite` — security-critical.
- **Chunked transfer:** Body without fixed `Content-Length` (streaming).
- **HTTP/2+:** Same semantics; different framing (multiplexing) — see section 1.12.

#### Response compression (gzip / Brotli)

Large JSON API responses (paginated lists, nested objects) inflate bandwidth and TTFB even when logic is fast. **Compression at the edge or server** shrinks payloads with no application code change when clients send `Accept-Encoding: gzip` (or `br`).

**When it helps:**

| Symptom | Typical cause | Compression impact |
|---------|---------------|-------------------|
| 8 KB+ JSON list responses | Verbose fields, no field filtering | 70–85% size reduction with gzip |
| Slow mobile clients | High latency × large payload | Fewer bytes = faster download |
| High egress cost | Repetitive API traffic | Lower bandwidth bill |

**Spring Boot (server-level):**

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/plain
    min-response-size: 1024   # only compress responses > 1 KB
```

**NGINX (recommended at reverse proxy / ingress):**

```nginx
http {
    gzip on;
    gzip_disable "msie6";
    gzip_min_length 1024;
    gzip_comp_level 5;
    gzip_types application/json text/plain text/xml application/xml text/css application/javascript;
    gzip_vary on;
    gzip_proxied any;
}
```

**Production rules:**

| Rule | Why |
|------|-----|
| Set `min-response-size` ~1 KB | Tiny responses cost more CPU than they save |
| Enable `gzip_vary on` | Correct CDN caching per `Accept-Encoding` |
| Prefer compression at **LB/nginx** | Offloads app JVM; one config for all services |
| Test with Postman/curl `-H 'Accept-Encoding: gzip'` | Verify `Content-Encoding: gzip` header |
| Consider **Brotli (`br`)** for HTTPS | Better ratio than gzip; CPU slightly higher |
| Don't compress already-compressed assets | JPEG/PNG/gzip files — wasted CPU |

```bash
curl -H "Accept-Encoding: gzip" -I https://api.example.com/users?page=1
# Content-Encoding: gzip
```

See also [1.19 CDN](#119-cdn) (edge compression) and [7.5 API Gateway](../07-api-design/README.md#75-api-gateway).

**Common interview mistakes:**

| Mistake | Correct mental model |
|---------|---------------------|
| GET with side effects (delete on GET) | Violates safe semantics; caches may replay GET |
| 401 vs 403 | 401 = not authenticated; 403 = authenticated but denied |
| POST is always create | POST is "process this" — create is common convention |
| HTTPS encrypts URL path | SNI/hostname visible; path encrypted in TLS 1.3+ (ESNI/ECH emerging) |

### When to use

- **REST/HTTP APIs:** Map resources to nouns; methods to safe/idempotent semantics.
- **CDN configuration:** Cache GET/HEAD with correct `Cache-Control`; don't cache POST.
- **Auth design:** Stateless JWT in `Authorization` vs session cookie.
- **TLS termination:** At LB (centralized certs, WAF) vs app (end-to-end, mTLS to backend).
- **Debugging:** `curl -v`, browser DevTools Network tab — status, headers, waterfall.

### Trade-offs / Pitfalls

| Pitfall | Impact | Mitigation |
|---------|--------|------------|
| HTTP/1.1 HOL blocking | One slow response blocks others on same connection | HTTP/2 multiplexing or more connections |
| Large cookies on every request | Inflates every request header | Slim tokens; domain-scoped cookies |
| Mixed content (HTTPS page, HTTP assets) | Browser blocks or warns | Upgrade all assets to HTTPS |
| `no-store` forgotten on sensitive pages | Cached at shared proxy | Explicit cache headers |
| Terminate TLS at LB only | Traffic LB→app may be plaintext | TLS to backend or private network |
| Long polling holds connections | Exhausts worker/connection limits | WebSocket/SSE or async workers |
| Retry POST on timeout | Duplicate side effects | Idempotency keys |

**Latency stack (recap):**

```text
DNS + TCP + TLS + HTTP request/response
≈ 0–100 ms + 1 RTT + 1–2 RTT + 1 RTT  (minimum one RTT for HTTP/1.1 response)
```

### References

- [HTTP and HTTPS — web protocol video](https://www.youtube.com/watch?v=FmgIQBQ87fo)

---


## 1.11 SSL/TLS


### What is it?

**TLS (Transport Layer Security)**, successor to SSL, provides **encryption**, **integrity**, and **server authentication** (optional client auth) for TCP connections. **HTTPS = HTTP + TLS**.

TLS sits **above TCP** in the stack — TCP must be established first (see [1.3 TCP Handshake](#13-tcp-handshake)), then TLS runs its own handshake before any HTTP bytes flow.

```mermaid
flowchart TB
    HTTP[HTTP request/response] --> TLS[TLS encrypt/decrypt]
    TLS --> TCP[TCP reliable transport]
    TCP --> IP[IP routing]
```

### Why it matters

TLS protects credentials, PII, and session tokens from interception and tampering. Certificate validation prevents man-in-the-middle attacks. TLS handshake cost affects connection latency (often +1–2 RTTs on top of TCP).

**Interview framing:** "HTTPS is not just encryption — the client must **trust** the server's identity via a CA-signed certificate, then both sides derive a **session key** that never crosses the wire."

### How it works

#### 1. Certificate issuance (one-time setup)

Before any client connects, the server obtains a **digital certificate** from a **Certificate Authority (CA)**.

**Actors:** Browser/client · Server (e.g. `google.com`) · Certificate Authority (Let's Encrypt, DigiCert, etc.)

**Server generates a key pair:**

| Key | Where it lives | Purpose |
|-----|----------------|---------|
| **Server public key** | Embedded in certificate; shared with world | Encrypt/verify; identity binding |
| **Server private key** | **Only on server** — never sent | Prove ownership of certificate |

**Server requests certificate** — sends domain name + public key (CSR) to CA. CA verifies domain ownership (HTTP challenge, DNS TXT) and issues:

```text
Certificate contents:
  - Domain name
  - Server public key
  - CA signature (proves CA vouches for this binding)
  - Validity dates, serial number
```

**Important:** The certificate is **not encrypted** — it is **signed** (identity card + trusted authority stamp). Anyone can read it; only the CA could produce a valid signature using the **CA private key**.

```mermaid
sequenceDiagram
    participant Server
    participant CA
    Server->>Server: Generate key pair
    Server->>CA: CSR (domain + public key)
    CA->>CA: Verify domain ownership
    CA->>Server: Signed certificate
```

#### 2. TLS handshake

**ClientHello** — client sends supported TLS versions, cipher suites, **client random**, and **SNI** (hostname, e.g. `api.example.com`).

**ServerHello** — server responds with chosen params, **server random**, and **certificate chain** (leaf → intermediate → root).

**Client verifies certificate** using OS trust store:

| Check | Failure result |
|-------|----------------|
| Not expired | Reject connection |
| Domain matches SNI (CN/SAN) | Name mismatch warning |
| CA signature valid | Untrusted certificate |
| Chain to trusted root | Unknown issuer |

**Server proves private key ownership** — typically **CertificateVerify** (TLS 1.3): signs handshake transcript with server private key. Blocks MITM even if an attacker has a stolen cert without the private key.

#### 3. Session key derivation

Both sides independently compute the same **symmetric session key** — it is **never sent on the network**.

| Input | Source |
|-------|--------|
| Client random | ClientHello |
| Server random | ServerHello |
| Shared secret | ECDHE key exchange |

A **key derivation function (KDF)** combines these into session keys (separate encrypt/decrypt keys in practice). Unique randoms per connection enable **forward secrecy** with ECDHE.

#### 4. Encrypted application data

1. Client encrypts `GET /users` → ciphertext over TCP.
2. Server decrypts with session key → processes request.
3. Server encrypts response → client decrypts.

Bulk traffic uses **symmetric** ciphers (AES-GCM, ChaCha20-Poly1305). Asymmetric keys are used only during the handshake.

```mermaid
sequenceDiagram
    participant Client
    participant Server
    Client->>Server: ClientHello (+ client random)
    Server->>Client: ServerHello + Certificate (+ server random)
    Client->>Client: Verify certificate chain
    Client->>Server: Key exchange + CertificateVerify proof
    Note over Client,Server: Both derive session key (never on wire)
    Client->>Server: Encrypted HTTP request
    Server->>Client: Encrypted HTTP response
```

**Handshake quick reference:**

1. ClientHello → 2. ServerHello + cert → 3. Verify + key exchange → 4. Finished → 5. Encrypted HTTP

### Key details

#### Who uses which key?

| Key | Used by | Purpose |
|-----|---------|---------|
| **CA private key** | CA only | Sign certificates |
| **CA public key** | Client (trust store) | Verify certificate signature |
| **Server public key** | Client | Know server's identity; key exchange |
| **Server private key** | Server only | Prove cert ownership |
| **Session key** | Client **and** server | Encrypt/decrypt all HTTPS traffic |

- **TLS 1.3:** 1-RTT full handshake; **0-RTT resumption** with replay risk
- **TLS 1.2:** often 2-RTT full handshake; still common on legacy systems
- **Certificate chain:** leaf → intermediate → root CA in trust store
- **mTLS:** client also presents a certificate — mutual auth for service mesh / internal APIs
- **Termination:** LB decrypts TLS at edge, forwards HTTP to backend (or re-encrypts)
- **Session resumption:** TLS tickets / PSK skip full handshake on repeat connections

**Summary:**

1. Server gets **CA-signed certificate** binding domain → public key.
2. Client **verifies** cert via trusted CA public keys.
3. Server **proves** private key ownership.
4. Both **derive** the same session key (never on the wire).
5. All HTTPS traffic encrypted with session key.

### When to use

- All public web traffic (HTTPS everywhere)
- gRPC over TLS, database connections (PostgreSQL SSL)
- Service mesh sidecar mTLS

### Trade-offs / Pitfalls

- Certificate expiry outages (automate with Let's Encrypt / ACME)
- TLS termination at LB — traffic LB→app may be plaintext unless re-encrypted
- TLS inspection proxies break end-to-end trust
- 0-RTT data vulnerable to replay attacks
- CPU cost of encryption — hardware AES-NI mitigates
- Confusing **certificate** (public, signed identity) with **encrypted channel** (session key does bulk encryption)

### References

- [SSL/TLS — encryption handshake video](https://www.youtube.com/watch?v=LJDsdSh1CYM)

---


## 1.12 HTTP2 & HTTP3


### What is it?

**HTTP/2** multiplexes many requests over one TCP connection with binary framing, header compression (HPACK), and stream prioritization — reducing connection count and head-of-line blocking at the HTTP layer. **HTTP/3** uses [QUIC](#113-quic) over UDP instead of TCP, eliminating TCP-level head-of-line blocking. See [1.13 QUIC](#113-quic) for transport details.

### Why it matters

HTTP/2 dramatically improved web performance (single connection per origin). HTTP/3 further improves lossy/mobile networks where TCP retransmission blocks all streams.

### How it works

**HTTP/2:**
1. One TCP + TLS connection per origin.
2. Requests split into **streams** with unique IDs.
3. Frames interleaved on wire; HPACK compresses headers.
4. Server push (largely deprecated in practice).
5. TCP loss still blocks all streams (HOL at transport layer).

**HTTP/3:**
1. QUIC connection over UDP with built-in TLS 1.3.
2. Independent streams - loss on one doesn't block others.
3. Connection migration by connection ID (WiFi -> cellular).

```mermaid
flowchart TB
    H2[HTTP/2 over TCP] --> S1[Stream 1]
    H2 --> S2[Stream 2]
    H3[HTTP/3 over QUIC] --> Q1[QUIC stream 1]
    H3 --> Q2[QUIC stream 2]
```

### Key details

- HTTP/2 requires TLS in all major browsers (facto HTTPS)
- **ALPN** negotiates h2 during TLS handshake
- Alt-Svc header advertises HTTP/3 availability
- Load balancers need L7 HTTP/2 or pass-through

### When to use

- HTTP/2: default for modern web servers (nginx, Cloudflare)
- HTTP/3: mobile apps, global users on lossy networks
- API gateways supporting multiplexed client connections

### Trade-offs / Pitfalls

- HTTP/2 multiplexing can overwhelm single server thread if not configured
- QUIC UDP blocked on some corporate firewalls
- Debugging HTTP/3 harder than TCP (Wireshark needs keys)
- Server push wasted bandwidth when assets cached

### References

- [HTTP/2 and HTTP/3 — protocol evolution video](https://www.youtube.com/watch?v=UMwQjFzTQXw)

---


## 1.13 QUIC


### What is it?

**QUIC (Quick UDP Internet Connections)** is a transport protocol on UDP combining encryption (TLS 1.3 integrated), multiplexed streams, connection migration, and reduced handshake latency. **HTTP/3** is its primary application — see [1.12 HTTP/2 & HTTP/3](#112-http2-http3) for how they relate.

### Why it matters

QUIC solves TCP's head-of-line blocking and slow handshake, improving web performance especially on mobile. Google pioneered it; now IETF standard with broad CDN/browser support.

### How it works

1. Client sends initial packet with crypto handshake (0-RTT possible with prior ticket).
2. Connection identified by **Connection ID**, not IP:port tuple.
3. Multiple bidirectional streams within one QUIC connection.
4. Loss recovery per-stream, not whole connection.
5. Built-in congestion control in userspace (not kernel TCP stack).

```mermaid
flowchart LR
    Client -->|UDP| QUIC
    QUIC --> Streams[Multiplexed streams]
    Streams --> H3[HTTP/3]
```

### Key details

- **0-RTT:** resumption sends data immediately (replay risk for non-idempotent ops)
- **Connection migration:** CID survives IP change
- Kernel bypass: userspace implementation (CPU consideration)
- Port 443 UDP standard for HTTP/3

### When to use

- Public websites via CDN (Cloudflare, Fastly default HTTP/3)
- Mobile-first applications
- When TCP middleboxes cause issues

### Trade-offs / Pitfalls

- UDP rate limiting by ISPs
- Higher CPU than kernel TCP at extreme throughput
- NAT rebinding can break CID if not handled
- Incomplete tooling ecosystem vs. TCP

### References

- [QUIC protocol — deep dive video](https://www.youtube.com/watch?v=HnDsMehSSY4)

---


## 1.14 Keep Alive Connections


### What is it?

**HTTP keep-alive (persistent connections)** reuses the same TCP connection for multiple HTTP requests/responses, avoiding repeated TCP (+ TLS) handshakes. Controlled by `Connection: keep-alive` header (default in HTTP/1.1).

### Why it matters

Connection setup can cost 2 - 3 RTTs (TCP + TLS). Without keep-alive, each asset on a page opens new connection - devastating for HTTP/1.1 with browser connection limits (6 per host).

### How it works

1. Client and server complete initial TCP + TLS handshake.
2. First HTTP request/response completes; connection stays open.
3. Subsequent requests sent on same connection (serial in HTTP/1.1).
4. Connection closed after `Keep-Alive: timeout=N, max=M` or idle timeout.
5. HTTP/2 multiplexes many requests on one keep-alive connection.

```mermaid
sequenceDiagram
    participant Client
    participant Server
    Note over Client,Server: Single TCP connection
    Client->>Server: Request 1
    Server-->>Client: Response 1
    Client->>Server: Request 2
    Server-->>Client: Response 2
```

### Key details

- **HTTP/1.1 default:** persistent unless `Connection: close`
- Server `keepalive_timeout` (nginx default 75s)
- **Connection pooling:** client-side reuse to same host (HttpClient, OkHttp)
- **Head-of-line blocking** in HTTP/1.1 motivates HTTP/2 single connection

### When to use

- Always for HTTP/1.1 production servers
- Client libraries: configure pool size matching expected concurrency
- Database and Redis connection pooling (same concept, different protocol)

### Trade-offs / Pitfalls

- Idle connections consume server file descriptors and memory
- Load balancer idle timeout must exceed client keep-alive (or RST mid-request)
- Sticky connection to dead server until client reconnects
- Too many pooled connections across microservices -> connection storm on DB

### References

- [Keep-Alive connections — HTTP performance video](https://www.youtube.com/watch?v=zRUdSu3JlK8)

---


## 1.15 Forward & Reverse Proxy


### What is it?

A **proxy** is an intermediary that forwards requests on behalf of another party.

| Type | Sits in front of | Client knows proxy? | Typical use |
|------|------------------|---------------------|-------------|
| **Forward proxy** | Clients (outbound) | Yes (configured) | Corporate egress, VPN, privacy |
| **Reverse proxy** | Servers (inbound) | No (looks like origin) | Load balancing, TLS, API gateway |

**Reverse proxy** is what most system design discussions mean: nginx, HAProxy, Envoy, AWS ALB, Cloudflare. Overlaps heavily with [1.20 Load Balancer](#120-load-balancer) and [1.19 CDN](#119-cdn) edge caching.

### Why it matters

- **TLS termination** at edge - backends speak plain HTTP in private VPC
- **Load balancing** across app pool
- **Caching** static responses at edge
- **Security** - WAF, rate limiting, DDoS absorption, hide internal IPs
- **Routing** - path-based routing (`/api` -> service A, `/admin` -> service B)

Forward proxies control **outbound** traffic: block malicious sites, log employee browsing, cache downloads.

### How it works

**Reverse proxy flow:**

```text
1. Client resolves api.example.com -> LB/proxy IP (public)
2. Client TLS handshake with proxy (certificate for api.example.com)
3. Proxy decrypts, inspects HTTP request
4. Proxy selects backend (round-robin, least-conn, consistent hash)
5. Proxy forwards to backend:10.0.1.5 (private IP)
6. Backend responds; proxy returns to client (may re-encrypt)
```

**Forward proxy flow:**

```text
1. Browser configured: HTTP_PROXY=proxy.corp.com:8080
2. Browser sends GET http://example.com TO proxy (not direct)
3. Proxy fetches example.com on behalf of client
4. Proxy returns response; origin sees proxy IP, not client IP
```

```mermaid
flowchart LR
    subgraph Reverse
        C1[Internet Client] --> RP[Reverse Proxy / nginx]
        RP --> S1[App Server 1]
        RP --> S2[App Server 2]
    end
    subgraph Forward
        C2[Corporate Client] --> FP[Forward Proxy / Squid]
        FP --> Internet[Internet Origin]
    end
```

**Critical headers (reverse proxy):**

| Header | Purpose |
|--------|---------|
| `X-Forwarded-For` | Original client IP chain |
| `X-Forwarded-Proto` | `http` or `https` as client used |
| `X-Request-ID` | Correlation for tracing |
| `Host` | Original host header for virtual hosting |

Apps must trust these only from known proxy IPs.

### Key details

- **nginx** - most common reverse proxy + static file server
- **Envoy** - L7 proxy, foundation of Istio service mesh
- **API Gateway** (Kong, AWS API Gateway) - reverse proxy + auth + rate limit + analytics
- **Transparent forward proxy** - intercepts traffic via network policy without client config
- **Service mesh sidecar** - each pod has local reverse+forward proxy (Envoy) for east-west traffic

### When to use

- **Reverse:** every production web/API tier; SSL at edge; path routing to microservices
- **Forward:** corporate networks, compliance filtering, anonymizing scrapers
- **Both together:** mesh sidecar proxies outbound calls from app container

### Trade-offs / Pitfalls

- Reverse proxy **SPOF** without HA (keepalived VIP or cloud-managed LB)
- Wrong `X-Forwarded-For` trust -> IP spoofing bypasses rate limits
- TLS termination shifts trust boundary - encrypt VPC east-west or use mTLS to backends
- Extra network hop adds latency (~0.5-2ms)
- Large file uploads need proxy buffer tuning (`client_max_body_size`)

### References

- [Proxy Server - Hareram Singh](https://medium.com/@hareramcse/proxy-server-d390bc83f3a9)
- [Forward and Reverse Proxy - video](https://www.youtube.com/watch?v=xo5V9g9joFs)

---


## 1.16 NAT


### What is it?

**Network Address Translation (NAT)** maps private IP addresses inside a network to public IP(s) on the internet, modifying packet headers in transit. Most home routers and cloud NAT gateways use **NAPT** (port-level translation).

### Why it matters

NAT conserves scarce IPv4 addresses and hides internal topology. It also breaks inbound connections unless port forwarding configured - affecting P2P, WebRTC, and debugging client IPs.

### How it works

1. Internal host (192.168.1.10:5000) sends packet to external server.
2. NAT router replaces source with (public_ip:ephemeral_port).
3. NAT table maps ephemeral_port -> internal host:port.
4. Return packets reverse translation using table.
5. Table entries timeout after idle period.

```mermaid
flowchart LR
    H[Private 10.0.0.5] --> NAT[NAT Gateway]
    NAT -->|Public IP| Internet
```

### Key details

- **SNAT:** source translation (outbound from private)
- **DNAT:** destination translation (port forwarding inbound)
- Cloud **NAT Gateway** for private subnet egress without public IPs
- **Carrier-grade NAT (CGNAT)** shares public IP across ISP customers

### When to use

- Private subnet internet access in VPC
- Home/office networks on IPv4
- Hiding internal server IPs from clients

### Trade-offs / Pitfalls

- Breaks end-to-end connectivity - needs STUN/TURN for WebRTC
- Log correlation harder (many clients share one public IP)
- NAT table exhaustion under high connection count
- IPv6 designed to reduce NAT need

### References

- [NAT — network address translation video](https://www.youtube.com/watch?v=FTUV0t6JaDA)

---


## 1.17 VPN


### What is it?

A **Virtual Private Network (VPN)** creates an encrypted tunnel over a public network, making remote hosts appear on a private network. Protocols include IPsec, OpenVPN, WireGuard, and TLS-based corporate VPNs.

### Why it matters

VPNs secure remote access, connect site-to-site networks, and bypass geo-restrictions. Zero-trust architectures reduce blanket VPN reliance but VPN remains common for admin access and hybrid cloud.

### How it works

1. Client authenticates to VPN concentrator.
2. Encrypted tunnel established (IPsec SA or WireGuard handshake).
3. Client receives virtual IP in VPN address space.
4. Traffic routed through tunnel (full or split tunnel).
5. Decrypted at gateway; forwarded to internal resources.

```mermaid
flowchart LR
    Remote[Remote user] -->|encrypted tunnel| GW[VPN Gateway]
    GW --> Internal[Internal network]
```

### Key details

- **Split tunnel:** only corporate traffic via VPN; internet direct
- **Full tunnel:** all traffic via corporate (inspection, DLP)
- **WireGuard:** modern, minimal, fast kernel implementation
- **Site-to-site:** gateway-to-gateway for datacenter/cloud linking

### When to use

- Remote employee access to internal tools
- Connecting cloud VPC to on-prem datacenter
- Secure admin access to production (bastion alternative)

### Trade-offs / Pitfalls

- VPN concentrator SPOF and bandwidth bottleneck
- Full tunnel adds latency for SaaS apps
- Compromised VPN creds grant broad network access
- Zero-trust replaces "perimeter VPN trust" with per-request auth

### References

- [VPN — virtual private networks video](https://www.youtube.com/watch?v=R-JUOpCgTZc)

---


## 1.18 Anycast/Multicast/Broadcast

**Broadcast** — one-to-all on a LAN subnet (e.g., ARP, DHCP); does not cross routers.

**Multicast** — one-to-many to subscribed hosts (IGMP, `224.0.0.0/4`); common inside datacenters; limited on the public internet.

**Anycast** — same IP announced from multiple sites via BGP; routers deliver to the **nearest** node. Powers global **DNS (Domain Name System)** resolvers (`8.8.8.8`), **CDN (Content Delivery Network)** edges, and DDoS scrubbing.

**When asked in interviews:** contrast the three delivery semantics; note anycast + stateful TCP breaks if BGP path changes mid-session; multicast is rarely available in cloud VPCs.

See [1.8 DNS](#18-dns) and [1.19 CDN](#119-cdn) for production examples.

---


## 1.19 CDN


### What is it?

A **Content Delivery Network (CDN)** distributes cached copies of static (and sometimes dynamic) content to **edge PoPs** geographically close to users, reducing latency and origin load.

### Why it matters

Users globally expect fast page loads. CDN offloads 80 - 90% of static traffic, absorbs DDoS, and provides TLS at edge. Essential for media, e-commerce, and any global web property.

### How it works

1. Origin (S3, nginx) serves content; CDN pulls on first request (**cache miss**).
2. CDN caches at edge PoP with TTL from `Cache-Control`.
3. Subsequent nearby users get **cache hit** from edge.
4. DNS (or anycast) routes user to nearest PoP.
5. Purge API invalidates cached objects on content update.

```mermaid
flowchart LR
    User --> Edge[CDN Edge PoP]
    Edge -->|miss| Origin[Origin Server]
    Edge -->|hit| User
```

### Key details

- **Cache keys** include URL, query string, headers (Vary)
- **Dynamic acceleration:** route API through CDN private backbone
- Providers: Cloudflare, Akamai, Fastly, CloudFront
- **Stale-while-revalidate** serves old while refreshing

#### Push CDN vs pull CDN

CDNs populate edge caches in two fundamentally different ways:

| Model | How content reaches edge | Who initiates upload | Storage at edge | Best for |
|-------|--------------------------|----------------------|-----------------|----------|
| **Pull CDN** | Edge fetches from origin on first user request (cache miss) | Origin passive; CDN pulls on demand | Minimized — only recently requested content | High-traffic sites; content accessed unpredictably |
| **Push CDN** | You upload or pre-publish content directly to CDN | Developer/CI pushes to CDN | Higher — you control what is stored | Low-traffic sites; infrequent updates; large releases you want live immediately |

**Pull CDN flow:**
1. User requests `cdn.example.com/app.js`.
2. Edge PoP has miss → fetches from origin.
3. Caches with TTL from `Cache-Control`.
4. Subsequent users at that PoP get cache hit.

**Push CDN flow:**
1. Build pipeline uploads assets to CDN storage (S3 + CloudFront invalidation, or FTP/API upload).
2. URLs point directly to CDN hostname.
3. Content is at edge before first user request — no cold-start latency on first hit.

```mermaid
flowchart LR
    subgraph Pull["Pull CDN"]
        U1[User] --> E1[Edge]
        E1 -->|miss| O1[Origin]
    end
    subgraph Push["Push CDN"]
        CI[CI / deploy] --> E2[Edge storage]
        U2[User] --> E2
    end
```

**Trade-offs:**
- **Pull:** simpler origin integration; first request per PoP is slow; TTL expiry can cause redundant origin fetches if content unchanged.
- **Push:** predictable go-live; you manage upload + purge; wasted edge storage if content is rarely accessed.

Most modern CDNs (CloudFront, Cloudflare, Fastly) support **both** — static sites often use pull with origin (S3/nginx); release artifacts and versioned bundles are often pushed via deploy hooks.

### Production rules

#### Purge and invalidation

| Operation | Scope | Propagation | Use when |
|-----------|-------|-------------|----------|
| **URL purge** | Single path | Seconds–minutes per PoP | One bad object, emergency fix |
| **Wildcard / prefix purge** | `/*` or `/assets/*` | Minutes; may rate-limit | Bad deploy of static bundle |
| **Surrogate-key purge** | Tag-based (Fastly, Cloudflare) | Fast, batched | Purge all objects for `product-123` |
| **Versioned URLs** | No purge needed | Instant (new URL) | **Preferred** — `app.a1b2c3.js` immutable |

```text
Runbook — "users still see old JS after deploy":
1. Confirm deploy uploaded NEW object (check origin ETag/hash)
2. If cache-busted URL (hash in filename): purge NOT needed — check HTML still references old URL
3. If purge API called: check API response 200 + purge ID; verify not rate-limited (429)
4. dig edge IP → curl -H "Cache-Control: no-cache" https://cdn.../app.js — compare PoPs
5. If partial PoP stale: open provider ticket; temporary fix = lower TTL on that path
6. Nuclear: purge /* (expect origin spike on next global miss wave)
```

#### Purge failure modes

| Failure | Symptom | Mitigation |
|---------|---------|------------|
| **Rate limit** | 429; partial purge | Batch surrogate keys; use versioning instead |
| **Wrong cache key** | Purge "succeeds" but object stale | Include query string / `Vary` headers in key; purge exact URL |
| **PoP propagation lag** | Fixed in US, stale in EU for 5 min | Wait SLA (provider-specific); don't chain deploys |
| **Stale-while-revalidate** | Old content served while refresh async | Set `stale-while-revalidate=0` for HTML; immutable for assets |
| **Purge API down** | Deploy blocked | Fallback: new versioned filename; bypass CDN to origin temporarily |
| **Origin still old** | Purge irrelevant | Fix origin first — CDN re-fetches stale on miss |

#### Sizing

| Signal | Rule | Notes |
|--------|------|-------|
| Origin spike after purge | Every PoP may refetch same object | Purge off-peak; warm critical paths |
| Bandwidth | Cache hit ratio < 85% on static | Review TTL; versioned assets |
| Purge API quota | ~100–1000 paths/min (provider-dependent) | Surrogate keys beat 10k URL purges |
| HTML TTL | 60–300 s max if not versioned | Long TTL only with fingerprinted assets |

### When to use

- Static assets (JS, CSS, images, video)
- Downloadable files, software updates
- DDoS protection and WAF at edge

### Trade-offs / Pitfalls

| Pitfall | Consequence | Mitigation |
|---------|-------------|------------|
| Cache invalidation delay after deploy | Users see stale assets | Versioned URLs; surrogate-key purge |
| Personalized content hard to cache | Low hit ratio | Edge logic; short TTL; cache key design |
| Dynamic HTML caching | Wrong user sees wrong page | `private`, `Vary`, don't cache authenticated HTML |
| Cost scales with bandwidth egress | Bill shock | Compress; hit ratio; tiered pricing |
| Purge without versioned URLs | Every deploy needs global invalidation | Fingerprint assets; purge HTML only |
| Full-site purge at peak | Origin miss storm; latency spike | Purge off-peak; warm cache; versioned bundles |
| Ignoring `Vary: Accept-Encoding` | Purged gzip; brotli variant still stale | Purge all variants or normalize encoding at edge |

### References

- [CDN — content delivery networks video](https://www.youtube.com/watch?v=ouqqU0FJjhQ)

---


## 1.20 Load Balancer


### What is it?

A **load balancer (LB)** sits in front of multiple backend servers and **distributes incoming traffic** across them. Goals: higher **throughput**, better **availability** (survive node failure), and horizontal **scalability**.

Two main layers:
- **L4 (Transport)** - routes TCP/UDP connections by IP:port; fast; no HTTP awareness
- **L7 (Application)** - routes HTTP/gRPC by URL path, headers, host; TLS termination, cookies

Examples: AWS ALB/NLB, nginx, HAProxy, F5, Envoy, Kubernetes Service + Ingress.

### Why it matters

No single server handles modern traffic volumes. The load balancer is the **front door** of almost every scaled system - it enables rolling deploys (drain unhealthy nodes), health-based routing, and SSL at the edge.

### How it works

1. Client resolves DNS to LB VIP (virtual IP) or hostname (`api.example.com`)
2. Client connects to LB (TLS often terminates here at L7)
3. LB selects backend using algorithm — see [1.21 Load Balancer Algorithm](#121-load-balancer-algorithm) (round-robin, least connections, consistent hash)
4. LB forwards request (L7 proxy) or connection (L4 pass-through)
5. **Health checks** probe backends (`GET /health` every 10s); unhealthy nodes removed from pool
6. Optional **session affinity (sticky sessions):** same client always hits same backend via cookie or source IP hash

```mermaid
flowchart TB
    Users --> LB[Load Balancer]
    LB --> B1[Backend 1]
    LB --> B2[Backend 2]
    LB --> B3[Backend 3]
    B1 -.->|health check fail| LB
```

**L4 vs L7 decision:**

| | L4 (NLB) | L7 (ALB/nginx) |
|---|----------|----------------|
| Speed | Faster (no HTTP parse) | Slightly more CPU |
| Routing | IP + port only | Path, header, host |
| TLS | Pass-through or terminate | Terminate + route |
| Use case | DB proxy, gaming UDP, raw TCP | REST APIs, WebSocket upgrade |
| WebSocket | TCP pass-through works | Needs upgrade support |

**High availability for the LB itself:**
- Active-passive pair (keepalived/VRRP) with floating VIP
- Cloud-managed LB (AWS ELB) spans AZs automatically
- DNS failover to secondary region (higher TTL trade-off)

**Deployment patterns:**
- **Active-active:** all backends serve traffic simultaneously
- **Active-passive:** standby waits for failover
- **Cross-zone:** LB distributes across availability zones (survive AZ outage)

### Key details

- **Health check tuning:** too aggressive -> flapping; too lazy -> send traffic to dead nodes
- **Connection draining:** on deploy, stop new connections to instance, wait for in-flight to finish
- **X-Forwarded-For / X-Real-IP:** LB injects client IP for backend logging and rate limiting
- **WebSocket:** requires L7 with HTTP upgrade support + often **sticky sessions** or shared pub/sub backplane
- **gRPC:** L7 LB with HTTP/2 aware routing (path, metadata)
- **SSL/TLS:** terminate at LB (centralized cert management) vs pass-through (end-to-end encryption)

### Production rules

#### Health checks — avoid flapping

Flapping: backend alternates healthy ↔ unhealthy → connections churn, alerts noise, uneven load.

| Parameter | Too aggressive | Too lazy | Production starting point |
|-----------|----------------|----------|---------------------------|
| **Interval** | CPU spike from probes; false fail on slow GC | Traffic to dying node 30+ s | 10–30 s |
| **Timeout** | Fails during normal latency spike | Hung backends stay in pool | 2–5 s (match p99 app latency) |
| **Healthy threshold** | Slow to mark up after deploy | — | 2–3 consecutive successes |
| **Unhealthy threshold** | **Flapping** on single timeout | — | 2–5 consecutive failures |
| **Path** | 200 on `/` that hits DB | — | Dedicated `/health` or `/ready` (liveness vs readiness) |

```text
Split probes:
  Liveness  (/health)  → process up; LB keeps sending traffic
  Readiness (/ready)   → can serve traffic; K8s removes from Service, LB should mirror

Runbook — "LB flapping during deploy":
1. Check if health path shares fate with DB (mark unhealthy when DB slow → cascade)
2. Increase unhealthy threshold temporarily OR use preStop hook + drain
3. Verify probe not hitting auth-gated path (401 → unhealthy)
4. Align LB idle timeout > app graceful shutdown window
```

#### Connection draining (deploy runbook)

| Step | AWS ALB | nginx / generic |
|------|---------|-----------------|
| 1. Stop new traffic | Target deregistration delay | `weight=0` or remove from upstream |
| 2. Wait in-flight | `deregistration_delay` (default 300 s) | `proxy_read_timeout` + worker drain |
| 3. App shutdown | `preStop` sleep ≥ drain period | `SIGTERM` handler stops accept, finishes requests |
| 4. Force kill | After `terminationGracePeriodSeconds` | `worker_shutdown_timeout` |

```text
Sizing drain window:
  drain_seconds ≥ p99_request_duration + p99_websocket_idle_between_messages
  Typical API: 30–60 s
  WebSocket/chat: 300–3600 s OR force disconnect with client reconnect
```

**Rule:** `LB deregistration delay` ≥ `app graceful shutdown` ≥ longest in-flight request. If app exits first, LB RSTs active connections.

#### Sizing

| Resource | Rule of thumb | Notes |
|----------|---------------|-------|
| Connections per L7 LB | 10k–100k+ (managed ELB scales) | Watch `SurgeQueueLength`, `TargetConnectionErrorCount` |
| Health check QPS | `N_targets × (1/interval)` | 100 targets @ 10s = 10 RPS probe load — use `/ready` lightweight |
| New connections/sec | SYN rate limit at LB under viral spike | Pre-warm; use CDN; connection pooling from clients |
| Cross-zone LB | +1–2 ms latency | Worth it for AZ survival; enable on ALB |

### When to use

- More than one app server instance (almost always in production)
- Zero-downtime rolling deployments
- Geographic or AZ redundancy
- DDoS absorption at edge (CDN + LB)

### Trade-offs / Pitfalls

| Pitfall | Consequence | Mitigation |
|---------|-------------|------------|
| Sticky sessions | Uneven load; required for in-memory state | External session store (Redis) |
| SSL termination at LB | Plaintext LB→backend | TLS to backend or private VPC |
| Misconfigured health checks | Healthy nodes marked bad | Dedicated `/health`; no auth on probe |
| Single LB SPOF | Full outage if LB fails | HA pair or managed multi-AZ LB |
| Source IP hash + carrier NAT | Hot backend; unfair distribution | Cookie or user-id affinity |
| Health check flapping | Traffic yo-yo; alert fatigue; session drops | Raise unhealthy threshold; liveness vs readiness |
| Drain shorter than in-flight | RST mid-request; 502 burst on deploy | `deregistration_delay` ≥ p99 request time |
| No preStop / graceful shutdown | Pod killed while LB still sends traffic | `preStop` sleep; SIGTERM handler |
| Readiness tied to dependency | One DB blip drains entire fleet | Readiness = local only; degrade gracefully |

### References

- [Load Balancer Algorithms - comparison video](https://www.youtube.com/watch?v=1fN2UDbtGDQ)

---


## 1.21 Load Balancer Algorithm


### What is it?

A **load balancing algorithm** is the policy a load balancer (L4 or L7) uses to pick **one backend** from a healthy pool for each incoming connection or request. The algorithm sees only what the LB tracks—connection counts, weights, hashes of IP/URL/header—not application-level "CPU busy" unless exposed via custom metrics or slow-start.

**Core algorithms (tier-1):**

| Algorithm | Selection rule |
|-----------|----------------|
| **Round robin** | Rotate through backends in fixed order |
| **Weighted round robin** | Round robin proportional to weight |
| **Least connections** | Backend with fewest active connections |
| **IP hash** | `hash(client_ip) % N` → fixed backend |
| **Consistent hash** | Hash into ring; minimal remap when nodes added/removed |

### Why it matters

The wrong algorithm creates **false capacity**: three servers at 90% CPU while two sit idle, or sticky sessions break when mobile clients change IP. Algorithm choice interacts with **connection duration**, **request cost variance**, and **statefulness**.

| Workload shape | Bad choice | Symptom |
|----------------|------------|---------|
| Long WebSocket sessions | Round robin | Even count but one server holds all heavy rooms |
| Heterogeneous instance sizes | Plain round robin | Small nodes overwhelmed |
| In-memory session cache | Round robin | Cache miss storm on every server |
| Sharded cache cluster | Round robin | Same key on all nodes—no locality |

**Interview point:** L4 LB sees TCP connections; L7 LB sees HTTP requests. One HTTP/2 connection with 100 streams = **1 connection** for least-conn but **100 requests** for per-request RR.

### How it works

**Round robin (RR)**

```text
Backends: [A, B, C]
Request 1 → A,  Request 2 → B,  Request 3 → C,  Request 4 → A, ...
```

Pseudo-code:

```text
index = 0
on each request:
    backend = pool[index % pool.size]
    index++
    forward(backend)
```

Assumes **equal capacity**, **stateless** handlers, and **uniform** request cost.

**Weighted round robin (WRR)**

```text
Weights: A=3, B=1, C=1  →  pattern A,A,A,B,C repeating
```

Use when nodes differ (large vs small instances, canary with weight=1 vs prod weight=9).

**Least connections**

```text
On each new connection:
    pick backend with minimum active_connection_count
    increment count on assign; decrement on close
```

Best when requests hold connections for **variable or long** durations (TLS, DB through LB, WebSocket, gRPC streams).

```mermaid
flowchart TB
    Req[New TCP connection] --> LC{Least connections}
    LC --> B1["Server A: 120 conn"]
    LC --> B2["Server B: 45 conn  <- chosen"]
    LC --> B3["Server C: 89 conn"]
```

**IP hash (session affinity without cookies)**

```text
backend_index = hash(client_src_ip) % number_of_backends
```

Same client IP → same backend until pool size changes (then **most** mappings reshuffle—unlike consistent hash).

**Consistent hashing**

Place backends and keys on a **hash ring** (0 to 2³²-1). Key (URL, user id, cookie) walks clockwise to first backend.

```mermaid
flowchart LR
    subgraph ring [Hash ring]
        N1[Node A]
        N2[Node B]
        N3[Node C]
    end
    K1["hash user:42"] --> N2
    K2["hash user:99"] --> N3
```

When a node is **added/removed**, only keys **adjacent** to that node on the ring move—not all keys (unlike `hash % N`).

**Worked example — RR vs least conn:**

```text
5 backends, 5 new connections/sec, each lives 60 seconds

Round robin: each backend gets 1 conn/sec → 60 active each (balanced)

If 1 connection is a 24h WebSocket and others are 1s HTTP:
  RR: unlucky backend holds WebSocket + fair share of short — skewed load
  Least conn: new shorts avoid the WebSocket-heavy backend
```

**Consistent hash + virtual nodes (vnodes):**

Each physical server gets **many** points on the ring (e.g., 100) to spread load evenly when server count is small.

```text
add_server("B"):
  only keys between A's vnode and B's vnodes move from A to B
  (~1/N of keys move for N servers)
```

### Key details

| Algorithm | Best for | Avoid when |
|-----------|----------|------------|
| **Round robin** | Homogeneous stateless APIs, short requests | Long connections, skewed work |
| **Weighted RR** | Mixed instance sizes, canary traffic split | Need dynamic load awareness |
| **Least connections** | WebSocket, gRPC streams, LDAP, DB pool LB | Connection counting expensive at huge scale |
| **IP hash** | Simple stickiness without app cookies | Mobile/carrier NAT (many users → one IP) |
| **Consistent hash** | Distributed caches, sharded state | Hot keys dominate one node |
| **Random** | Surprisingly even at scale; zero state | Need stickiness |
| **Least response time** | Heterogeneous latency (LB adds RTT probe) | Noisy measurements |

**Health checks interact with algorithms:** unhealthy backends removed from pool → connections redistributed → possible **thundering herd** on remaining nodes.

**Maglev hashing (Google):** Deterministic permutation table—fast lookup, used in some L4/L7 proxies.

**Interview point:** Consistent hash solves **cache locality** when adding/removing nodes; IP hash is simpler but remaps almost everything when N changes.

### Production rules

#### NAT, mobile, and affinity pitfalls

| Client type | IP stability | Safe affinity key | Risk |
|-------------|--------------|-------------------|------|
| **Desktop broadband** | Stable hours–days | Source IP hash OK | CGNAT still collides |
| **Mobile 4G/5G** | Changes on tower handoff | **Cookie / user-id hash** | IP hash → mid-session backend switch |
| **Corporate NAT** | Thousands share one IP | Never IP hash for load spread | One backend gets NAT avalanche |
| **IPv6 privacy addresses** | Rotates (RFC 4941) | Cookie or auth token | IP hash useless |
| **WebSocket long-lived** | IP may change on reconnect | Sticky cookie + pub/sub backplane | See [1.22](#122-sse-polling--websockets) |

```text
Decision tree:
  Stateless REST API        → round robin or least conn (no stickiness)
  In-memory session         → externalize session OR cookie stickiness
  Public mobile API         → NEVER IP hash for distribution
  Sharded cache             → consistent hash on tenant_id / cache key
  Carrier NAT office egress → WRR or least conn; not IP hash
```

#### Sizing and tuning

| Workload | Algorithm | Weight / conn limit hint |
|----------|-----------|---------------------------|
| Homogeneous K8s pods (8 vCPU each) | Round robin | Equal weights |
| Mixed ASG (4 vCPU + 16 vCPU) | Weighted RR | Weight ∝ CPU or measured RPS |
| WebSocket fanout | Least connections | 1 conn = 1 user; cap ~10k–50k conn/instance |
| Memcached via LB | Consistent hash + vnodes | 100–200 vnodes per physical node |
| Canary 5% | Weighted RR | new=1, stable=19 |

**HTTP/2 note:** One TCP connection multiplexes many requests → **least connections** at L4 understates load; prefer L7 per-request RR or enable connection coalescing awareness.

### When to use

- **Round robin:** Default for stateless REST behind homogeneous pods (Kubernetes Service default).
- **Weighted RR:** Canary deploys (90/10), mixed instance types in ASG.
- **Least connections:** API gateway to WebSocket servers, TCP proxy to legacy app servers.
- **IP hash / consistent hash:** Memcached/Redis client-less clustering through LB, sticky sessions without `Set-Cookie`.
- **Consistent hash on URL:** CDN origin selection, API sharding gateway (`user_id` in path).

### Trade-offs / Pitfalls

| Pitfall | Why it hurts | Mitigation |
|---------|--------------|------------|
| RR ignores live load | Slow server gets equal share | Least conn or adaptive LB |
| IP hash + mobile users | IP changes mid-session → new backend | App session cookie or user-id hash |
| IP hash + carrier NAT | Thousands of users share one IP → one backend hot | Don't use IP hash for public mobile APIs |
| Consistent hash hot keys | Celebrity user id overloads one shard | Salting, sub-shards, application-level split |
| Least conn bookkeeping | CPU/memory per connection table | Sampling approximations at extreme scale |
| Wrong weights in WRR | Small node starved or large node overloaded | Measure CPU/RPS; tune weights |
| HTTP/2 multiplexing + RR | One TCP carries many requests to one pod | L7 per-request balancing or more pods |
| Removing backend without drain | In-flight requests killed | Connection draining, graceful shutdown |

**Comparison — hash % N vs consistent hash (3 → 4 backends):**

```text
hash % N:     ~75% of keys remap (almost full reshuffle)
consistent:   ~25% of keys remap (only new node's arc)
```

### References

- [Load Balancer Algorithms — comparison video](https://www.youtube.com/watch?v=1fN2UDbtGDQ)

---


## 1.22 SSE, Polling & WebSockets


### What is it?

Three families of techniques for **real-time or near-real-time** communication between client and server:

| Pattern | Direction | Connection |
|---------|-----------|------------|
| **Short polling** | Client pulls | Repeated HTTP requests every N seconds |
| **Long polling** | Client pulls (held) | HTTP request stays open until event or timeout |
| **SSE (Server-Sent Events)** | Server pushes | One persistent HTTP stream (`text/event-stream`) |
| **WebSocket** | Bidirectional | Single TCP connection upgraded from HTTP; full-duplex framed messages |

**WebSocket** is a protocol (`ws://` or `wss://`) enabling **bidirectional, full-duplex** communication over a **persistent** connection with minimal per-message overhead after the initial handshake.

### Why it matters

Choosing the wrong pattern wastes bandwidth (polling empty responses), ties up threads (sync long polling), or over-engineers (WebSocket when SSE suffices). Live chat, notifications, collaborative editing, stock tickers, and multiplayer games all depend on picking the right mechanism.

**WebSocket use cases:**
- **Live chat** - customer support, livestream chat, team messaging
- **Broadcast** - sports scores, traffic, stock quotes, news alerts (often combined with pub/sub)
- **Data sync** - DB change pushed to all connected clients (polls, live dashboards)
- **Multiplayer collaboration** - cursors, presence, shared documents (Figma-style)
- **In-app notifications** - event-driven alerts
- **Live location** - rideshare, fleet tracking, delivery ETA

### How it works

**a) Short polling**

1. Client calls API every 1-2 seconds (e.g. `GET /messages?since=...`)
2. Server returns current state; often **empty** when nothing changed
3. High HTTP overhead; poor for true real-time

**b) Long polling**

1. Client sends HTTP request; server **holds** it until data arrives or timeout
2. Client immediately opens new long poll after response
3. Challenges: message ordering with multiple parallel connections; still reconnects after each timeout

**c) Server-Sent Events (SSE)**

1. Client: `GET /events` with `Accept: text/event-stream`
2. Server keeps connection open; pushes `data: {...}\n\n` lines
3. Browser `EventSource` API auto-reconnects on disconnect
4. **One-way only** (server -> client); client still uses normal HTTP for commands

**d) WebSocket**

1. **TCP connection** established (same as HTTP)
2. **HTTP upgrade handshake:**
   - Client: `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key`
   - Server: `101 Switching Protocols`, `Sec-WebSocket-Accept`
3. Subprotocol negotiated (e.g. `json`, `mqtt`)
4. Connection switches to **WebSocket framing** - async bidirectional messages
5. Uses `ws://` (plain) or **`wss://`** (TLS) - always use `wss://` in production

```mermaid
sequenceDiagram
    participant Client
    participant Server
    Client->>Server: HTTP GET + Upgrade: websocket
    Server-->>Client: 101 Switching Protocols
    Note over Client,Server: Persistent WebSocket connection
    Client->>Server: { type: "subscribe", channel: "chat" }
    Server->>Client: { type: "message", text: "hello" }
    Client->>Server: { type: "message", text: "hi" }
```

**Comparison at a glance:**

| | Short poll | Long poll | SSE | WebSocket |
|---|------------|-----------|-----|-----------|
| Direction | Pull | Pull | Server push | Bidirectional |
| Overhead | Very high | Medium | Low | Lowest after handshake |
| Real-time | Poor | Better | Good | Best |
| Browser API | fetch | fetch | EventSource | WebSocket |
| Proxy friendly | Yes | Mostly | Yes | Needs L7 proxy support |

**Scaling WebSockets across servers:**

- Connections are **stateful** - server holds socket per client
- **Sticky sessions** (session affinity) route same client to same instance, OR
- **Pub/sub backplane** (Redis, Kafka) so any server can push to clients on other nodes
- Load balancer must support **HTTP upgrade** (L7 ALB/nginx, not naive L4 TCP drain)

**Popular libraries:** Socket.IO (reconnect + fallbacks), SignalR (.NET), SockJS (fallback transports), `ws` (Node.js minimal)

### Key details

- **When NOT to use WebSocket:** CRUD-heavy apps with no realtime need -> HTTP is simpler; audio/video streaming -> WebRTC; server-only push of text -> **SSE is simpler and scales easier**
- **Drawbacks of WebSocket:** stateful connections consume memory; harder to scale than stateless HTTP; some corporate proxies/firewalls block WS; no built-in reconnect spec (app must implement); presence/detection of disconnects is imperfect
- **Security:** use `wss://`; validate `Origin` header; authenticate during handshake (JWT in query or cookie); guard against XSS injecting into WS messages
- **SSE advantages:** automatic reconnect; works over standard HTTP/2; simpler ops than WS cluster
- **HTTP/1.1 connection limit** (~6 per domain) matters less once upgraded to single WS

### Production rules

#### Sticky sessions vs shared state

| Approach | How | Pros | Cons |
|----------|-----|------|------|
| **LB cookie stickiness** | `AWSALB` / `SERVERID` cookie | Simple; no app change | Uneven load; lost on cookie clear; deploy must drain |
| **IP hash** | L4/L7 hash of source IP | No cookie | NAT/mobile breaks; hot spots |
| **External session store** | Redis/DB for session | Even LB distribution; survives deploy | Extra infra; latency |
| **Pub/sub backplane** | Redis Pub/Sub, Kafka, NATS | Any node can push to any client | Complexity; backplane SPOF without cluster |

```text
Prefer: external session store + round robin (stateless nodes)
Use sticky only when: legacy app with in-memory state you cannot refactor yet
Always pair sticky with: connection drain on deploy (1.20)
```

#### Pub/sub backplane runbook

```text
Architecture:
  Client WS → Server A (subscribes user:123 on Redis channel)
  Event on Server B → PUBLISH user:123 → Server A → push to client socket

Runbook — "messages not delivered cross-server":
1. Verify all app instances connected to same Redis cluster (not localhost)
2. Check channel naming matches (tenant prefix, user id)
3. Monitor Redis: connected_clients, pubsub_channels, memory
4. On Redis failover: clients must reconnect; expect brief message gap
5. Scale: Redis Pub/Sub does not persist — missed messages if subscriber offline
   → for guaranteed delivery use Kafka + per-user consumer or SSE with offset
```

#### Sizing

| Resource | Rule of thumb | Notes |
|----------|---------------|-------|
| WS connections per instance | 10k–65k (depends on RAM, msg size) | ~10–50 KB per idle connection |
| Heartbeat interval | 30 s ping/pong | Detect dead peers; keep NAT binding alive |
| Redis Pub/Sub fanout | O(subscribers) per message | Hot channel (1M subs) → shard channels or edge broadcast |
| Reconnect storm | Plan 2× normal connect rate after outage | Jittered exponential backoff on client |
| LB idle timeout | > app heartbeat interval | NAT UDP/TCP timeout 30–120 s — align WS ping |

### When to use

| Need | Choose |
|------|--------|
| Updates every 30+ seconds, simple app | Short polling |
| Near-real-time, server push only (scores, logs, notifications) | **SSE** |
| Chat, gaming, collaboration, bidirectional sync | **WebSocket** |
| Firewall/proxy uncertainty | SSE or long polling first; SockIO fallbacks |

### Trade-offs / Pitfalls

| Pitfall | Consequence | Mitigation |
|---------|-------------|------------|
| Short polling | Empty responses waste bandwidth | SSE or WebSocket when updates are frequent |
| Long polling | Ties up worker threads if sync | Async server (Node, Netty) |
| SSE one-way | Client commands need separate channel | HTTP POST or companion API |
| WebSocket multi-server | Messages lost without shared state | Sticky sessions or pub/sub backplane |
| Connection storms on reconnect | LB/origin overload after outage | Jittered backoff; rate limit handshakes |
| Default to WebSocket everywhere | Unnecessary ops complexity | SSE for server-push-only feeds |
| Sticky without drain | Deploy kills active WS; mass disconnect | Drain + client reconnect with backoff |
| Pub/sub without persistence | Gap during subscriber reconnect | Kafka or client-side catch-up API |
| Redis backplane SPOF | All cross-node push fails | Redis Cluster / Sentinel |
| LB L4 only for WS | Upgrade headers stripped | L7 ALB/nginx with `proxy_http_version 1.1` |

### References

- [WebSockets - Hareram Singh (use cases, handshake, polling vs SSE comparison)](https://medium.com/@hareramcse/websockets-74244f33bff4)
- [SSE, Polling, and WebSockets - video](https://www.youtube.com/watch?v=WS352jTTkPU)

---

[<- Back to master index](../README.md)
