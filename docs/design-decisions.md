# Distributed Rate Limiter — Design Decisions

## 1. Purpose

This document records the major architectural and implementation decisions made for the distributed rate limiter.

Each decision includes:

- The problem being solved
- The selected approach
- Alternatives considered
- Trade-offs

The goal is to make the system's design intentional and explainable.

---

# 2. Token Bucket as the Primary Algorithm

## Decision

Use the **Token Bucket** algorithm.

```text
Capacity = maximum burst size
Refill Rate = sustained request rate
```

Each accepted request consumes one token.

---

## Why

Token Bucket provides:

- Controlled sustained traffic
- Burst tolerance
- Constant-time decision making
- Small state
- Straightforward implementation

Example:

```text
Capacity:   100
Refill:     10 tokens/sec
```

The client can make a burst of up to 100 requests when the bucket is full, while the long-term rate converges toward approximately 10 requests/sec.

---

## Alternatives

### Fixed Window

Example:

```text
100 requests
per 60-second window
```

Problem:

```text
59.9 sec → 100 requests
60.1 sec → 100 requests
```

Potentially 200 requests can occur within a very short period around the boundary.

**Decision:** Not selected because of boundary bursts.

---

### Sliding Window

Provides better temporal accuracy.

However, it generally requires more state and more computation than Token Bucket.

**Decision:** Not selected for the initial implementation.

---

### Leaky Bucket

Provides a smoother output rate.

However, strict smoothing is not required for this project, and burst tolerance is desirable.

**Decision:** Not selected as the primary algorithm.

---

# 3. Separate Algorithm from Storage

## Decision

The Token Bucket logic must not directly depend on Redis.

Architecture:

```text
RateLimiter
     |
     v
Storage abstraction
     |
     +---- Memory
     |
     +---- Redis
```

---

## Why

This provides:

- Unit-testability
- Easier local development
- Storage independence
- Cleaner separation of concerns
- Ability to benchmark local vs distributed implementations

It also prevents the core rate-limiting algorithm from becoming coupled to infrastructure.

---

## Trade-off

An abstraction adds some interfaces and indirection.

For a tiny application this could be unnecessary.

For this project, the abstraction is justified because distributed storage is a core requirement.

---

# 4. Redis for Distributed State

## Decision

Use Redis as the shared state store.

---

## Why Redis

Rate limiting requires:

- Low latency
- High request throughput
- Shared state
- Atomic operations
- Key expiration

Redis provides these characteristics and is well suited to frequently changing ephemeral state.

---

## Why not PostgreSQL?

PostgreSQL could store rate-limit state, but the workload is dominated by extremely frequent reads/writes to small pieces of ephemeral state.

A relational database would introduce unnecessary:

- Query overhead
- Transaction overhead
- Persistent storage requirements
- Database contention

Redis is a better fit for this workload.

---

## Why not application memory?

Memory is fast but cannot coordinate state across application instances.

For example:

```text
Server 1 → 100 requests
Server 2 → 100 requests
```

would result in separate local counters.

Therefore:

```text
Local memory ≠ global distributed state
```

---

# 5. Lua for Redis Atomicity

## Decision

Use a Redis Lua script to perform the bucket state transition atomically.

---

## Problem

A naive implementation might perform:

```text
GET
  ↓
Calculate
  ↓
SET
```

Two application instances can interleave these operations.

Example:

```text
Initial tokens = 1

Server A              Server B

GET → 1               GET → 1

consume               consume

SET → 0               SET → 0
```

Both requests may be accepted despite only one token being available.

---

## Selected Solution

Execute:

```text
Read
→ Refill
→ Check
→ Consume
→ Update
→ Expire
```

inside one Redis Lua script.

Redis executes the script atomically.

---

## Alternatives

### MULTI/EXEC

Redis transactions can group commands, but conditional logic and state-dependent calculations are less convenient than a Lua script.

### WATCH

Optimistic locking can work but introduces retry complexity under contention.

### Application-side locking

A mutex only protects one process.

It cannot coordinate:

```text
Server 1
Server 2
Server 3
```

Therefore it does not solve distributed concurrency.

---

# 6. In-Memory Mutex for Local Limiter

## Decision

Use synchronization around local bucket state.

The critical state transition is:

```text
Find bucket
→ Refill
→ Check
→ Consume
→ Update
```

This operation must be protected against concurrent goroutines.

---

## Why

Go applications commonly process multiple requests concurrently using goroutines.

Without synchronization:

```text
Goroutine A ─┐
Goroutine B ─┼──> same bucket
Goroutine C ─┘
```

could modify the same bucket simultaneously.

This creates race conditions and incorrect token counts.

---

## Alternative

`sync.Map` could be considered for concurrent map access.

However, the operation involves more than simply accessing the map.

We need to atomically perform:

```text
lookup + refill + consume + update
```

Therefore explicit synchronization around the logical state transition is simpler and easier to reason about initially.

---

# 7. Floating-Point Token State

## Decision

Represent token count as a floating-point value.

Example:

```text
tokens = 7.5
```

---

## Why

The refill rate may not be an integer.

For example:

```text
5 tokens / second
```

is straightforward, but policies may eventually require:

```text
2.5 tokens / second
```

Using fractional tokens allows precise elapsed-time calculations.

---

## Trade-off

Floating-point arithmetic introduces precision considerations.

The implementation must therefore ensure:

```text
tokens >= 0
tokens <= capacity
```

and avoid relying on exact floating-point equality.

---

# 8. Time-Based Refill Instead of a Background Goroutine

## Decision

Do not continuously refill every bucket with a background goroutine.

Instead, refill lazily when a request arrives.

---

## Why

A background refill mechanism would require work for every active bucket:

```text
Bucket 1 → goroutine/timer
Bucket 2 → goroutine/timer
Bucket 3 → goroutine/timer
...
Bucket N → goroutine/timer
```

This scales poorly with large numbers of clients.

Lazy refill requires only:

```text
current_time - last_refill_time
```

when the bucket is accessed.

---

## Result

No dedicated refill goroutine is necessary.

The bucket's logical state advances only when it is accessed.

---

# 9. TTL for Bucket State

## Decision

Expire inactive Redis buckets.

---

## Why

Client cardinality can become large.

Without expiration:

```text
Client A → key
Client B → key
Client C → key
...
Client N → key
```

could cause Redis memory usage to grow indefinitely.

TTL ensures inactive state eventually disappears.

---

## Trade-off

An expired client loses its previous bucket state.

This is acceptable because the bucket represents ephemeral rate-limit state rather than durable business data.

---

# 10. Client-Based Keys

## Decision

Use a client identifier as the initial rate-limit key.

Example:

```text
rate_limit:client_123
```

---

## Why

This provides a clear and predictable isolation boundary.

Different clients receive independent buckets:

```text
client_123 → Bucket A
client_456 → Bucket B
```

---

## Future Extension

Keys can become composite:

```text
client_123:/api/orders
tenant_123:/api/orders
user_123:/api/search
```

This allows per-endpoint and per-tenant policies without changing the core algorithm.

---

# 11. HTTP Middleware Instead of Embedding HTTP Logic in the Limiter

## Decision

Keep HTTP concerns in middleware.

Architecture:

```text
HTTP
 ↓
Middleware
 ↓
RateLimiter
```

The rate limiter itself does not know about:

```text
HTTP requests
HTTP responses
status codes
headers
```

---

## Why

This follows separation of concerns.

The limiter answers:

```text
"Should this request be allowed?"
```

The middleware answers:

```text
"How should that decision be represented in HTTP?"
```

This also allows the limiter to be reused outside HTTP.

---

# 12. Context Propagation

## Decision

Every rate-limit operation accepts:

```go
context.Context
```

---

## Why

Distributed operations may involve Redis network calls.

The caller should be able to cancel them when:

- The HTTP request is cancelled.
- The request deadline expires.
- The server is shutting down.

This prevents unnecessary work from continuing after the request is no longer relevant.

---

# 13. Fail-Open vs Fail-Closed

## Decision

Make the behavior configurable.

```text
Fail Open
Fail Closed
```

---

## Fail Open

```text
Redis failure
     ↓
Allow request
```

### Advantage

Preserves application availability.

### Disadvantage

Rate limits can be bypassed during the Redis outage.

---

## Fail Closed

```text
Redis failure
     ↓
Reject request
```

### Advantage

Protects downstream systems.

### Disadvantage

A Redis outage can effectively become an application outage.

---

## Why Configurable?

Different systems have different priorities.

### Public API

Availability may be more important:

```text
Fail Open
```

### Expensive or sensitive backend

Resource protection may be more important:

```text
Fail Closed
```

Therefore the rate limiter should not hard-code one universal policy.

---

# 14. Bounded Redis Timeouts

## Decision

Redis operations must have bounded execution time.

---

## Why

Without a timeout:

```text
HTTP Request
     |
     v
Redis
     |
     X
Network problem
     |
     v
Request waits indefinitely
```

This could exhaust application resources.

Instead:

```text
HTTP Request
     |
     v
Redis
     |
     X
Timeout
     |
     v
Failure Policy
```

The timeout should be shorter than the overall request deadline.

---

# 15. Stateless Application Instances

## Decision

Application instances should not maintain distributed rate-limit state locally.

```text
Server 1 ──┐
Server 2 ──┼──> Redis
Server 3 ──┘
```

---

## Why

This allows horizontal scaling.

Adding another application instance should not change the configured global limit.

```text
3 servers
   ↓
shared state

10 servers
   ↓
same shared state
```

---

# 16. Redis as a Single Logical State Store

## Decision

All application instances use the same logical Redis namespace for rate-limit state.

Example:

```text
rate_limit:{client_id}
```

---

## Why

A shared namespace guarantees that a client's requests are evaluated against the same bucket.

Future Redis Cluster deployments can preserve this model while distributing keys across nodes.

---

# 17. Avoid Unnecessary Network Round Trips

## Decision

The complete rate-limit decision should ideally require one Redis round trip.

Instead of:

```text
GET
 ↓
Application
 ↓
Calculate
 ↓
SET
```

use:

```text
EVAL
 ↓
Read
 ↓
Calculate
 ↓
Update
 ↓
Return
```

---

## Benefits

- Lower latency
- Fewer network operations
- Smaller race window
- Atomic state transition

This is one of the main reasons the Lua approach was selected.

---

# 18. Error Separation

## Decision

A rejected request is not treated as an infrastructure error.

There are two different outcomes:

### Normal rate-limit rejection

```text
Allowed = false
Error = nil
```

### Infrastructure failure

```text
Allowed = false
Error = ErrRedisUnavailable
```

These must remain distinct.

Otherwise the middleware cannot correctly apply the configured failure policy.

---

# 19. Configuration Through Environment Variables

## Decision

Runtime configuration should come from environment variables.

Example:

```text
SERVER_ADDR
REDIS_ADDR
RATE_LIMIT_CAPACITY
RATE_LIMIT_REFILL_RATE
RATE_LIMIT_FAILURE_MODE
REDIS_TIMEOUT
```

---

## Why

Environment-based configuration works well with:

- Docker
- Kubernetes
- CI/CD
- Cloud deployments
- Twelve-factor applications

Secrets should never be committed to source control.

---

# 20. No Database for Rate-Limit State

## Decision

The initial system does not use PostgreSQL or another durable database for bucket state.

---

## Why

Rate-limit state is:

- Temporary
- Frequently updated
- Latency-sensitive
- Derived from recent traffic

It does not need durable relational storage.

Business configuration such as subscription plans or quotas could later live in a database, while Redis contains the active runtime state.

---

# 21. Rate Limiter as a Library Component

## Decision

The core rate limiter remains an internal Go component rather than immediately becoming a standalone network service.

---

## Why

A separate rate-limiter service would introduce:

```text
Application
    |
Network
    |
Rate Limiter Service
    |
Network
    |
Redis
```

This adds network latency and another availability dependency.

Embedding the limiter into each application instance gives:

```text
Application
    |
Rate Limiter
    |
Redis
```

which is simpler and faster.

---

# 22. When a Dedicated Rate-Limiter Service Would Make Sense

A dedicated service may become appropriate when:

- Many unrelated applications need the same centralized policy.
- Rate-limit configuration must be centrally managed.
- Organizations require centralized observability.
- Different languages need the same implementation.
- Rate limiting becomes a platform capability.

That architecture would be:

```text
                +-------------+
                | Application |
                +------+------+
                       |
                       v
                +-------------+
                | Rate Limit  |
                | Service     |
                +------+------+
                       |
                       v
                    Redis
```

This is deliberately outside the first version.

---

# 23. Security Decision

## Decision

Do not expose Redis directly to clients.

Architecture:

```text
Internet
   |
   v
Application
   |
   v
Private Redis
```

Redis should remain inside a trusted network boundary.

Credentials must be provided through environment/secret management rather than source code.

---

# 24. Testing Strategy Decision

Testing is divided by responsibility.

### Unit tests

Test the algorithm without Redis.

### Concurrency tests

Use:

```bash
go test -race ./...
```

### Integration tests

Use a real Redis instance.

### Benchmarks

Measure local and distributed implementations independently.

### Failure tests

Explicitly simulate:

- Redis timeout
- Connection failure
- Redis unavailable
- Concurrent requests
- Expired buckets

---

# 25. Design Invariants

The implementation must preserve these invariants.

### Invariant 1

```text
0 <= tokens <= capacity
```

### Invariant 2

A successful request consumes exactly one token.

### Invariant 3

A rejected request does not consume a token.

### Invariant 4

Distributed bucket updates are atomic.

### Invariant 5

Different client keys have independent buckets.

### Invariant 6

Inactive bucket state eventually expires.

### Invariant 7

Infrastructure failures are distinguishable from normal rate-limit rejection.

### Invariant 8

Application instances do not maintain independent distributed rate-limit state.

---

# 26. Summary of Major Decisions

| Area | Decision | Primary Reason |
|---|---|---|
| Algorithm | Token Bucket | Burst tolerance + simple state |
| Local state | In-memory | Fast and simple |
| Local synchronization | Mutex | Protect compound state transition |
| Distributed state | Redis | Low latency shared state |
| Atomicity | Lua script | Atomic bucket transition |
| Expiration | Redis TTL | Prevent unbounded key growth |
| Refill | Lazy | Avoid per-bucket timers |
| HTTP integration | Middleware | Separation of concerns |
| Client identity | Client/API key | Clear isolation boundary |
| Timeouts | Bounded | Prevent request hangs |
| Failure behavior | Configurable | Availability vs protection |
| Configuration | Environment | Deployment flexibility |
| Core architecture | Embedded component | Avoid unnecessary network hop |
| Testing | Unit + race + integration + benchmark | Validate correctness and performance |

---

# 27. Core Architectural Principle

The most important design principle is:

> **Separate the rate-limiting algorithm from transport and persistence, while making the distributed state transition atomic.**

This results in:

```text
             HTTP
              |
              v
         Middleware
              |
              v
        Rate Limiter
              |
              v
       Storage Interface
          /         \
         /           \
     Memory          Redis
                       |
                       v
                  Lua Script
```

The design remains simple locally while providing a path toward a horizontally scalable distributed implementation.