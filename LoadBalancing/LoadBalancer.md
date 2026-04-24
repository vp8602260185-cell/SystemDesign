# Load Balancers & Algorithms in System Design

Load balancing is a core infrastructure pattern that ensures high availability and scalability by distributing incoming traffic across a pool of backend servers.

## 1. What are Load Balancers?
A Load Balancer (LB) acts as a "traffic cop," sitting in front of your servers and routing client requests to ensure no single server becomes a bottleneck.

### Why Do We Need Them?
- **Scalability**: Allows you to handle more traffic by adding more servers (Horizontal Scaling).
- **High Availability**: Automatically detects server failures and reroutes traffic to healthy nodes.
- **Performance**: Prevents server overload, leading to lower latency for users.
- **Security**: Provides a single point of entry to hide internal IP addresses and protect against DDoS attacks.

### Key Features
- **SSL Termination**: Decrypts HTTPS traffic at the LB level to offload CPU-intensive work from backend servers.
- **Health Checks**: Periodically "pings" servers to ensure they are alive before sending traffic.
- **Sticky Sessions**: Ensures a specific user's requests always go to the same server (useful for stateful apps).

## 2. How Load Balancing Works
The process typically follows these four steps:
1. **Traffic Reception**: The LB receives a request at its public IP.
2. **Decision Logic**: The LB selects a server based on a pre-defined **Load Balancing Algorithm**.
3. **Health Check Validation**: The LB verifies the chosen server is healthy.
4. **Response Handling**: The server processes the request and sends the response back through the LB to the client.

## 3. Load Balancing Algorithms
Algorithms are categorized into **Static** (fixed logic) and **Dynamic** (state-aware logic).

### Static Algorithms
| Algorithm | How it Works | Best For... |
| :--- | :--- | :--- |
| **Round Robin** | Requests are sent sequentially to each server in a loop. | Servers with identical specs. |
| **Weighted Round Robin** | Servers are assigned "weights" based on capacity; higher weights get more traffic. | Heterogeneous server clusters. |
| **IP Hash** | Uses a hash of the client's IP address to determine the server. | Maintaining "Sticky Sessions." |

### Dynamic Algorithms
| Algorithm | How it Works | Best For... |
| :--- | :--- | :--- |
| **Least Connections** | Routes traffic to the server with the fewest active connections. | Long-lived connections (e.g., WebSockets). |
| **Least Response Time** | Measures the latency of each server and picks the fastest one. | Optimizing user experience. |
| **Least Bandwidth** | Routes to the server currently serving the least amount of data (Mbps). | Bandwidth-heavy applications. |

## 4. Layer 4 vs. Layer 7 Load Balancing
- **Layer 4 (Transport Layer)**: Routes based on network data (IP and Port). It is faster but "blind" to the actual content of the request.
- **Layer 7 (Application Layer)**: Routes based on application data (HTTP headers, cookies, URL paths). It is more intelligent and can perform content-based routing.

---