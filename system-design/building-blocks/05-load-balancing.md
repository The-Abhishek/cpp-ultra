# 05. Load Balancing

Load balancers (LB) distribute incoming network traffic across a group of backend servers to ensure high availability and reliability.

## L4 vs L7 Load Balancing

### Layer 4 (Transport Layer)
- **How it works:** Routes traffic based on IP address and TCP/UDP ports.
- **Characteristics:** Fast, low CPU overhead. It does not inspect the message payload.
- **Example:** AWS Network Load Balancer (NLB).

### Layer 7 (Application Layer)
- **How it works:** Routes traffic based on HTTP/HTTPS headers, URLs, cookies, and payload data.
- **Characteristics:** Slower than L4 but much smarter. Can terminate SSL/TLS.
- **Example:** Nginx, AWS Application Load Balancer (ALB).

## Routing Algorithms
1. **Round Robin:** Distributes requests sequentially across servers.
2. **Weighted Round Robin:** Assigns more requests to servers with higher capacity (weights).
3. **Least Connections:** Sends the request to the server with the fewest active connections.
4. **IP Hash:** Computes a hash of the client's IP address to ensure a client always reaches the same server.
5. **Consistent Hashing:** Used in caching clusters to minimize data movement on topology changes.

## Health Checks
LBs must ensure they only route traffic to healthy servers.
- **Active (Ping):** The LB periodically sends a request (e.g., `GET /health`) to servers.
- **Passive:** The LB monitors traffic. If a server returns multiple 5xx errors, it's marked unhealthy.

## Global vs Local Load Balancing
- **Global Server Load Balancing (GSLB):** DNS-based routing. Directs users to the closest geographical data center.
- **Local Load Balancing:** Distributes traffic within a single data center across a cluster of application servers.

## Sticky Sessions (Session Affinity)
- **What is it?** Ensures a user's requests are consistently routed to the same backend server.
- **Why use it?** If user session data (like a shopping cart) is stored in the memory of a specific server rather than a distributed cache.
- **Downside:** Can lead to uneven load distribution and fails if that specific server crashes.

> **Interview Tip:** Always prefer stateless servers with a centralized cache (like Redis) over using Sticky Sessions.
