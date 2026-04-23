# 🔌 TCP vs UDP — Revision Notes

---

## 1. The Role of the Transport Layer (Layer 4)

The **Transport Layer** sits between the Network Layer (IP) and the Application Layer. Its job is to deliver data **between specific processes** on two machines — not just between machines.

It provides:
- **Multiplexing** — multiple apps share one network connection via **ports**
- **Segmentation** — breaks large messages into smaller chunks for transmission
- **Reassembly** — puts chunks back together at the destination
- **Error handling** — either detects + fixes errors (TCP) or detects-only (UDP)

> A **socket** = IP Address + Port. This uniquely identifies a process on a machine.
> Example: `192.168.1.10:443` = HTTPS server process on that machine.

The two core Transport Layer protocols are **TCP** and **UDP** — they make fundamentally different tradeoffs.

---

## 2. TCP — Transmission Control Protocol

### What is TCP?
TCP is a **connection-oriented, reliable** protocol. Before any data is sent, a connection must be established. Every byte sent is acknowledged, and lost data is retransmitted.

> Think of TCP like a **phone call** — you dial, the other side picks up, you confirm you can hear each other, then you talk. Every word is heard or repeated.

### The TCP 3-Way Handshake (Connection Setup)

```
Client                      Server
  │                            │
  │──── SYN (seq=x) ──────────►│   "I want to connect"
  │                            │
  │◄─── SYN-ACK (seq=y,ack=x+1)│   "OK. Ready. Got your seq?"
  │                            │
  │──── ACK (ack=y+1) ────────►│   "Yes. Let's go."
  │                            │
  │      ← Data flows now →    │
```

**Cost:** 1.5 RTTs before first byte of data can be sent.

### TCP Connection Teardown (4-Way)

```
Client                      Server
  │──── FIN ──────────────────►│
  │◄─── ACK ───────────────────│
  │◄─── FIN ───────────────────│
  │──── ACK ──────────────────►│
         Connection closed
```

### How TCP Ensures Reliability

#### Sequence Numbers & Acknowledgements
- Every byte is numbered with a **sequence number**
- Receiver sends back **ACK** with the next expected byte number
- If ACK not received within timeout → **retransmit**

```
Sender:   [Seg 1: bytes 1–1000]→ [Seg 2: 1001–2000]→ [Seg 3: 2001–3000]
Receiver: ACK 1001              → ACK 2001           → ACK 3001
```

#### Flow Control — Sliding Window
- Receiver advertises a **window size** (available buffer space)
- Sender never sends more unacknowledged data than the window allows
- Prevents fast sender from overwhelming a slow receiver

```
Receiver tells sender: "window = 16384 bytes (16KB)"
Sender: can have up to 16KB of unacknowledged data in flight at once
```

#### Congestion Control
TCP automatically reduces send rate when network congestion is detected.

| Phase | Behaviour |
|---|---|
| **Slow Start** | Begin with small window, double every RTT until threshold |
| **Congestion Avoidance** | Grow window by 1 MSS per RTT (linear) after threshold |
| **Fast Retransmit** | Retransmit after 3 duplicate ACKs — don't wait for timeout |
| **Fast Recovery** | Halve window on loss; don't go back to slow start |

### TCP Header (20 bytes minimum)

| Field | Size | Purpose |
|---|---|---|
| Source Port | 16 bits | Sender's port number |
| Destination Port | 16 bits | Receiver's port number |
| Sequence Number | 32 bits | Position of first byte in this segment |
| Acknowledgement No. | 32 bits | Next expected byte from the other side |
| Window Size | 16 bits | Receiver's available buffer |
| Checksum | 16 bits | Error detection |
| Flags | 6 bits | SYN, ACK, FIN, RST, PSH, URG |

---

## 3. UDP — User Datagram Protocol

### What is UDP?
UDP is a **connectionless, best-effort** protocol. It fires datagrams at the destination with no handshake, no acknowledgement, and no retransmission.

> Think of UDP like **sending a postcard** — you write it and drop it in the mailbox. No confirmation it arrived. No guaranteed order. But fast and cheap.

### How UDP Works
1. App hands data to UDP
2. UDP adds an 8-byte header
3. Datagram is sent **immediately** — no setup, no waiting
4. Receiver gets it (or doesn't) — no feedback to sender
5. **If the app needs reliability, it must build it itself**

### UDP Header (8 bytes — fixed, minimal)

| Field | Size | Purpose |
|---|---|---|
| Source Port | 16 bits | Sender's port |
| Destination Port | 16 bits | Receiver's port |
| Length | 16 bits | Header + data length |
| Checksum | 16 bits | Optional error detection |

> UDP header is **8 bytes** vs TCP's **20+ bytes** — significantly lower overhead per packet.

### What UDP Does NOT Provide
- ❌ No connection establishment
- ❌ No delivery guarantee
- ❌ No ordering
- ❌ No retransmission
- ❌ No congestion control
- ❌ No flow control

---

## 4. Head-to-Head Comparison

| Feature | TCP | UDP |
|---|---|---|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | Guaranteed delivery + retransmission | Best-effort, no guarantee |
| **Ordering** | Always in-order delivery | No ordering guarantee |
| **Flow Control** | Yes (sliding window) | No |
| **Congestion Control** | Yes (slow start, AIMD) | No |
| **Speed** | Slower (overhead, ACKs, handshake) | Faster (no setup, minimal overhead) |
| **Header Size** | 20–60 bytes | 8 bytes (fixed) |
| **Error Detection** | Checksum + recovery | Checksum only (no recovery) |
| **Broadcast/Multicast** | Not supported | Supported |
| **Use Cases** | HTTP, FTP, SSH, email, DB | DNS, VoIP, video stream, gaming |

---

## 5. Data Delivery Model

### TCP — Byte Stream (ordered)
```
App sends:    "Hello, World!"
TCP segments: [Hel] [lo, ] [Wor] [ld!]
TCP delivers: "Hello, World!"  ← perfectly reassembled, in order
```

### UDP — Datagram (independent packets)
```
App sends:    Packet 1, Packet 2, Packet 3
UDP delivers: Packet 1 ✅  Packet 3 ✅ (out of order)  Packet 2 ✗ (lost)
App receives: Packet 1, Packet 3   ← gaps, wrong order — app handles it
```

---

## 6. Head-of-Line (HoL) Blocking — TCP's Key Weakness

TCP delivers data **in strict order**. If one segment is lost, ALL later segments wait in the buffer — even if they've already arrived.

```
Sent:       [Seg 1] [Seg 2 LOST ✗] [Seg 3 ✅] [Seg 4 ✅]
Receiver:   Deliver Seg 1
            BLOCKED ← waiting for Seg 2 retransmit
            Seg 3, 4 sit in buffer unused
```

> Causes **unpredictable latency spikes (jitter)** — fatal for real-time apps like gaming and video calls.

UDP has **no HoL blocking** — every datagram is independent.

---

## 7. Modern Innovation — QUIC & HTTP/3

### Problem with TCP + TLS
- TCP handshake: **1 RTT**
- TLS 1.3 handshake: **1 additional RTT**
- First data: **2 RTTs minimum**
- TCP HoL blocking still applies across all HTTP/2 streams

### QUIC (Quick UDP Internet Connections)
Developed by Google, now the foundation of **HTTP/3**.

- Built **on top of UDP** — bypasses OS kernel's TCP implementation
- Combines transport + TLS 1.3 into **1 RTT handshake**
- **0-RTT resumption** for previously connected clients
- **Independent streams** — one lost packet only blocks that stream, not others
- **Connection migration** — connection survives IP address changes (WiFi → 4G)
- Mandatory encryption — no unencrypted QUIC

```
TCP + TLS 1.3:
  [TCP SYN]──►[SYN-ACK]──►[ACK + TLS Hello]──►[TLS Done]──►Data
  ├── 1 RTT ──┤            ├────── 1 RTT ──────┤
  Total: 2 RTTs before data

QUIC:
  [QUIC Initial (crypto + transport)]──►[Server response]──►Data
  ├────────────────── 1 RTT ──────────────────────────────┤
  Total: 1 RTT (0-RTT for returning clients)
```

### QUIC vs TCP

| Feature | TCP | QUIC (HTTP/3) |
|---|---|---|
| Base protocol | TCP | UDP |
| Encryption | Optional (add TLS) | Mandatory (built-in TLS 1.3) |
| Handshake cost | 2 RTT (TCP + TLS) | 1 RTT (0-RTT for resumption) |
| HoL Blocking | Yes | No (per-stream independence) |
| Connection migration | No (IP change = reconnect) | Yes (survives network switch) |
| Adoption | Universal | ~30% of web, default in Chrome |

---

## 8. Choosing the Right Protocol

```
Need guaranteed, ordered delivery?
        │
       YES → Use TCP
        │
        NO → Is low latency / real-time critical?
                │
               YES → Does app tolerate all loss?
                        │
                    YES → Plain UDP (DNS, DHCP)
                        │
                    NO  → UDP + custom reliability (gaming, VoIP)
               NO  → Use TCP
```

### Use TCP When:
- Data **must** arrive completely and in order
- Correctness > speed
- Examples: **HTTP/HTTPS, FTP, SSH, SMTP, database queries**

### Use UDP When:
- Speed and low latency > guaranteed delivery
- Some data loss is acceptable (old video frame = skip it)
- App can implement lightweight selective reliability
- Need broadcast or multicast
- Examples: **DNS, VoIP, video streaming, gaming, DHCP, SNMP**

---

## 9. Real-World Use Cases

| Application | Protocol | Why |
|---|---|---|
| Web browsing (HTTP/HTTPS) | TCP | Full page must load correctly |
| File transfer (FTP, SCP) | TCP | Every byte must arrive intact |
| Email (SMTP, IMAP) | TCP | Message integrity required |
| SSH | TCP | Commands must be reliable |
| DNS lookup | UDP | Single small round-trip; speed > reliability |
| DNS large responses / zone transfer | TCP | Fallback when response > 512 bytes |
| Video streaming (Netflix, YouTube) | UDP / QUIC | Tolerate minor loss; rebuffering is worse |
| VoIP / Video calls (Zoom, WhatsApp) | UDP | Real-time; old audio is worthless if delayed |
| Online gaming | UDP | Latency critical; stale state is useless |
| DHCP | UDP | Client has no IP yet — can't TCP handshake |
| SNMP monitoring | UDP | Lightweight polling; loss acceptable |
| HTTP/3 | QUIC (UDP) | Faster handshake, no HoL blocking |

---

## 10. Quick Summary

| Concept | Key Takeaway |
|---|---|
| **TCP** | Reliable, ordered, connection-oriented — pays with latency and overhead |
| **UDP** | Fast, lightweight, connectionless — app handles reliability if needed |
| **3-way handshake** | TCP setup costs 1.5 RTTs before data flows |
| **Sliding window** | TCP flow control — receiver caps sender's speed |
| **Congestion control** | TCP self-throttles under congestion (slow start, AIMD) |
| **HoL blocking** | TCP's biggest weakness — one lost packet stalls all others |
| **UDP header** | Only 8 bytes — minimal overhead per datagram |
| **QUIC** | UDP-based, 1-RTT handshake, no HoL blocking, built-in TLS — powers HTTP/3 |
| **DNS uses UDP** | Fast single round-trip; falls back to TCP for large responses |
| **Gaming uses UDP** | Stale position data is useless; latency beats reliability |

---

---

# 🎯 Most Asked Interview Questions (Scenario-Based)

---

**Q1. You're building a video conferencing app like Zoom. Which transport protocol do you use and why?**

> **UDP**. In a live video call, real-time latency matters far more than perfect reliability. If a video frame is lost, the right response is to skip it or interpolate — not wait for a retransmit (which would arrive too late to be useful). TCP's retransmission and head-of-line blocking would cause **jitter** (irregular delays), making the call choppy and unusable.
>
> In practice: Zoom uses **UDP with application-level FEC (Forward Error Correction)** — redundant data sent alongside packets so minor loss is recovered without retransmission. For signalling (joining, leaving, chat), it uses TCP/HTTPS since those must be reliable. This hybrid — **UDP for media, TCP for control** — is the standard pattern in WebRTC-based applications.

---

**Q2. DNS uses UDP. Under what conditions does it fall back to TCP?**

> DNS uses **UDP port 53** by default — queries and responses are tiny (under 512 bytes), so a single fast round-trip is ideal.
>
> DNS **switches to TCP** in these cases:
> 1. **Response too large** — if response exceeds 512 bytes (or 4096 with EDNS), the server returns a truncated packet with the TC flag set, signalling the client to retry over TCP
> 2. **Zone transfers (AXFR)** — secondary DNS server replicating an entire zone file from primary; data is large and must arrive completely
> 3. **DNSSEC responses** — cryptographic signatures make responses much larger
> 4. **DNS-over-TLS (DoT) / DNS-over-HTTPS (DoH)** — privacy-focused DNS variants, both TCP-based
>
> Key interview point: DNS is a great example of choosing UDP for performance, with TCP as a reliability fallback for edge cases.

---

**Q3. Explain TCP's sliding window and why it matters for high-latency connections.**

> The **sliding window** is TCP's flow control mechanism. The receiver advertises its **available buffer size** (window size), and the sender never has more unacknowledged bytes in flight than that window.
>
> On **high-latency links** (satellite internet, cross-continental connections), this becomes a hard bottleneck. Example: window = 65KB, RTT = 500ms → max throughput = 65KB / 0.5s = **~1 Mbps**, even on a 1 Gbps link. This is the **Bandwidth-Delay Product (BDP)** problem.
>
> **Fix:** TCP Window Scaling (RFC 1323) extends window size up to **1 GB**. This is why tuning `net.ipv4.tcp_rmem` and `tcp_wmem` matters for cross-region database replication and large file transfers over high-latency links. Without it, you're leaving most of the bandwidth unused.

---

**Q4. What is head-of-line blocking in TCP and how does QUIC solve it?**

> In TCP, data is delivered to the application in **strict sequential order**. If segment 2 is lost, segments 3, 4, 5 are held in the receive buffer and not passed to the app — even if they arrived — until segment 2 is retransmitted and received.
>
> This causes **latency spikes whenever any packet drops**, which is especially damaging for multiplexed HTTP/2 connections where dozens of resources share one TCP connection — a single lost packet blocks all of them.
>
> **QUIC solves this** by running **independent streams** over UDP. Each QUIC stream has its own sequencing. A lost packet only stalls the stream it belongs to — all other streams continue delivering data unaffected. This is why HTTP/3 loads pages faster than HTTP/2 under real-world packet loss conditions, even on the same network.

---

**Q5. You're designing a 5GB file upload service. Which protocol and what design considerations apply?**

> **TCP** — every byte must arrive intact and in order or the file is corrupted. Correctness is non-negotiable here.
>
> Key design considerations:
> - **Chunked multipart upload** — split the 5GB into chunks (e.g. 50–100MB each). Upload chunks in parallel or sequentially. On failure, resume from the last successful chunk. S3's multipart upload follows this exact pattern.
> - **Large TCP send/receive buffers** (`SO_SNDBUF`, `SO_RCVBUF`) — critical for high throughput on high-latency links (Bandwidth-Delay Product)
> - **Application-level checksum** (MD5 / SHA-256 per chunk) — TCP guarantees network-level integrity, not storage or application bugs
> - **HTTP/2 or HTTP/3** for the upload endpoint — reduces connection overhead when uploading multiple chunks
> - **Backpressure handling** — if the client uploads faster than the server can write to disk, TCP's flow control naturally throttles it via zero-window signals

---

**Q6. Why can't TCP broadcast? How does UDP handle it?**

> **TCP is strictly point-to-point** — requires a dedicated connection between exactly two endpoints. To send the same data to 1000 clients, you need 1000 separate TCP connections, 1000 handshakes, and 1000 independent ACK streams. This scales poorly.
>
> **UDP supports two one-to-many modes:**
> - **Broadcast** (`255.255.255.255`): sends one datagram to every device on the local subnet. Used by **DHCP** (client has no IP yet, so it broadcasts a discovery message) and **ARP**.
> - **Multicast** (address range `224.0.0.0–239.255.255.255`): only devices that have joined the multicast group receive the packet. One packet goes out, network infrastructure delivers it to all subscribers. Used by **IPTV**, **live sports streaming**, and **financial market data feeds** (one stock price update reaches thousands of trading terminals).
>
> Multicast is why live TV and financial feeds are far more network-efficient than unicast streaming to each subscriber individually.

---

**Q7. An online multiplayer game uses UDP. How does it handle reliability for critical events like player death?**

> The game **selectively applies reliability only where needed**, rather than paying TCP's overhead across all traffic:
>
> - **High-frequency state (position, rotation)** — fire-and-forget UDP. Sent 20–60× per second. A lost packet is immediately superseded by the next one, so retransmitting stale positions would make things worse.
> - **Sequence numbers** — every packet has a sequence number. Receiver discards duplicates and out-of-order packets with old state.
> - **Critical events (player death, score, item pickup)** — these use **application-level ACK + retransmit over UDP**. The sender keeps retransmitting until it receives an application-level acknowledgement.
> - **Forward Error Correction (FEC)** — send redundant packets (e.g. send packet n+1 encoded with info to reconstruct n) so minor loss is recovered without round-trip retransmit latency.
> - Some games open a **separate TCP connection** solely for critical game state, while UDP handles real-time updates.
>
> This gives the best of both worlds: sub-20ms latency for real-time state, guaranteed delivery for game-critical events.

---

**Q8. What happens when a receiver's TCP window fills up (zero window situation)?**

> When the receiver's buffer fills (app not reading fast enough), it advertises **window size = 0** in its ACK. The sender sees this and **immediately stops sending** — it cannot push any more data.
>
> To avoid deadlock, the sender starts sending **TCP Window Probe** packets at intervals (starting ~1s, exponentially backing off). These probes contain 1 byte of data to elicit an ACK. When the receiver drains its buffer and has space again, it sends a **Window Update** with a non-zero window, and the sender resumes.
>
> **Real-world diagnosis:** In Wireshark, `TCP ZeroWindow` followed by `TCP Window Full` indicates this condition. The bottleneck is **the receiving application not consuming data fast enough** — not the network. Fix: make the app process data faster, increase `SO_RCVBUF`, or use async I/O so the app never falls behind.

---

**Q9. Compare HTTP/1.1, HTTP/2, and HTTP/3 from a TCP/UDP and performance perspective.**

> **HTTP/1.1 (TCP):**
> - Without keep-alive: one request per TCP connection — new handshake for every resource
> - With keep-alive: requests are serial — request 2 waits for request 1's full response (application-level HoL blocking)
> - Workaround: browsers open 6 parallel TCP connections per origin — wasteful and limited
>
> **HTTP/2 (TCP):**
> - Multiplexes multiple requests as **streams** over a single TCP connection — eliminates application-level HoL blocking
> - Header compression (HPACK) reduces overhead
> - Still suffers from **TCP-level HoL blocking** — one dropped packet stalls all streams
> - Two RTTs before data (TCP handshake + TLS)
>
> **HTTP/3 (QUIC over UDP):**
> - Truly independent streams — one dropped packet only stalls its own stream
> - **1 RTT** total (0-RTT for repeat connections)
> - Built-in TLS 1.3 — always encrypted
> - Connection migration — mobile users switching from WiFi to 4G keep their session
> - Default in Chrome; ~30% of the web
>
> The evolution: each version solves the HoL blocking problem at a deeper layer, while also reducing connection setup cost.

---

**Q10. Your microservice makes thousands of short-lived DB connections. What TCP issue arises and how do you fix it?**

> **TIME_WAIT state exhaustion.** After a TCP connection closes, the side that initiated the close enters **TIME_WAIT** for `2 × MSL` (typically 60–120 seconds). During this period, the local port is held reserved — preventing the same (src IP, src port, dst IP, dst port) tuple from being reused.
>
> With thousands of short-lived connections, you exhaust the **ephemeral port range** (49152–65535 ≈ 16K ports). New connections fail with `EADDRNOTAVAIL` or `ECONNREFUSED`.
>
> **Solutions (in order of preference):**
> 1. **Connection pooling** — the correct fix. Reuse long-lived TCP connections instead of creating new ones per request. HikariCP (Java), pgBouncer (Postgres), redis-py all do this. Eliminates the problem entirely.
> 2. **Increase ephemeral port range** — `sysctl net.ipv4.ip_local_port_range = 1024 65535` (Linux)
> 3. **`tcp_tw_reuse = 1`** — allows reuse of TIME_WAIT sockets for outbound connections when safe
> 4. **Reduce `tcp_fin_timeout`** — shortens TIME_WAIT duration (use cautiously in production)
>
> Core lesson: **always use connection pooling** for DB and any downstream service — it solves TIME_WAIT exhaustion, reduces handshake overhead, and dramatically improves throughput.