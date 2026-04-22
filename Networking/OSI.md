# 🌐 OSI Model — Revision Notes

---

## 1. Why Does the OSI Model Exist?

In the early days of networking, every vendor (IBM, Honeywell, DEC) had their own proprietary networking architecture — their devices couldn't communicate with each other.

In **1984**, the ISO published the **OSI (Open Systems Interconnection) model** as a **universal reference framework** — a contract that defines *what responsibility* each layer has, not *how* to implement it. As long as each layer keeps its contract, devices from different vendors can work together.

> **Used today as:** A mental model for reasoning about where problems/protocols live. Real-world internet uses **TCP/IP** (4 layers), but OSI's 7 layers offer more precision for debugging and design.

### 🧠 Mnemonic (Bottom → Top)
```
Please  Do    Not   Throw  Sausage  Pizza  Away
  1      2     3      4       5       6      7
Physical Data  Net  Transport Session Pres  App
```

---

## 2. The 7 Layers — Quick Reference

| Layer | Name | Data Unit | Key Devices | Key Protocols |
|---|---|---|---|---|
| 7 | Application | Data | — | HTTP, HTTPS, FTP, SSH, DNS, SMTP |
| 6 | Presentation | Data | — | TLS/SSL, gzip, JSON, UTF-8 |
| 5 | Session | Data | — | NetBIOS, RPC, SIP |
| 4 | Transport | Segment / Datagram | — | TCP, UDP |
| 3 | Network | Packet | Router | IP, ICMP, ARP, BGP, OSPF |
| 2 | Data Link | Frame | Switch | Ethernet, WiFi (802.11), MAC |
| 1 | Physical | Bits | Hub, NIC, Cables | Ethernet cable, Fiber, WiFi |

---

## 3. Each Layer Explained

### Layer 1 — Physical
**"Raw bits on a wire"**

- Transmits binary 0s and 1s over a physical medium (copper, fiber, radio waves)
- No concept of meaning — just moves signals from A to B
- Responsible for: connector types, cable specs, bit synchronization
- Devices: **Hubs**, **Repeaters**, **NICs**, **Cables**

| Medium | Speed | Max Distance |
|---|---|---|
| Cat5e Ethernet | 1 Gbps | 100m |
| Cat6a Ethernet | 10 Gbps | 100m |
| Single-mode Fiber | 100 Gbps | 40+ km |
| WiFi 6 (802.11ax) | 9.6 Gbps | ~50m |

> Nothing above Layer 1 works if the cable is broken.

---

### Layer 2 — Data Link
**"Structured bits within a local network"**

- Organises bits into **frames** and introduces **MAC addresses**
- MAC address = 48-bit hardware identifier burned into every NIC (e.g. `00:1A:2B:3C:4D:5E`)
- Responsibilities: framing, MAC addressing, error detection (checksum/FCS), media access control
- Devices: **Switches** (read MAC → forward to correct port), vs Hubs (Layer 1 — blindly broadcasts)

| Sublayer | Function |
|---|---|
| LLC (Logical Link Control) | Flow control, error checking |
| MAC (Media Access Control) | Addressing, medium access |

> Layer 2 only works **within one network**. To cross networks → need Layer 3.

---

### Layer 3 — Network
**"Routing across networks"**

- Assigns **IP addresses** (logical addresses) and routes **packets** across multiple networks
- Routers read destination IP → forward packet one hop at a time toward destination
- Also handles: fragmentation (break large packets for link MTU), path selection

| Protocol | Purpose |
|---|---|
| IPv4 / IPv6 | Addressing and routing |
| ICMP | Error reporting, diagnostics (`ping`, `traceroute`) |
| ARP | Maps IP address → MAC address |
| OSPF / BGP | Routing protocols (find best paths) |

**MAC vs IP:**
- MAC = permanent, hardware-level, local delivery only
- IP = logical, can change, used for routing across networks
- You need **both**: MAC for local hop, IP for end-to-end routing

> Layer 3 gets data to the right **machine**. It doesn't know which **application** should receive it.

---

### Layer 4 — Transport
**"Port-to-port delivery + reliability"**

- Adds **port numbers** to identify which application gets the data
- **Socket** = IP address + Port (e.g. `192.168.1.1:443`)
- Handles: segmentation, flow control, error recovery, ordering

#### TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection | 3-way handshake | Connectionless |
| Reliability | Guaranteed + retransmission | Best-effort, no guarantee |
| Ordering | In-order delivery | No ordering |
| Overhead | High (headers, ACKs) | Minimal |
| Speed | Slower | Faster |
| Use cases | HTTP, email, file transfer | DNS, video streaming, gaming, VoIP |

#### TCP 3-Way Handshake
```
Client          Server
  │──── SYN ────►│
  │◄─── SYN-ACK─│
  │──── ACK ────►│
  │   Connected  │
```

#### Port Ranges
| Range | Type | Examples |
|---|---|---|
| 0–1023 | Well-known | HTTP (80), HTTPS (443), SSH (22), DNS (53) |
| 1024–49151 | Registered | MySQL (3306), PostgreSQL (5432) |
| 49152–65535 | Dynamic/Ephemeral | Client-side connections |

---

### Layer 5 — Session
**"Connection lifecycle management"**

- Manages setup, maintenance, and teardown of sessions between applications
- Handles: session checkpointing (resume downloads after disconnect), synchronisation
- In practice: mostly **folded into Application layer** in modern protocols

| Protocol | Function |
|---|---|
| NetBIOS | Name resolution and session management |
| RPC | Remote procedure call sessions |
| SIP | VoIP session setup and teardown |

---

### Layer 6 — Presentation
**"Translation, encryption, compression"**

- Converts data between app format and network format
- Three main jobs: **Encryption**, **Compression**, **Encoding/Format conversion**
- In practice: mostly **folded into Application layer** (e.g. TLS runs as part of HTTPS)

| Function | Examples |
|---|---|
| Encryption | TLS/SSL, AES |
| Compression | gzip, Brotli, deflate |
| Encoding | ASCII, UTF-8, Base64 |
| Data Formats | JPEG, PNG, JSON, MPEG |

---

### Layer 7 — Application
**"What the user/app sees"**

- Highest layer — the protocols and interfaces end users and applications interact with directly
- Handles: HTTP requests, DNS lookups, email sending, authentication

| Protocol | Port | Purpose |
|---|---|---|
| HTTP | 80 | Web (unencrypted) |
| HTTPS | 443 | Secure web |
| FTP | 21 | File transfer |
| SSH | 22 | Secure remote access |
| SMTP | 25 | Sending email |
| DNS | 53 | Domain name resolution |
| DHCP | 67/68 | Dynamic IP assignment |

---

## 4. Data Flow — Encapsulation & Decapsulation

### Sending (Encapsulation — Layer 7 → Layer 1)
Each layer **wraps** data with its own header as it moves **down** the stack:

```
App Data  →  [HTTP Request]
               ↓ Layer 4 wraps with TCP header
[TCP Header | HTTP Request]              ← Segment
               ↓ Layer 3 wraps with IP header
[IP Header | TCP Header | HTTP Request]  ← Packet
               ↓ Layer 2 wraps with Ethernet header + trailer
[ETH | IP | TCP | HTTP | FCS]            ← Frame
               ↓ Layer 1 converts to bits
0101001010110...                         ← Bits on wire
```

### Receiving (Decapsulation — Layer 1 → Layer 7)
Each layer **strips** its header and passes payload **up**:
```
Bits → Frame → Packet → Segment → Data
         ↑         ↑         ↑
    ETH stripped  IP stripped  TCP stripped
```

### Data Unit Names

| Layer | Unit |
|---|---|
| 7 – Application | Data |
| 4 – Transport | Segment (TCP) / Datagram (UDP) |
| 3 – Network | Packet |
| 2 – Data Link | Frame |
| 1 – Physical | Bits |

---

## 5. OSI vs TCP/IP Model

| OSI Layer(s) | TCP/IP Layer | Protocols |
|---|---|---|
| 7 + 6 + 5 (App + Pres + Session) | Application | HTTP, FTP, DNS, SSH, TLS |
| 4 – Transport | Transport | TCP, UDP |
| 3 – Network | Internet | IP, ICMP, ARP |
| 2 + 1 (Data Link + Physical) | Network Access | Ethernet, WiFi, PPP |

**Why learn OSI if TCP/IP is used in practice?**
> OSI gives **7 precise layers** for pinpointing problems. "Layer 4 issue" = everyone knows it's TCP/UDP/ports. TCP/IP's single "Application" layer is too broad for this kind of diagnostic precision.

---

## 6. Where Common Technologies Live

| Technology | OSI Layer | Why |
|---|---|---|
| Ethernet cable, WiFi signal | Layer 1 | Raw bit transmission |
| Switch | Layer 2 | Forwards by MAC address |
| Router | Layer 3 | Routes by IP address |
| Firewall (packet filter) | Layer 3–4 | Filters by IP + port |
| Load Balancer (L4) | Layer 4 | Routes by TCP/UDP port |
| Load Balancer (L7) | Layer 7 | Routes by URL, headers, cookies |
| TLS/SSL | Layer 6 (or 4/7) | Encrypts transport data |
| HTTP/HTTPS | Layer 7 | Application protocol |
| DNS | Layer 7 | Name resolution |
| Ping / Traceroute | Layer 3 | ICMP diagnostic |

---

## 7. Quick Summary

| Layer | One-line job | Remember by |
|---|---|---|
| 7 – Application | User-facing protocols (HTTP, DNS, SSH) | What apps use |
| 6 – Presentation | Encrypt, compress, encode | TLS, gzip, JSON |
| 5 – Session | Open, maintain, close connections | SIP, RPC |
| 4 – Transport | Port-to-port, TCP vs UDP | Reliability choice |
| 3 – Network | IP routing across networks | Routers, IP |
| 2 – Data Link | MAC addressing within local network | Switches, frames |
| 1 – Physical | Raw bits on a wire | Cables, hubs |

---

---

# 🎯 Most Asked Interview Questions (Scenario-Based)

---

**Q1. A user types `https://google.com` in their browser. Walk through what happens at each OSI layer.**

> **Layer 7 (Application):** Browser prepares an HTTPS/HTTP GET request for `google.com`.
>
> **Layer 6 (Presentation):** TLS handshake encrypts the request; data is encoded (UTF-8, JSON formatting).
>
> **Layer 5 (Session):** A TLS session is established and tracked for the connection's lifetime.
>
> **Layer 4 (Transport):** TCP 3-way handshake occurs (SYN → SYN-ACK → ACK). Data is broken into segments with source/destination ports (source: ephemeral, destination: 443).
>
> **Layer 3 (Network):** DNS resolves `google.com` → IP (e.g. `142.250.80.46`). IP header added with source + destination IP. Router hops determine the path.
>
> **Layer 2 (Data Link):** ARP resolves the next-hop router's IP → MAC address. Ethernet frame created with source/destination MAC.
>
> **Layer 1 (Physical):** Frame converted to electrical signals (or light pulses on fiber / radio waves on WiFi) and sent over the wire.
>
> On Google's side, decapsulation reverses this: bits → frame → packet → segment → data, until the HTTP GET request reaches the web server.

---

**Q2. What is the difference between a hub, a switch, and a router? Which OSI layer does each operate at?**

> **Hub (Layer 1 — Physical):** Receives a signal and blindly broadcasts it to every connected port. No intelligence — every device sees every packet. Creates unnecessary traffic and collisions.
>
> **Switch (Layer 2 — Data Link):** Reads the destination **MAC address** in each Ethernet frame. Maintains a MAC address table mapping ports to MACs. Forwards the frame only to the correct port. More efficient — traffic is isolated per connection.
>
> **Router (Layer 3 — Network):** Reads the destination **IP address** in each packet. Makes routing decisions to forward packets toward the destination across different networks. Required to connect your home network to the internet.
>
> Summary: Hub = dumb repeater. Switch = smart local traffic director. Router = cross-network traffic director.

---

**Q3. What is the difference between Layer 4 and Layer 7 load balancers? When would you use each?**

> **L4 Load Balancer (Transport Layer):** Routes traffic based on IP address and TCP/UDP port. It doesn't inspect the actual content of requests. Very fast — just redirects the TCP stream. Used when you need high throughput and low overhead. Example: routing all port-443 traffic to a pool of servers without caring about URL paths.
>
> **L7 Load Balancer (Application Layer):** Reads the full HTTP request — URL path, headers, cookies, request body. Enables smart routing: send `/api/*` to API servers, `/images/*` to image servers, route based on `Authorization` header, do SSL termination. More flexible but slightly more overhead. Tools: **Nginx**, **HAProxy**, **AWS ALB**.
>
> **When to use L4:** Simple, very high-volume TCP routing, gaming servers, raw TCP proxying.
> **When to use L7:** Microservices routing, canary deployments, A/B testing, path-based routing, authentication at the gateway.

---

**Q4. Why does TCP use a 3-way handshake? What problem does it solve?**

> TCP is designed for **reliable, ordered data delivery**. Before sending data, both sides need to agree that:
> 1. The client can reach the server (SYN)
> 2. The server can reach the client (SYN-ACK)
> 3. The client acknowledges the server's response (ACK)
>
> This establishes **sequence numbers** on both sides, which TCP uses to reorder out-of-order packets and detect missing ones. Without this handshake, TCP couldn't guarantee ordered delivery or know which sequence number to start retransmitting from.
>
> **Cost:** Adds 1.5 round trips of latency before data can flow. This is why HTTP/3 (QUIC) moved to UDP + application-level reliability, reducing connection setup to 0-RTT or 1-RTT.

---

**Q5. You're designing a real-time multiplayer game. Would you use TCP or UDP? Why?**

> **UDP** is the right choice. In a real-time game, **latency matters more than reliability**. If a position update packet is lost, retransmitting it is useless — the player has already moved. Receiving a 200ms-old position is worse than receiving no update and interpolating.
>
> TCP's retransmission causes **head-of-line blocking**: if one packet is lost, delivery of subsequent packets stalls until the lost one is retransmitted — introducing unpredictable latency spikes called jitter, which are fatal for real-time games.
>
> With UDP, the game builds its own lightweight reliability: send position updates at 60Hz, handle packet loss by just skipping that frame, use sequence numbers to discard out-of-order packets. Only critical events (player death, score) use reliable UDP (or a small TCP channel for critical messages).
>
> Examples: **Quake**, **Counter-Strike**, **Valorant** all use UDP.

---

**Q6. What happens at Layer 3 when a packet travels from your laptop to a server in another country? How do routers know where to send it?**

> Your packet has a **destination IP** (e.g. `13.225.100.1`). Your laptop's first hop is the **default gateway** (your home router). The router checks its **routing table** — a list of IP prefix ranges and which interface/next-hop to use.
>
> If no specific route matches, the packet goes to the **ISP's router**, which has a much larger routing table (learned via **BGP — Border Gateway Protocol**). BGP is how ISPs and large networks announce which IP ranges they own. Each router along the path does a **longest prefix match** on the destination IP and forwards the packet to the next hop — one hop at a time — until it reaches the destination network.
>
> `traceroute` shows each of these Layer 3 hops. Each hop decrements the packet's **TTL (Time To Live)** by 1. If TTL hits 0, the router drops the packet and sends an ICMP "Time Exceeded" message back — preventing infinite loops.

---

**Q7. What is ARP and why is it needed? What layer does it operate at?**

> **ARP (Address Resolution Protocol)** operates at **Layer 2/3 boundary** — it bridges the gap between IP addresses (Layer 3) and MAC addresses (Layer 2).
>
> When your computer wants to send a packet to `192.168.1.1` (your router), it knows the IP but needs the MAC address to create an Ethernet frame. ARP broadcasts: *"Who has IP 192.168.1.1? Tell me your MAC."* The router responds with its MAC. Your computer caches this in its **ARP table** (`arp -a`).
>
> Without ARP, Layer 3 routing couldn't hand off to Layer 2 for local delivery. Every cross-hop packet needs ARP to resolve the **next-hop's MAC**, even though the destination IP stays the same throughout the journey.

---

**Q8. Where does TLS fit in the OSI model? Explain the TLS handshake briefly.**

> TLS technically spans **Layer 6 (Presentation)** — it encrypts data — but in TCP/IP terms it sits between Layer 4 (TCP) and Layer 7 (HTTP). It's often called "Layer 4.5" or just described as part of the Application layer.
>
> **TLS Handshake (TLS 1.3, simplified):**
> 1. **Client Hello** — client sends supported cipher suites and a random number
> 2. **Server Hello** — server picks a cipher, sends its certificate (containing its public key)
> 3. **Client verifies** the certificate against a trusted CA (Certificate Authority)
> 4. **Key exchange** — client and server derive a shared symmetric key (using Diffie-Hellman)
> 5. **Finished** — both sides confirm the handshake; encrypted data can now flow
>
> TLS 1.3 reduced this to **1-RTT** (down from 2-RTT in TLS 1.2), and supports **0-RTT resumption** for reconnecting clients — important for latency-sensitive apps.

---

**Q9. A network engineer says "it's a Layer 2 issue." What does that mean and how would you troubleshoot it?**

> A **Layer 2 issue** means the problem is in the Data Link layer — frames aren't being delivered correctly within the local network segment. This means IP routing is fine (Layer 3 is OK) but the local delivery is broken.
>
> **Common Layer 2 issues:**
> - **Incorrect VLAN** configuration — device is in the wrong VLAN, so its traffic is isolated
> - **MAC table overflow** on a switch — switch starts broadcasting all traffic, causing congestion
> - **Duplex mismatch** — one side is full-duplex, other is half-duplex → collisions
> - **Spanning Tree loop** — network loop causing broadcast storms
> - **ARP poisoning** — attacker sends fake ARP replies, redirecting traffic (Layer 2 security issue)
>
> **Troubleshoot with:** `arp -a` (check ARP table), `ping` local gateway (Layer 3 test), check switch port status, verify VLAN assignments, `tcpdump` / Wireshark at the frame level.

---

**Q10. How does a firewall differ from a router? At which OSI layers do firewalls operate?**

> A **router** (Layer 3) simply forwards packets toward their destination based on routing tables — it doesn't make security decisions.
>
> A **firewall** adds security by filtering traffic. Depending on its type, it operates at different layers:
>
> | Firewall Type | Layer | How it works |
> |---|---|---|
> | Packet Filter | L3–L4 | Allows/blocks by source IP, dest IP, port, protocol |
> | Stateful Firewall | L4 | Tracks TCP connection state — only allows return traffic for established connections |
> | Application Firewall (WAF) | L7 | Inspects HTTP content — blocks SQL injection, XSS, blocks specific URL patterns |
>
> A **WAF (Web Application Firewall)** like AWS WAF or Cloudflare operates at **Layer 7** — it can read HTTP headers, URLs, and request bodies to detect attacks. A simple **Security Group** on AWS is a stateful Layer 4 firewall — it allows/blocks by IP + port. Deep packet inspection (DPI) firewalls scan all the way to Layer 7 for threat patterns.