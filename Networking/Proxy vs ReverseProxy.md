# Proxy vs. Reverse Proxy in System Design

Proxies and reverse proxies are intermediary servers that sit between clients and servers to enhance security, privacy, and performance.

## 1. Proxy Server (Forward Proxy)
A **Forward Proxy** acts on behalf of **clients**. When a client sends a request, it goes through the proxy, which then communicates with the internet.

- **Position**: Sits in front of the client (e.g., within a corporate network).
- **Core Goal**: Protect the client's identity (anonymity).
- **Key Features**:
    - **Anonymity**: The destination server sees the proxy's IP, not the client's.
    - **Content Filtering**: Organizations use proxies to block access to specific websites.
    - **Bypassing Restrictions**: Used to access geo-blocked content.
    - **Caching**: Stores copies of frequently accessed web pages to speed up requests.

## 2. Reverse Proxy
A **Reverse Proxy** acts on behalf of **servers**. It sits at the edge of the server network, intercepting client requests before they reach the backend.

- **Position**: Sits in front of one or more backend servers.
- **Core Goal**: Protect the server infrastructure and optimize performance.
- **Key Features**:
    - **Load Balancing**: Distributes incoming traffic across multiple backend servers to prevent overload.
    - **Enhanced Security**: Hides the internal IP addresses of backend servers, making them harder to attack.
    - **SSL Termination**: Handles the heavy lifting of SSL/TLS encryption/decryption, offloading the task from backend servers.
    - **Caching Static Content**: Stores images, CSS, and JS files to serve them faster without hitting the backend.
    - **Compression**: Compresses outgoing responses to save bandwidth.

## 3. Comparison Table

| Feature | Proxy (Forward) | Reverse Proxy |
| :--- | :--- | :--- |
| **Acting on behalf of** | Client | Server |
| **Visibility to Client** | Client knows they are using it. | Client thinks they are talking to the real server. |
| **Visibility to Server** | Server sees the proxy, not the client. | Server is hidden from the client. |
| **Primary Use Case** | Privacy, Filtering, Bypassing geo-blocks. | Load Balancing, Security, SSL Offloading. |
| **Common Examples** | VPNs, Corporate Web Proxies. | Nginx, HAProxy, Cloudflare. |

## 4. Why it matters in System Design
- **Scalability**: Reverse proxies are the backbone of horizontal scaling (Load Balancing).
- **Security**: They provide a single point of entry, making it easier to monitor traffic and block DDoS attacks.
- **Availability**: If a backend server fails, the reverse proxy can reroute traffic to healthy servers.

---