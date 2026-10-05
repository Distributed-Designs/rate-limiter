# Distributed Rate Limiter — High-Level Design

## 1. Overview

The Distributed Rate Limiter is a Go-based service/component responsible for controlling the rate at which clients can access an application.

The system initially implements the **Token Bucket** algorithm and evolves from a local in-memory implementation into a distributed implementation backed by Redis.

The primary objective is to provide:

- Controlled request rates
- Burst tolerance
- Thread-safe operation
- Consistent limits across multiple application instances
- Low request-path latency
- Graceful handling of storage failures
- Observable rate-limit decisions

---

## 2. Problem Statement

In a distributed application, multiple instances of an application may receive requests from the same client.

For example:

```text
                    Load Balancer
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Server 1       Server 2       Server 3
          |              |              |
          +--------------+--------------+
                         |
                       Redis
```

If rate-limit state is maintained only in application memory, each server maintains an independent view of the client's usage.

For a limit of:

```text
100 requests / minute
```

a client could potentially make approximately:

```text
100 requests → Server 1
100 requests → Server 2
100 requests → Server 3
```

resulting in approximately 300 requests instead of the intended 100.

Therefore, a distributed rate limiter requires shared state.

---

# 3. Goals

## 3.1 Functional Goals

The system must:

1. Determine whether a request should be allowed.
2. Identify clients using a configurable key.
3. Support configurable rate limits.
4. Support burst traffic within the configured bucket capacity.
5. Return remaining capacity information.
6. Return retry information when requests are rejected.
7. Integrate with HTTP middleware.
8. Support multiple application instances.
9. Maintain shared rate-limit state through Redis.
10. Handle concurrent requests safely.

---

## 3.2 Non-Functional Goals

### Low latency

Rate limiting executes on the critical request path.

The additional latency introduced by the limiter should therefore be minimized.

### Thread safety

Multiple goroutines may access the same limiter simultaneously.

All state transitions must be safe under concurrent access.

### Horizontal scalability

The design should support:

```text
Application Instance 1
Application Instance 2
Application Instance 3
...
Application Instance N
```

while maintaining a shared rate-limit state.

### Fault tolerance

The system must define behavior when Redis becomes unavailable or slow.

### Testability

The limiter should be testable independently of HTTP and Redis.

---

# 4. Non-Goals

The initial version will not attempt to provide:

- Global traffic shaping across geographic regions
- Distributed consensus
- Persistent historical analytics
- Complex billing or quota management
- Adaptive rate limiting
- Machine-learning-based limits
- Multi-region strongly consistent rate limiting

These may be considered future extensions.

---

# 5. Rate-Limiting Model

The primary algorithm is **Token Bucket**.

Each client has a logical bucket:

```text
Capacity = maximum number of tokens

Refill Rate = tokens added per unit of time
```

Each accepted request consumes one token.

For example:

```text
Capacity:    10 tokens
Refill rate: 2 tokens / second
```

Initial state:

```text
+-----------------------+
| ● ● ● ● ● ● ● ● ● ● |
+-----------------------+
          10
```

After three requests:

```text
+-----------------+
| ● ● ● ● ● ● ● |
+-----------------+
        7
```

When no token is available:

```text
Request
   |
   v
No token
   |
   v
HTTP 429
```

Tokens are continuously replenished according to the configured refill rate.

The bucket cannot exceed its configured capacity.

---

# 6. Why Token Bucket?

Several rate-limiting algorithms are possible.

| Algorithm | Advantage | Limitation |
|---|---|---|
| Fixed Window | Simple and inexpensive | Boundary bursts |
| Sliding Window | More accurate traffic control | More state |
| Leaky Bucket | Smooth output rate | Less burst flexibility |
| Token Bucket | Burst tolerance + sustained rate | Requires token state |

Token Bucket is selected because it provides a useful balance between:

- Accuracy
- Performance
- Implementation complexity
- Burst handling
- Memory efficiency

---

# 7. High-Level Architecture

The system consists of the following logical components:

```text
                         Client
                           |
                           v
                    +--------------+
                    | HTTP Server  |
                    +------+-------+
                           |
                           v
                 +-------------------+
                 | Rate Limit        |
                 | Middleware        |
                 +---------+---------+
                           |
                           v
                 +-------------------+
                 | Rate Limiter      |
                 |                   |
                 | Token Bucket      |
                 +---------+---------+
                           |
                           v
                 +-------------------+
                 | Storage Abstraction|
                 +---------+---------+
                           |
                 +---------+---------+
                 |                   |
                 v                   v
             In-Memory             Redis
              Storage             Storage
```

The storage layer is abstracted from the rate-limiting algorithm.

This allows the same rate-limiting logic to operate with different storage implementations.

---

# 8. Distributed Architecture

In production, multiple application instances may run concurrently.

```text
                         Client
                           |
                           v
                    +--------------+
                    | Load Balancer|
                    +------+-------+
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
        +---------+   +---------+   +---------+
        | Server 1|   | Server 2|   | Server 3|
        +----+----+   +----+----+   +----+----+
             |             |             |
             +-------------+-------------+
                           |
                           v
                     +-----------+
                     |   Redis   |
                     +-----------+
```

All application instances use the same Redis-backed rate-limit state.

Therefore:

```text
Client A
   |
   +----> Server 1 ----+
   |                   |
   +----> Server 2 ----+----> Shared Redis state
   |                   |
   +----> Server 3 ----+
```

The client receives a globally coordinated rate-limit decision rather than an independent decision from each server.

---

# 9. Request Flow

A request follows this path:

```text
Client
  |
  v
HTTP Server
  |
  v
Rate Limit Middleware
  |
  |-- Extract client key
  |
  v
Rate Limiter
  |
  |-- Determine bucket state
  |
  |-- Refill tokens
  |
  |-- Attempt token consumption
  |
  v
Decision
  |
  +---- Allowed ----> Application Handler
  |
  +---- Rejected ---> HTTP 429
```

For a distributed deployment:

```text
Rate Limiter
      |
      v
Redis
      |
      +-- Read/update bucket state
      |
      v
Decision
```

The read/update operation must eventually be performed atomically.

---

# 10. Client Identification

The rate limiter requires a key identifying the entity being limited.

The initial implementation will support an API-key-style identifier:

```text
client_id = "client_123"
```

The architecture can later support:

```text
API Key
User ID
Tenant ID
IP Address
Endpoint + Client
```

For example:

```text
tenant_123:/api/orders
```

could represent a tenant-specific endpoint limit.

---

# 11. Rate-Limit Policy

A policy consists conceptually of:

```text
capacity
refill rate
```

Example:

```text
capacity   = 100
refillRate = 10 tokens / second
```

This allows short bursts of up to 100 requests while maintaining a long-term average rate of approximately 10 requests per second.

Different clients can have different policies:

```text
Free:
    capacity = 20
    refill   = 1/sec

Pro:
    capacity = 100
    refill   = 10/sec

Enterprise:
    capacity = 1000
    refill   = 100/sec
```

Policy management itself is outside the initial scope of the rate limiter.

---

# 12. HTTP Behavior

Allowed request:

```http
HTTP/1.1 200 OK

X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
```

Rejected request:

```http
HTTP/1.1 429 Too Many Requests

Retry-After: 3
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

The exact HTTP response format will be finalized during the LLD phase.

---

# 13. Redis as Distributed State

Redis will store the state required to make distributed rate-limit decisions.

Conceptually:

```text
rate_limit:{client_id}
```

may contain information such as:

```text
tokens
last_refill_time
```

For example:

```text
rate_limit:client_123

tokens = 73
last_refill = <timestamp>
```

The exact Redis data representation will be defined in the LLD.

---

# 14. Atomicity

A major requirement of the distributed implementation is atomic state modification.

A naive implementation could perform:

```text
GET bucket
     |
calculate new state
     |
SET bucket
```

This is unsafe under concurrent requests.

For example:

```text
Request A              Request B
    |                      |
    |---- GET: 5 --------->|
    |                      |
    |                 GET: 5
    |                      |
calculate 4          calculate 4
    |                      |
    |---- SET: 4 ----------|
                           |
                      SET: 4
```

Two requests consumed tokens, but the final state indicates only one token was consumed.

The distributed implementation will therefore require an atomic Redis-side operation.

A Redis Lua script is the planned mechanism for performing the bucket calculation and update atomically.

---

# 15. Failure Handling

Redis is part of the request path in the distributed implementation.

Therefore Redis failures must be explicitly handled.

Potential states include:

```text
Application
    |
    v
Redis
    |
    +---- Healthy
    |
    +---- Timeout
    |
    +---- Connection failure
    |
    +---- Unavailable
```

The system will support an explicit failure policy.

### Fail Open

```text
Redis unavailable
       |
       v
Allow request
```

Advantages:

- Application remains available.
- Rate limiter does not become a single point of failure.

Disadvantage:

- Rate limits may be bypassed during the outage.

### Fail Closed

```text
Redis unavailable
       |
       v
Reject request
```

Advantages:

- Protects downstream resources.

Disadvantages:

- Redis outage can become an application outage.

The final policy will be configurable and documented as part of the LLD/design decisions.

---

# 16. Scalability

The application layer is horizontally scalable:

```text
             Load Balancer
                  |
       +----------+----------+
       |          |          |
       v          v          v
    Server 1   Server 2   Server 3
       |          |          |
       +----------+----------+
                  |
                  v
               Redis
```

Redis itself may eventually require:

- Replication
- Failover
- Clustering
- Sharding

These are infrastructure concerns and are outside the first implementation.

---

# 17. Key Performance Considerations

The rate limiter is on the critical path.

Important performance factors include:

### Network latency

Distributed rate limiting introduces:

```text
Application → Redis → Application
```

latency.

### Contention

Popular clients may generate many concurrent requests against the same rate-limit key.

### Serialization

Redis operations must remain lightweight.

### Key cardinality

A system with millions of clients may create millions of rate-limit keys.

### Expiration

Inactive rate-limit state should eventually expire so Redis does not grow indefinitely.

---

# 18. Observability

The system should expose enough information to understand its behavior.

Potential metrics include:

```text
rate_limit_requests_total
rate_limit_allowed_total
rate_limit_rejected_total
rate_limit_redis_errors_total
rate_limit_redis_latency
rate_limit_decision_latency
```

Logs should contain relevant information such as:

```text
client key
decision
remaining capacity
failure reason
```

Sensitive identifiers should not be logged unnecessarily.

---

# 19. Testing Strategy

Testing will occur at multiple levels.

### Unit tests

Test:

```text
Token refill
Token consumption
Bucket capacity
Request rejection
Retry calculation
Configuration
```

### Concurrency tests

Test simultaneous requests using:

```bash
go test -race ./...
```

### Integration tests

Test:

```text
HTTP middleware
Redis
Distributed state
```

### Failure tests

Simulate:

```text
Redis timeout
Redis unavailable
Malformed configuration
Concurrent access
```

### Benchmarks

Measure:

```text
Requests/sec
Latency
Memory usage
Local vs Redis performance
```

---

# 20. Evolution of the System

The implementation will be developed in stages.

## Phase 1 — Local limiter

```text
HTTP
 |
Limiter
 |
Memory
```

Purpose:

- Validate Token Bucket algorithm.
- Establish interfaces.
- Test concurrency.

---

## Phase 2 — HTTP middleware

```text
HTTP Request
     |
Middleware
     |
Limiter
     |
Handler
```

Purpose:

- Integrate the limiter into an actual request path.
- Produce HTTP 429 responses.

---

## Phase 3 — Redis storage

```text
HTTP
 |
Limiter
 |
Redis
```

Purpose:

- Introduce shared state.
- Support multiple application instances.

---

## Phase 4 — Atomic distributed operations

```text
Limiter
   |
   v
Redis Lua Script
   |
   +-- Read state
   +-- Refill
   +-- Consume
   +-- Update
   +-- Return result
```

Purpose:

- Prevent race conditions between distributed clients.

---

## Phase 5 — Failure handling

Add:

```text
timeouts
connection handling
fail-open/fail-closed policy
```

---

## Phase 6 — Performance and observability

Add:

```text
benchmarks
metrics
structured logging
load testing
```

---

# 21. Future Extensions

Possible future improvements include:

- Per-user limits
- Per-tenant limits
- Per-endpoint limits
- Hierarchical quotas
- Multiple rate-limit policies
- Redis Cluster
- Multi-region rate limiting
- Dynamic configuration
- Administrative APIs
- Prometheus metrics
- OpenTelemetry tracing
- Distributed rate-limit configuration service

These are intentionally outside the initial implementation.

---

# 22. Final High-Level Architecture

The target architecture is:

```text
                           Client
                              |
                              v
                     +----------------+
                     | Load Balancer  |
                     +-------+--------+
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
        +-----------+  +-----------+  +-----------+
        | App       |  | App       |  | App       |
        | Instance 1|  | Instance 2|  | Instance 3|
        +-----+-----+  +-----+-----+  +-----+-----+
              |              |              |
              +--------------+--------------+
                             |
                             v
                  +----------------------+
                  | Rate Limit Middleware|
                  +----------+-----------+
                             |
                             v
                  +----------------------+
                  |    Rate Limiter      |
                  |    Token Bucket       |
                  +----------+-----------+
                             |
                             v
                  +----------------------+
                  | Storage Abstraction  |
                  +----------+-----------+
                             |
                             v
                  +----------------------+
                  |        Redis         |
                  | Shared Bucket State  |
                  +----------------------+
```

The key architectural principle is:

> **The rate-limiting algorithm is independent of the storage mechanism, while distributed deployments use shared Redis state to coordinate decisions across application instances.**