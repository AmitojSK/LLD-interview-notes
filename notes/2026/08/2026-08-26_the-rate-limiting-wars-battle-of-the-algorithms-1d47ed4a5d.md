# ⏱️ The Rate Limiting Wars: Battle of the Algorithms

> Automatically generated interview-preparation note.

## Original problem

Fixed Windows vs Token Buckets vs Sliding Logs — Which Algorithm Survives Production Traffic?

## Interview-ready answer

## Problem understanding

Rate limiting is a fundamental technique to control the number of requests a system accepts over time, protecting backend resources from overload and abuse. The key challenge is how to implement rate limiting that is both fair and performant under production traffic conditions. The main algorithms commonly used include:

- **Fixed Window Counter**: Counts requests in fixed time intervals (e.g., per minute). Simple but can lead to bursts at window edges.
- **Token Bucket**: Tokens are added at a fixed rate; each request consumes a token if available. Allows smooth request rate and bursting up to bucket size.
- **Sliding Logs**: Maintains a timestamped log of requests and counts requests within a sliding time window. Most precise but costly in memory and lookup time.

The email's posed question is which algorithm "survives" production traffic best, i.e., which balances fairness, performance, and memory use under high load.

## Interview answer

Each rate limiting algorithm has trade-offs:

- **Fixed Window** is the simplest and easiest to implement. It uses low memory and CPU but suffers from boundary conditions where a user can send twice the limit in a short burst spanning two windows. This may or may not be acceptable depending on application tolerance for bursts.

- **Token Bucket** improves on Fixed Window by allowing smooth rate limiting with natural bursting up to bucket capacity. It is efficient since the state is just the token count and timestamp. It requires careful synchronization in distributed systems but gracefully handles bursty traffic.

- **Sliding Log** offers the most accurate rate limit enforcement by strictly counting requests within a sliding window. However, it requires storing each timestamp (potentially many entries), consuming more memory and CPU to prune old entries. It may become a bottleneck under high traffic unless optimized with data structures like time buckets or approximations.

In high-throughput, production environments, token buckets are typically preferred due to their balance of accuracy and efficiency. Fixed windows may be acceptable in low traffic or non-critical paths, and sliding logs are chosen if precise strict limits are essential despite overhead.

The ultimate choice depends on the system's tolerance for burstiness, required accuracy, available memory, and request load.

## Java implementation

Here is a simplified, thread-safe Java implementation of a **Token Bucket** rate limiter.

```java
import java.util.concurrent.atomic.AtomicLong;

public class TokenBucketRateLimiter {
    private final long capacity;
    private final long refillTokensPerMillis;
    private AtomicLong availableTokens;
    private volatile long lastRefillTimestamp;

    /**
     * @param capacity          Maximum number of tokens in the bucket
     * @param refillTokensPerSec Number of tokens added per second
     */
    public TokenBucketRateLimiter(long capacity, long refillTokensPerSec) {
        this.capacity = capacity;
        this.refillTokensPerMillis = refillTokensPerSec / 1000;
        this.availableTokens = new AtomicLong(capacity);
        this.lastRefillTimestamp = System.currentTimeMillis();
    }

    /**
     * Attempts to consume one token. Returns true if successful.
     */
    public synchronized boolean tryConsume() {
        refill();

        if (availableTokens.get() > 0) {
            availableTokens.decrementAndGet();
            return true;
        }
        return false;
    }

    /**
     * Refill tokens based on elapsed time
     */
    private synchronized void refill() {
        long now = System.currentTimeMillis();
        long elapsedMillis = now - lastRefillTimestamp;

        if (elapsedMillis > 0) {
            long tokensToAdd = elapsedMillis * refillTokensPerMillis;
            if (tokensToAdd > 0) {
                long newTokenCount = Math.min(capacity, availableTokens.get() + tokensToAdd);
                availableTokens.set(newTokenCount);
                lastRefillTimestamp = now;
            }
        }
    }
}
```

**Discussion:**

- The bucket refills at a fixed rate.
- The `tryConsume()` method is synchronized to ensure atomicity.
- This implementation assumes a single JVM instance. In a distributed system, token state must be shared via external storage like Redis or consistent hash ring.
- Token refill granularity is in milliseconds, which can be adjusted.
- For high throughput, consider using lock-free structures or optimistic concurrency.

## Key follow-up questions

1. **How would you implement distributed rate limiting?**
   - Using a centralized data store (Redis, Memcached) with atomic increment and expiration operations.
   - Using distributed consensus or consistent hashing.
2. **What are the downsides of Fixed Window and how to mitigate?**
   - Bursts at window edges; mitigated by sliding window or token bucket.
3. **Can sliding logs be optimized?**
   - Use approximate data structures like leaky bucket or time-bucketed counters to reduce memory overhead.
4. **How to handle multiple clients or users?**
   - Maintain separate buckets keyed by user ID or API key.
5. **What failure modes should you anticipate?**
   - Clock skew causing incorrect token refill.
   - Synchronization bottlenecks under high concurrency.
   - State loss or inconsistency in distributed env.

## Takeaways

- Rate limiting algorithms balance accuracy, memory, and performance.
- Fixed window counters are simple but suffer from burstiness.
- Token buckets provide smoothing and controlled bursts, ideal for production load balancing.
- Sliding logs are the most accurate but resource intensive.
- Proper choice depends on system constraints, required precision, and traffic patterns.
- Implementation details like concurrency control and state storage strongly impact system behavior.
- Distributed environments require external shared state and atomic operations to maintain correctness.
