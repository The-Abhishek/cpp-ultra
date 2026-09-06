# 09. API Gateway

An API Gateway is a server that acts as a single entry point into a system of microservices. It sits between the clients and the backend services.

## Core Responsibilities
Instead of clients communicating directly with 50 different microservices, the API Gateway handles:
1. **Routing:** Directing `/users` to the User Service, and `/orders` to the Order Service.
2. **Authentication/Authorization:** Validating JWTs before letting requests into the internal network.
3. **Rate Limiting & Throttling:** Preventing DDoS attacks and limiting API usage per user.
4. **Request/Response Transformation:** Converting protocols (e.g., HTTP to gRPC) or filtering sensitive fields.
5. **Load Balancing:** Distributing requests across multiple instances of a service.

## Direct Client-to-Microservice vs API Gateway

- **Direct Communication:** Client must know the IP/URL of every service. Difficult to refactor backends. High chatiness (client makes multiple calls).
- **API Gateway:** Hides internal architecture. Client makes one call to the gateway, gateway orchestrates calls to multiple backends and aggregates the result.

## BFF (Backend For Frontend) Pattern
Instead of one massive API Gateway, we create smaller, specialized gateways for different client types.
- **Mobile BFF:** Optimizes payload size, aggregates data to reduce round trips over slow cellular networks.
- **Web BFF:** Might return richer data, HTML snippets, or handle browser-specific cookies.

## API Gateway vs Service Mesh
- **API Gateway:** Handles **North-South** traffic (External Client -> Internal Services). Focuses on business logic, auth, and external exposure.
- **Service Mesh (e.g., Istio):** Handles **East-West** traffic (Internal Service -> Internal Service). Focuses on internal networking, mTLS, and observability.

## Popular Tools
- **Kong:** Fast, Lua/Nginx based.
- **AWS API Gateway:** Managed, integrates easily with Lambda.
- **Envoy/Nginx:** Core proxies often used to build gateways.

> **Interview Tip:** The API Gateway is a Single Point of Failure (SPOF) and a potential bottleneck. Ensure you mention that it needs to be highly available (HA), scaled horizontally behind a Load Balancer, and kept as stateless as possible.
