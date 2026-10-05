# Distributed Rate Limiter — Low-Level Design

## 1. Purpose

This document defines the low-level design of the distributed rate limiter.

The implementation is written in Go and uses the Token Bucket algorithm.

The design supports two storage implementations:

1. In-memory storage for local development and unit testing.
2. Redis storage for distributed deployments.

The rate-limiting algorithm remains independent of the storage implementation.

---

# 2. Package Structure

```text
distributed-rate-limiter/
│
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   ├── limiter/
│   │   ├── limiter.go
│   │   ├── token_bucket.go
│   │   ├── local_limiter.go
│   │   └── distributed_limiter.go
│   │
│   ├── storage/
│   │   ├── store.go
│   │   └── redis_store.go
│   │
│   ├── middleware/
│   │   └── http.go
│   │
│   └── config/
│       └── config.go
│
├── pkg/
│   └── response/
│       └── response.go
│
└── tests/
    ├── integration/
    └── benchmark/
```

---

# 3. Core Abstractions

The most important design principle is separation of concerns.

```text
HTTP Middleware
       |
       v
RateLimiter
       |
       v
Token Bucket Logic
       |
       v
Storage
       |
       +------ Memory
       |
       +------ Redis
```

The limiter should not directly depend on Redis.

---

# 4. RateLimiter Interface

File:

```text
internal/limiter/limiter.go
```

The primary interface is:

```go
type RateLimiter interface {
    Allow(ctx context.Context, key string) (Result, error)
}
```

The interface accepts:

- `context.Context` for cancellation and deadlines.
- `key` identifying the client.

It returns:

- A rate-limit decision.
- Metadata about the decision.
- An error when the decision cannot be safely determined.

---

# 5. Result

The limiter returns a structured result:

```go
type Result struct {
    Allowed    bool
    Limit      int
    Remaining  int
    RetryAfter time.Duration
}
```

### Fields

| Field | Meaning |
|---|---|
| `Allowed` | Whether the request may continue |
| `Limit` | Maximum bucket capacity |
| `Remaining` | Tokens currently available |
| `RetryAfter` | Approximate time before another request may succeed |

Example:

```text
Allowed:    false
Limit:      100
Remaining:  0
RetryAfter: 2s
```

---

# 6. Rate-Limit Configuration

The Token Bucket requires:

```go
type Config struct {
    Capacity   int
    RefillRate float64
}
```

Where:

### Capacity

Maximum number of tokens that can exist in the bucket.

Example:

```text
Capacity = 100
```

### RefillRate

Number of tokens added per second.

Example:

```text
RefillRate = 10
```

This represents approximately:

```text
10 requests / second
```

while allowing a burst of up to:

```text
100 requests
```

---

# 7. Token Bucket State

Each client has logical state:

```go
type Bucket struct {
    Tokens     float64
    LastRefill time.Time
}
```

Example:

```text
Bucket
├── Tokens: 73
└── LastRefill: 2026-10-05T12:00:03Z
```

The state must never allow:

```text
Tokens < 0
```

or:

```text
Tokens > Capacity
```

---

# 8. Token Refill Algorithm

Given:

```text
currentTime
lastRefillTime
refillRate
capacity
```

calculate:

```text
elapsed = currentTime - lastRefillTime
```

Then:

```text
newTokens = elapsedSeconds × refillRate
```

And:

```text
tokens = min(capacity, tokens + newTokens)
```

Finally:

```text
lastRefillTime = currentTime
```

Example:

```text
Capacity    = 10
RefillRate  = 2 tokens/sec
Current     = 4 tokens
Elapsed     = 2 sec
```

Therefore:

```text
new tokens = 2 × 2
           = 4

tokens = 4 + 4
       = 8
```

---

# 9. Token Consumption

For a request requiring one token:

```text
if tokens >= 1:
    tokens -= 1
    allowed = true
else:
    allowed = false
```

For the initial implementation, every request costs exactly one token.

Future versions may support variable request costs.

---

# 10. Retry Calculation

When:

```text
tokens < 1
```

the request is rejected.

The approximate wait time is:

```text
requiredTokens = 1 - tokens

retryAfter = requiredTokens / refillRate
```

Example:

```text
tokens = 0.25
refillRate = 2 tokens/sec
```

Then:

```text
required = 0.75

retryAfter = 0.75 / 2
           = 0.375 seconds
```

The HTTP layer may round this value appropriately for the `Retry-After` header.

---

# 11. Local Limiter

File:

```text
internal/limiter/local_limiter.go
```

The local limiter maintains bucket state inside the application process.

Conceptually:

```text
LocalLimiter
│
├── configuration
│
├── buckets
│
└── synchronization
```

Buckets are indexed by client key:

```text
map[string]*Bucket
```

Example:

```text
buckets
├── client_123 → Bucket
├── client_456 → Bucket
└── client_789 → Bucket
```

---

# 12. Concurrency Control

Multiple goroutines can access the map simultaneously.

Therefore access must be synchronized.

The local implementation will use:

```go
sync.Mutex
```

or an appropriately scoped synchronization strategy.

The critical section includes:

```text
1. Locate bucket
2. Refill bucket
3. Check tokens
4. Consume token
5. Update timestamp
```

These operations must be treated as one logical state transition.

We must not allow:

```text
Goroutine A → read tokens
Goroutine B → read tokens
Goroutine A → consume
Goroutine B → consume
```

to produce an inconsistent result.

---

# 13. Local Limiter Flow

```text
Allow(key)
   |
   v
Acquire lock
   |
   v
Find bucket
   |
   +---- Doesn't exist
   |         |
   |         v
   |      Create bucket
   |
   v
Calculate elapsed time
   |
   v
Refill tokens
   |
   v
tokens >= 1 ?
   |
   +---- YES ----> Consume token
   |                   |
   |                   v
   |               Allowed
   |
   +---- NO -----> Calculate retry
                       |
                       v
                    Rejected
   |
   v
Release lock
```

---

# 14. Storage Abstraction

File:

```text
internal/storage/store.go
```

The storage layer abstracts persistence of bucket state.

The exact interface will be kept intentionally small.

Conceptually:

```go
type Store interface {
    Get(ctx context.Context, key string) (BucketState, error)
    Set(ctx context.Context, key string, state BucketState, ttl time.Duration) error
}
```

However, this interface is primarily useful for simple/local implementations.

For the distributed Redis implementation, a normal:

```text
Get → calculate → Set
```

sequence is unsafe.

Therefore Redis will eventually expose an atomic operation specifically designed for rate limiting.

---

# 15. Redis Storage

File:

```text
internal/storage/redis_store.go
```

Redis is responsible for shared bucket state.

The logical key is:

```text
rate_limit:{client_key}
```

Example:

```text
rate_limit:client_123
```

A bucket stores:

```text
tokens
last_refill
```

The exact Redis representation will be optimized during implementation.

---

# 16. Redis TTL

Inactive buckets should not remain indefinitely.

Example:

```text
Client sends requests
       |
       v
Redis bucket created
       |
       v
Client stops sending requests
       |
       v
TTL expires
       |
       v
Bucket removed
```

This prevents unbounded growth from clients that make only occasional requests.

The TTL should be derived from the configured refill behavior rather than using an arbitrary permanent key.

---

# 17. Distributed Limiter

File:

```text
internal/limiter/distributed_limiter.go
```

The distributed limiter uses Redis as the shared state store.

Conceptually:

```text
DistributedLimiter
│
├── Redis store
├── configuration
└── failure policy
```

Request flow:

```text
Allow(key)
    |
    v
Redis atomic operation
    |
    +---- Allowed
    |
    +---- Rejected
    |
    +---- Redis error
```

---

# 18. Redis Atomicity

A distributed request must perform the following operations atomically:

```text
1. Read bucket state
2. Determine elapsed time
3. Refill tokens
4. Cap tokens at capacity
5. Check whether a token is available
6. Consume token if available
7. Calculate remaining tokens
8. Calculate retry time if required
9. Persist state
10. Set/update expiration
```

These operations must execute as a single atomic Redis operation.

The planned mechanism is a **Redis Lua script**.

---

# 19. Redis Lua Script

Conceptually:

```text
Application
     |
     | EVAL
     v
Redis
     |
     v
+---------------------------+
| Lua Script                |
|                           |
| Read state                |
| Calculate refill          |
| Check capacity            |
| Consume token             |
| Update state              |
| Set expiration            |
| Return result             |
+---------------------------+
```

The application should receive the complete decision from Redis.

For example:

```text
[
    allowed,
    remaining,
    retry_after
]
```

This prevents the application from performing separate Redis operations that could race with another application instance.

---

# 20. Distributed Request Flow

```text
Server 1                       Redis

Request
   |
   v
Limiter
   |
   |---- atomic script ------->|
   |                            |
   |                       Read bucket
   |                       Refill
   |                       Consume
   |                       Update
   |                            |
   |<---- result ---------------|
   |
   v
HTTP response
```

Server 2 can perform the same operation simultaneously.

Redis serializes the atomic script execution for the affected state.

---

# 21. Clock Handling

Token refill depends on elapsed time.

The system should avoid relying on an application instance's local wall-clock timestamp for distributed correctness where possible.

Potential clock differences:

```text
Server 1 → 12:00:01.000
Server 2 → 12:00:01.100
```

can produce slightly different calculations.

For the distributed implementation, time handling should therefore be designed so that the state transition is evaluated consistently inside Redis.

The exact timestamp strategy will be finalized during implementation.

---

# 22. Middleware

File:

```text
internal/middleware/http.go
```

The middleware wraps an HTTP handler.

Conceptually:

```go
func RateLimit(next http.Handler) http.Handler
```

Flow:

```text
HTTP Request
     |
     v
Extract client key
     |
     v
limiter.Allow(...)
     |
     +---- Error ------> failure policy
     |
     +---- Rejected ---> 429
     |
     +---- Allowed ----> next.ServeHTTP
```

---

# 23. Client Key Extraction

The initial implementation will use an API key.

For example:

```http
Authorization: Bearer <token>
```

or:

```http
X-API-Key: client_123
```

The middleware converts this into:

```text
client_123
```

which becomes the limiter key.

The extraction mechanism should remain separate from the Token Bucket logic.

---

# 24. HTTP Response Handling

### Allowed

The middleware sets:

```http
X-RateLimit-Limit: <limit>
X-RateLimit-Remaining: <remaining>
```

and calls the next handler.

### Rejected

The middleware returns:

```http
HTTP/1.1 429 Too Many Requests
```

with:

```http
Retry-After: <seconds>
X-RateLimit-Limit: <limit>
X-RateLimit-Remaining: 0
```

The request must not reach the application handler.

---

# 25. Failure Policy

The distributed limiter can encounter:

```text
Redis timeout
Redis connection failure
Redis unavailable
Lua execution error
Invalid Redis response
```

The limiter will expose an explicit failure policy.

Conceptually:

```go
type FailureMode int

const (
    FailOpen FailureMode = iota
    FailClosed
)
```

### Fail Open

```text
Redis error
    |
    v
Allow request
```

### Fail Closed

```text
Redis error
    |
    v
Reject request
```

The default will be selected after evaluating the intended use case.

---

# 26. Configuration

File:

```text
internal/config/config.go
```

Configuration should include:

```text
server address
Redis address
Redis credentials
bucket capacity
refill rate
Redis timeout
bucket TTL
failure mode
```

Configuration should be loaded from environment variables.

Example:

```text
SERVER_ADDR=:8080
REDIS_ADDR=localhost:6379
RATE_LIMIT_CAPACITY=100
RATE_LIMIT_REFILL_RATE=10
RATE_LIMIT_FAILURE_MODE=fail_closed
```

Secrets must not be hardcoded into the repository.

---

# 27. Dependency Direction

The dependency graph should remain:

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
```

The reverse dependency should be avoided.

For example:

```text
storage → middleware
```

should not occur.

This keeps the core limiter independent of HTTP.

---

# 28. Error Handling

Errors should be categorized.

Examples:

```text
ErrRedisUnavailable
ErrRedisTimeout
ErrInvalidConfiguration
ErrInvalidClientKey
ErrStorageFailure
```

The HTTP middleware should not expose internal infrastructure errors directly to clients.

For example:

```text
Internal error:
"redis: connection refused"

Client response:
"Rate limiter temporarily unavailable"
```

Detailed errors should remain available to logs/metrics.

---

# 29. Testing Boundaries

The design allows each component to be tested independently.

### Token Bucket

```text
TokenBucket
    |
    +-- refill tests
    +-- consumption tests
    +-- capacity tests
    +-- retry tests
```

### Local Limiter

```text
LocalLimiter
    |
    +-- concurrency tests
    +-- multiple clients
    +-- isolation
```

### Redis Limiter

```text
DistributedLimiter
    |
    +-- shared state
    +-- atomicity
    +-- expiration
    +-- Redis failures
```

### Middleware

```text
HTTP
 |
Middleware
 |
Mock Limiter
```

This avoids requiring Redis for every unit test.

---

# 30. Concurrency Requirements

The following must be true:

### Local implementation

For N concurrent requests and an initially full bucket:

```text
accepted <= capacity
```

within the relevant interval.

### Distributed implementation

Across multiple application instances:

```text
total accepted requests
```

must respect the shared bucket state.

The system must not produce additional tokens because multiple servers accessed the same bucket concurrently.

---

# 31. Benchmark Targets

Benchmarks should compare:

```text
Local limiter
vs
Redis-backed limiter
```

Metrics:

```text
operations/sec
ns/op
allocations/op
p50 latency
p95 latency
p99 latency
```

The benchmark should also test:

```text
single client
many clients
low contention
high contention
```

---

# 32. Security Considerations

The rate limiter itself should not trust arbitrary user-controlled identifiers without validation.

Consider:

```text
very long API keys
empty keys
malformed keys
high-cardinality attacker-generated keys
```

A malicious client could attempt to create millions of unique keys.

Therefore:

- Validate client identifiers.
- Limit identifier length.
- Apply Redis TTLs.
- Avoid logging secrets.
- Do not expose Redis credentials.
- Apply infrastructure-level protection where appropriate.

---

# 33. Final Component Relationships

```text
                    +----------------+
                    | HTTP Middleware|
                    +-------+--------+
                            |
                            v
                    +---------------+
                    | RateLimiter   |
                    |   Interface   |
                    +-------+-------+
                            |
               +------------+------------+
               |                         |
               v                         v
       +---------------+         +---------------+
       | LocalLimiter  |         | Distributed   |
       |               |         | Limiter       |
       +-------+-------+         +-------+-------+
               |                         |
               v                         v
       +---------------+         +---------------+
       | Memory State  |         | Redis Store   |
       +---------------+         +-------+-------+
                                         |
                                         v
                                  +-------------+
                                  | Lua Script  |
                                  +-------------+
```

The core design principle is:

> **The rate-limiting policy and algorithm are independent from transport and persistence.**

This allows the same conceptual rate limiter to operate locally for testing and through Redis for distributed deployments.