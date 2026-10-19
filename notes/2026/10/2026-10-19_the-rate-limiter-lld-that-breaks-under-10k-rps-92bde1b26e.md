# The Rate Limiter LLD That Breaks Under 10K RPS

> Automatically generated interview-preparation note.

## Original problem

I'll use Strategy for Fixed and Sliding Window, Factory for creating limiters, Singleton...

## Interview-ready answer

## Problem understanding

Design a high-throughput Rate Limiter system that can handle at least 10,000 requests per second (RPS) reliably. The system should support different rate limiting strategies such as Fixed Window and Sliding Window, allow easy extensibility via a Factory pattern, and ensure global consistency (e.g., via Singleton pattern or shared state when necessary). The key constraints are high efficiency, thread safety, extensibility, and correctness under concurrency. The design must avoid performance bottlenecks or correctness flaws that cause the system to break down at scale.

## Interview answer

### Design goals
- **Support multiple rate limiting algorithms:** Fixed Window, Sliding Window, with pluggable strategy.
- **High throughput and thread safety:** Support at least 10K RPS without contention or race conditions.
- **Extensibility:** Using Factory pattern to instantiate specific limiter implementations based on configuration.
- **Singleton/global consistency:** Ensure that the limiters can be accessed from a shared context, avoiding duplicated or inconsistent state.
- **Fail-fast and maintain correctness:** Handle overflow, time window boundary transitions, and concurrency edge cases gracefully.

### Core design

1. **Strategy Pattern:** Define a common `RateLimiter` interface with a method like `boolean allowRequest(String clientId)`. Implement concrete strategies: `FixedWindowRateLimiter` and `SlidingWindowRateLimiter`.
   - *Fixed Window:* Uses counters reset on fixed time windows (e.g., every second).
   - *Sliding Window:* Maintains timestamps or buckets for requests to create a sliding view and avoid spikes at boundaries.

2. **Factory Pattern:** Implement a `RateLimiterFactory` that returns the correct `RateLimiter` instance based on configuration (e.g., strategy type, thresholds).

3. **Singleton:** Make the factory or manager singleton if the design requires a single shared source to get rate limiters. Possibly manage one rate limiter per client or API key.

4. **Data Structure & Concurrency:**
   - Use concurrent data structures like `ConcurrentHashMap` to store counters per client.
   - Use atomic operations or locks only when absolutely necessary.
   - For fixed window counters, `AtomicInteger` per client; for sliding windows, possibly `ConcurrentLinkedDeque` of timestamps or a pre-bucketed approach.
   
5. **Handling high RPS:**
   - Avoid expensive operations like sorting or multiple lock acquisitions.
   - Evict stale data efficiently.
   - Batch or approximate counts where precision can be sacrificed (e.g., leaky bucket algorithms).

### Trade-offs and pitfalls
- Fixed window is simpler but causes burstiness at window boundaries.
- Sliding window is smoother but more complex, higher memory, and potentially slower.
- Naively synchronized solutions will bottleneck and be unsuitable under high RPS.
- Singleton pattern simplifies management, but can cause contention if the implementation is not efficient.
- Memory growth with many clients must be controlled with eviction policies.

## Java implementation

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public interface RateLimiter {
    boolean allowRequest(String clientId);
}

// Fixed window implementation
class FixedWindowRateLimiter implements RateLimiter {
    private final int maxRequests;
    private final long windowSizeInMillis;

    private static class Window {
        volatile long windowStart;
        AtomicInteger count;

        Window(long windowStart) {
            this.windowStart = windowStart;
            this.count = new AtomicInteger(0);
        }
    }

    private final ConcurrentHashMap<String, Window> clientWindows = new ConcurrentHashMap<>();

    public FixedWindowRateLimiter(int maxRequests, long windowSizeInMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeInMillis = windowSizeInMillis;
    }

    @Override
    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();

        Window window = clientWindows.compute(clientId, (key, curWindow) -> {
            if (curWindow == null || now - curWindow.windowStart >= windowSizeInMillis) {
                return new Window(now);
            } else {
                return curWindow;
            }
        });

        while (true) {
            int currentCount = window.count.get();
            if (currentCount >= maxRequests) {
                return false;
            }
            if (window.count.compareAndSet(currentCount, currentCount + 1)) {
                return true;
            }
        }
    }

    // Optional: eviction of stale entries can be added to prevent memory leak
}

// Sliding window implementation (simple approximation)

class SlidingWindowRateLimiter implements RateLimiter {
    private final int maxRequests;
    private final long windowSizeInMillis;
    private final int numberOfBuckets;
    private final long bucketSizeInMillis;
    
    private static class SlidingWindow {
        long windowStart;
        final AtomicInteger[] buckets;

        SlidingWindow(int numberOfBuckets) {
            this.buckets = new AtomicInteger[numberOfBuckets];
            for(int i=0; i<numberOfBuckets; i++) {
                buckets[i] = new AtomicInteger(0);
            }
        }
    }

    private final ConcurrentHashMap<String, SlidingWindow> clientWindows = new ConcurrentHashMap<>();

    public SlidingWindowRateLimiter(int maxRequests, long windowSizeInMillis, int buckets) {
        this.maxRequests = maxRequests;
        this.windowSizeInMillis = windowSizeInMillis;
        this.numberOfBuckets = buckets;
        this.bucketSizeInMillis = windowSizeInMillis / buckets;
    }

    @Override
    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        SlidingWindow window = clientWindows.computeIfAbsent(clientId, k -> {
            SlidingWindow w = new SlidingWindow(numberOfBuckets);
            w.windowStart = now;
            return w;
        });

        synchronized (window) { // synchronized per client for correctness
            long elapsed = now - window.windowStart;
            if (elapsed >= windowSizeInMillis) {
                // reset window
                window.windowStart = now;
                for (AtomicInteger bucket : window.buckets) {
                    bucket.set(0);
                }
            }

            int currentBucket = (int)((elapsed % windowSizeInMillis) / bucketSizeInMillis);
            window.buckets[currentBucket].incrementAndGet();
            
            int total = 0;
            for (AtomicInteger bucket : window.buckets) {
                total += bucket.get();
            }
            return total <= maxRequests;
        }
    }
}

enum RateLimiterType {
    FIXED,
    SLIDING
}

class RateLimiterFactory {
    private static final RateLimiterFactory INSTANCE = new RateLimiterFactory();

    private RateLimiterFactory() {}

    public static RateLimiterFactory getInstance() {
        return INSTANCE;
    }

    public RateLimiter createRateLimiter(RateLimiterType type, int maxRequests, long windowSizeInMillis) {
        switch (type) {
            case FIXED:
                return new FixedWindowRateLimiter(maxRequests, windowSizeInMillis);
            case SLIDING:
                int buckets = 10; // customizable
                return new SlidingWindowRateLimiter(maxRequests, windowSizeInMillis, buckets);
            default:
                throw new IllegalArgumentException("Unknown rate limiter type");
        }
    }
}
```

### Details

- `FixedWindowRateLimiter` stores count per client and resets at fixed intervals.
- `SlidingWindowRateLimiter` approximates by dividing the time window into buckets. Synchronized per client for thread safety.
- `RateLimiterFactory` exposes a singleton for creating rate limiters.
- The design assumes multiple clients can be independently rate limited.
- Memory management (evicting old clients) is not implemented but is critical in production.

## Key follow-up questions

1. **Q:** How would you handle memory growth for a large number of clients generating requests?
   **A:** Use a cache with TTL eviction, for example a `ConcurrentHashMap` wrapped by a time-based eviction policy (e.g., LRU with expiration). Evict client entries that did not send requests recently to avoid unbounded memory use.

2. **Q:** Why is Fixed Window susceptible to bursting traffic spikes?
   **A:** Because counts reset abruptly at window boundaries, clients can send max requests at the end of one window and max again at the start of the next, allowing effectively double the allowed rate in a short time.

3. **Q:** How can you improve the sliding window to avoid synchronized blocks?
   **A:** Use concurrent data structures and atomic counters, or adopt approximate algorithms like token buckets or leaky buckets that can be implemented with lock-free structures.

4. **Q:** How do you ensure the system is resilient under multi-threaded access with high throughput?
   **A:** Use thread-safe data structures (`ConcurrentHashMap`, `AtomicInteger`), minimize lock contention (e.g., lock per key/client instead of global lock), and perform atomic or CAS operations for counters.

5. **Q:** How would you scale this design horizontally across multiple machines?
   **A:** Use a distributed shared state via Redis or a similar distributed cache supporting atomic increments. Alternatively, use client-side tokens with request metadata or consistent hashing to route requests to nodes to maintain client state consistency.

## Takeaways

- Choosing an appropriate rate limiting strategy impacts performance and accuracy; sliding window is smoother but more complex, fixed window is easier but prone to bursts.
- Efficient concurrent data structures and minimizing locks are essential at high throughput (10K+ RPS).
- Factory and Strategy patterns improve extensibility and decoupling; Singleton can be useful for shared factory or manager but beware of contention.
- Eviction strategies to handle memory growth are vital in long-running systems with many clients.
- Horizontal scaling generally requires distributed coordination or external shared stores.
- Careful benchmarking and profiling are necessary to identify bottlenecks and memory usage under high load scenarios.
