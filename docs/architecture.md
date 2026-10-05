# Distributed Rate Limiter — Architecture

## 1. Architecture Overview

The rate limiter is designed as a reusable Go component that can operate in two modes:

```text
Local Mode
    Application
        |
        v
    Rate Limiter
        |
        v
    In-Memory State
```

and:

```text
Distributed Mode
    Application
        |
        v
    Rate Limiter
        |
        v
    Redis
```

The distributed mode allows multiple application instances to share the same rate-limit state.

---

# 2. Production Deployment

The intended deployment topology is:

```text
                              Internet
                                  |
                                  v
                         +----------------+
                         | Load Balancer  |
                         +-------+--------+
                                 |
                 +---------------+---------------+
                 |               |               |
                 v               v               v
          +-------------+ +-------------+ +-------------+
          | App Server 1| | App Server 2| | App Server 3|
          +------+------+ +------+------+ +------+------+
                 |               |               |
                 +---------------+---------------+
                                 |
                                 v
                       +---------------------+
                       | Distributed         |
                       | Rate Limiter        |
                       +----------+----------+
                                  |
                                  v
                         +------------------+
                         |      Redis       |
                         |                  |
                         | Shared State     |
                         +------------------+
```

Each application instance contains the rate-limiter logic.

Redis contains the shared bucket state.

The rate limiter therefore does not need to be deployed as a separate network service in the initial architecture.

---

# 3. Component Responsibilities

## 3.1 HTTP Server

Responsible for:

- Accepting HTTP requests.
- Routing requests.
- Applying middleware.
- Returning HTTP responses.

It does not implement rate-limiting logic.

---

## 3.2 Rate-Limit Middleware

Responsible for:

- Extracting the client identity.
- Calling the rate limiter.
- Converting the decision into an HTTP response.
- Returning HTTP 429 when appropriate.
- Setting rate-limit headers.

Flow:

```text
HTTP Request
     |
     v
Middleware
     |
     +---- Extract client key
     |
     v
RateLimiter.Allow()
     |
     +---- Allowed ----> Handler
     |
     +---- Rejected ---> 429
```

---

## 3.3 Rate Limiter

Responsible for:

- Applying the Token Bucket algorithm.
- Calculating token refill.
- Consuming tokens.
- Calculating remaining capacity.
- Calculating retry time.
- Returning a rate-limit decision.

It should not know about HTTP.

---

## 3.4 Storage

Responsible for:

- Persisting bucket state.
- Retrieving bucket state.
- Updating bucket state.
- Managing state expiration.

Two implementations are planned:

```text
Storage
   |
   +---- Memory
   |
   +---- Redis
```

---

## 3.5 Redis

Redis provides:

- Shared state.
- Atomic state transitions.
- Fast access.
- Key expiration.

The Redis implementation uses a Lua script to perform the complete bucket operation atomically.

---

# 4. Request Lifecycle

A normal request follows:

```text
Client
  |
  | HTTP request
  v
Load Balancer
  |
  v
Application Instance
  |
  v
Rate-Limit Middleware
  |
  | Extract client key
  v
Rate Limiter
  |
  | Evaluate bucket
  v
Redis
  |
  | Atomic Lua operation
  v
Rate Limiter
  |
  v
Middleware
  |
  +-------- Allowed --------> Application Handler
  |
  +-------- Rejected -------> HTTP 429
```

---

# 5. Allowed Request Sequence

Example:

```text
Capacity = 100
Current tokens = 73
Request cost = 1
```

Sequence:

```text
Client
  |
  v
Middleware
  |
  v
Limiter
  |
  v
Redis Lua Script
  |
  +-- Read bucket
  |
  +-- Calculate elapsed time
  |
  +-- Refill tokens
  |
  +-- Check tokens >= 1
  |
  +-- Consume token
  |
  +-- Store new state
  |
  +-- Return allowed=true
  |
  v
Middleware
  |
  v
Application Handler
```

Result:

```text
Allowed = true
Remaining = 72
```

---

# 6. Rejected Request Sequence

When the bucket has insufficient tokens:

```text
Client
  |
  v
Middleware
  |
  v
Limiter
  |
  v
Redis Lua Script
  |
  +-- Read bucket
  |
  +-- Refill
  |
  +-- Check tokens
  |
  +-- tokens < 1
  |
  +-- Do not consume
  |
  +-- Calculate retry time
  |
  v
Middleware
  |
  v
HTTP 429
```

The application handler is never executed.

---

# 7. Token Bucket State

The logical bucket state is:

```text
+-------------------------+
| Bucket                  |
+-------------------------+
| tokens                  |
| last_refill_time        |
+-------------------------+
```

For example:

```text
rate_limit:client_123

tokens = 72.5
last_refill_time = T
```

The configured policy is kept separately:

```text
capacity = 100
refill_rate = 10 tokens/sec
```

---

# 8. Token Refill Flow

The refill calculation is:

```text
elapsed = current_time - last_refill_time

refill = elapsed × refill_rate

tokens = min(
    capacity,
    tokens + refill
)
```

Then:

```text
last_refill_time = current_time
```

Example:

```text
Capacity:       10
Current tokens: 2
Refill rate:    4/sec
Elapsed:        1.5 sec
```

Calculation:

```text
refill = 1.5 × 4
       = 6

tokens = 2 + 6
       = 8
```

---

# 9. Redis Atomic Operation

The critical distributed operation is:

```text
             Redis
               |
               v
        +--------------+
        | Lua Script   |
        +------+-------+
               |
       +-------+-------+
       |               |
       v               v
 Read bucket       Read policy
       |               |
       +-------+-------+
               |
               v
        Calculate refill
               |
               v
        Check token count
          /           \
         /             \
        v               v
    Available        Unavailable
        |               |
        v               v
   Consume token    Calculate retry
        |               |
        +-------+-------+
                |
                v
          Save state
                |
                v
          Return result
```

The entire operation executes atomically within Redis.

---

# 10. Why Lua Is Required

A naive distributed implementation could perform:

```text
GET
 |
Calculate
 |
SET
```

This creates a race condition.

Example:

```text
Initial tokens = 1

Request A              Request B
    |                      |
    |------ GET: 1 ------->|
    |                      |
    |                  GET: 1
    |                      |
consume token          consume token
    |                      |
    |------ SET: 0 --------|
                           |
                      SET: 0
```

Both requests can be accepted even though only one token existed.

The Lua script makes:

```text
read → calculate → consume → write
```

one atomic operation.

---

# 11. Concurrent Requests

Suppose:

```text
Bucket = 10 tokens
```

and three application instances simultaneously receive requests.

```text
Server 1 ─────┐
Server 2 ─────┼────> Redis
Server 3 ─────┘
```

Each executes the same atomic operation.

Redis serializes execution of the Lua scripts.

Conceptually:

```text
Script A
   ↓
state = 10
   ↓
consume
   ↓
state = 9

Script B
   ↓
state = 9
   ↓
consume
   ↓
state = 8

Script C
   ↓
state = 8
   ↓
consume
   ↓
state = 7
```

The shared state remains consistent.

---

# 12. Client Key Model

The client key is the logical identifier used to isolate rate-limit state.

Example:

```text
client_123
```

maps to:

```text
rate_limit:client_123
```

Different clients have independent buckets:

```text
rate_limit:client_123
rate_limit:client_456
rate_limit:client_789
```

No client can consume another client's tokens.

---

# 13. Endpoint-Specific Limits

The key can later be composed from multiple attributes.

For example:

```text
client_123:/api/orders
```

or:

```text
tenant_123:/api/orders
```

This enables:

```text
Per-client limits
Per-tenant limits
Per-endpoint limits
Per-client-per-endpoint limits
```

without changing the underlying Token Bucket algorithm.

---

# 14. Redis Key Expiration

Each bucket should have an expiration time.

```text
Request
   |
   v
Create/update bucket
   |
   v
Set TTL
   |
   v
Client becomes inactive
   |
   v
TTL expires
   |
   v
Redis removes bucket
```

This prevents stale client keys from accumulating indefinitely.

---

# 15. Failure Architecture

Redis is a dependency in the distributed path.

Possible failure:

```text
Application
     |
     v
   Redis
     X
  Failure
```

The limiter must distinguish between:

```text
Rate limit rejection
```

and:

```text
Rate limiter infrastructure failure
```

These are fundamentally different conditions.

---

# 16. Fail-Open Flow

If configured for fail-open:

```text
Request
   |
   v
Rate Limiter
   |
   v
Redis
   X
Failure
   |
   v
Allow request
   |
   v
Application Handler
```

This prioritizes application availability.

---

# 17. Fail-Closed Flow

If configured for fail-closed:

```text
Request
   |
   v
Rate Limiter
   |
   v
Redis
   X
Failure
   |
   v
Reject request
```

This prioritizes protection of downstream resources.

The failure mode is an explicit configuration option.

---

# 18. Timeout Flow

Redis operations must have bounded timeouts.

```text
Request
   |
   v
Limiter
   |
   v
Redis
   |
   |--- timeout
   |
   v
Failure Policy
   |
   +---- Fail Open
   |
   +---- Fail Closed
```

The limiter must not allow a slow Redis operation to indefinitely block HTTP requests.

---

# 19. Context Propagation

The request context flows through the entire stack:

```text
HTTP Request
     |
     v
context.Context
     |
     v
Middleware
     |
     v
RateLimiter
     |
     v
Redis
```

If the client disconnects or the request deadline expires, downstream operations should be cancelled where supported.

---

# 20. Application Dependency Graph

The dependency direction is:

```text
                    cmd/server
                        |
                        v
                   middleware
                        |
                        v
                    limiter
                        |
                        v
                    storage
                     /     \
                    /       \
                   v         v
               Memory      Redis
```

Dependencies should flow inward toward the core logic.

The Token Bucket implementation should not import:

```text
net/http
```

or directly depend on Redis.

---

# 21. Local Development Architecture

For local development:

```text
+---------------------+
| Go Application      |
|                     |
| HTTP                |
| Middleware          |
| Rate Limiter        |
| Memory Storage      |
+---------------------+
```

No Redis dependency is required for basic algorithm development.

---

# 22. Distributed Development Architecture

For integration testing:

```text
+---------------------+
| Go Application      |
+----------+----------+
           |
           v
+---------------------+
| Redis               |
| localhost:6379      |
+---------------------+
```

Docker Compose will be used to provide Redis consistently.

---

# 23. Docker Deployment

The development environment will contain:

```text
docker/
└── docker-compose.yml
```

Conceptually:

```text
+--------------------+       +----------------+
| Rate Limiter App   | ----> | Redis          |
|                    |       |                |
| Go                 |       | Shared State   |
+--------------------+       +----------------+
```

The application itself can later be packaged using:

```text
Dockerfile
```

---

# 24. Observability Architecture

The rate limiter should eventually expose metrics:

```text
                Rate Limiter
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
    Allowed       Rejected      Errors
       |             |             |
       +-------------+-------------+
                     |
                     v
                  Metrics
```

Potential metrics:

```text
rate_limit_requests_total
rate_limit_allowed_total
rate_limit_rejected_total
rate_limit_errors_total
rate_limit_decision_duration
rate_limit_redis_duration
```

These metrics help distinguish:

```text
High traffic
```

from:

```text
High rejection rate
```

and:

```text
Infrastructure problems
```

---

# 25. Performance Architecture

The request path should remain minimal:

```text
HTTP
  ↓
Middleware
  ↓
Limiter
  ↓
Redis
  ↓
Limiter
  ↓
HTTP
```

There should be no unnecessary database queries, network calls, or application-level serialization.

The Redis Lua script combines multiple operations into one round trip.

This reduces:

```text
Network round trips
+
Race windows
+
Application-side coordination
```

---

# 26. Scaling Characteristics

### Application layer

Stateless and horizontally scalable:

```text
N application instances
```

### Redis layer

Initially:

```text
Single Redis instance
```

Future production scaling may use:

```text
Redis Sentinel
Redis Cluster
Managed Redis
```

depending on availability and scale requirements.

---

# 27. Hot-Key Consideration

A particularly active client may produce a hot key:

```text
rate_limit:popular_client
```

A large number of requests will target the same Redis key.

This can create contention.

Potential future strategies include:

- Client-level traffic shaping.
- Multiple buckets.
- Hierarchical rate limits.
- Redis Cluster-aware key design.
- Local pre-limiting before Redis.
- Dedicated infrastructure for extremely high-volume clients.

These optimizations are not part of the first implementation.

---

# 28. Failure Boundaries

The system has the following major failure boundaries:

```text
Client
  |
  X
Load Balancer
  |
  X
Application
  |
  X
Rate Limiter
  |
  X
Redis
```

Each boundary should be observable and have defined behavior.

The most important dependency is Redis because its failure directly affects distributed rate-limit decisions.

---

# 29. End-to-End Architecture

The final architecture is:

```text
                              CLIENT
                                |
                                v
                         +-------------+
                         |Load Balancer|
                         +------+------+
                                |
             +------------------+------------------+
             |                  |                  |
             v                  v                  v
        +---------+        +---------+        +---------+
        | Server 1|        | Server 2|        | Server 3|
        +----+----+        +----+----+        +----+----+
             |                  |                  |
             v                  v                  v
        +------------------------------------------------+
        |              Rate Limit Middleware             |
        +------------------------+-----------------------+
                                 |
                                 v
                       +-------------------+
                       |   Rate Limiter    |
                       |                   |
                       |   Token Bucket    |
                       +---------+---------+
                                 |
                                 v
                       +-------------------+
                       | Storage Interface |
                       +---------+---------+
                                 |
                                 v
                       +-------------------+
                       | Redis Store       |
                       +---------+---------+
                                 |
                                 v
                       +-------------------+
                       | Redis Lua Script  |
                       |                   |
                       | Atomic bucket     |
                       | transition        |
                       +-------------------+
```

The central architectural invariant is:

> **Every distributed rate-limit decision for a given key must be based on a single atomic transition of the shared bucket state.**

This invariant prevents concurrent application instances from independently accepting requests based on stale bucket state.