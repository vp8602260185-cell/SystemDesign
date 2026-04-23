# DNS (Domain Name System) in System Design

DNS is a distributed, hierarchical system that acts as the "phonebook" of the internet, translating human-readable domain names (e.g., `www.example.com`) into machine-friendly IP addresses.

## 1. The DNS Hierarchy
DNS is structured like a tree to ensure scalability and distributed management.
- **Root Level**: 13 logical root servers worldwide. They direct queries to the correct Top-Level Domain (TLD) servers.
- **Top-Level Domains (TLD)**: Managed by registries (e.g., `.com`, `.org`, `.net`).
- **Second-Level Domains**: The specific domain name registered (e.g., `example` in `example.com`).
- **Subdomains**: Optional organizational layers (e.g., `api.example.com`).

## 2. DNS Components & Roles
- **Recursive Resolver (DNS Resolver)**: Usually provided by an ISP. It handles the full resolution process on behalf of the client.
- **Root Name Servers**: The first stop for the resolver; points to TLD servers.
- **TLD Name Servers**: Points to the Authoritative Name Server for the specific domain.
- **Authoritative Name Servers**: The final authority that holds the actual DNS records (A, AAAA, MX, etc.) for a domain.

## 3. The Resolution Process
When you enter a URL, the following sequence occurs:
1. **Client Request**: Browser checks its cache, then queries the OS, then the Resolver.
2. **Recursive Search**: The Resolver queries the **Root Server**.
3. **TLD Referral**: Root directs Resolver to the **TLD Server** (e.g., `.com`).
4. **Authoritative Referral**: TLD directs Resolver to the **Authoritative Server**.
5. **Answer**: The Authoritative Server provides the IP, which the Resolver returns to the client.

## 4. Recursive vs. Iterative Queries
- **Recursive Query**: The client asks the server to do all the work and return the final answer. (e.g., Client -> Resolver).
- **Iterative Query**: The server provides a partial answer or a referral to the next server. (e.g., Resolver -> Root/TLD).

## 5. DNS Caching
Caching occurs at multiple levels to reduce latency and server load:
- **Browser Cache**: Temporary storage in the web browser.
- **OS Cache**: Local storage on the user's machine.
- **Resolver Cache**: The ISP's resolver saves previous lookups.
- **TTL (Time-to-Live)**: A value in the DNS record that dictates how long it should be cached before a fresh lookup is required.

## 6. Key Takeaways for System Design
- **High Availability**: DNS is globally distributed to avoid a Single Point of Failure.
- **Performance**: Heavy use of UDP (Port 53) for speed and caching for reduced latency.
- **Scalability**: The hierarchical nature allows billions of records to be managed without central bottlenecks.

---