# Engineer Fail the Gmail LLD Concurrency Round

> Automatically generated interview-preparation note.

## Problem understanding
The context is a Google Level 6 (senior engineer) interview question focused on concurrency in a low-level design (LLD) scenario, where a key challenge was to avoid using a global mutex. This implies the problem required designing a concurrent system or data structure that supports safe multi-threaded access without serializing all operations via a single lock, which can become a bottleneck and kill scalability.

A "global mutex" approach serializes all access and fails to utilize the power of concurrent programming. The interviewer expected a more fine-grained locking or lock-free design that allows safe parallelism, reducing contention and improving throughput.

## Interview answer
When asked to design a concurrent system (such as a cache, a counter, a map, or similar), explain why a global mutex is undesirable:

- It serializes all operations, creating a single point of contention.
- It reduces scalability since only one thread can proceed at a time.
- It can lead to poor latency when there is high concurrency.

Better approaches include:

1. **Fine-grained locking:** Lock parts of the data structure separately. For example, sharded locks or segment locks on subsets of data.

2. **Lock striping:** Use multiple locks that protect different buckets or partitions in a hash map to allow concurrent reads/writes that don’t collide.

3. **Lock-free or optimistic concurrency control:** Use atomic variables, compare-and-swap operations, or versions to achieve safe concurrency without blocking.

4. **Read-write locks:** If reads dominate, use ReadWriteLocks so multiple readers can proceed concurrently, with exclusive locks for writers.

5. **Thread-local or immutable data:** Reduce contention by avoiding shared mutable state.

Explain that the engineer "just used a global mutex" indicates a naive solution that does not scale and would fail in practical concurrency tests.

## Java implementation
An example showcasing fine-grained locking using lock striping for a concurrent map:

```java
import java.util.concurrent.locks.ReentrantLock;

public class StripedConcurrentMap<K, V> {
    private static final int DEFAULT_STRIPES = 16;
    private final ReentrantLock[] locks;
    private final Node<K, V>[] buckets;

    @SuppressWarnings("unchecked")
    public StripedConcurrentMap() {
        this.locks = new ReentrantLock[DEFAULT_STRIPES];
        for (int i = 0; i < DEFAULT_STRIPES; i++) {
            locks[i] = new ReentrantLock();
        }
        this.buckets = (Node<K, V>[]) new Node[DEFAULT_STRIPES];
    }

    private int stripeIndex(Object key) {
        return (key.hashCode() & 0x7fffffff) % DEFAULT_STRIPES;
    }

    public void put(K key, V value) {
        int stripe = stripeIndex(key);
        locks[stripe].lock();
        try {
            Node<K, V> node = buckets[stripe];
            Node<K, V> prev = null;
            while (node != null) {
                if (node.key.equals(key)) {
                    node.value = value;
                    return;
                }
                prev = node;
                node = node.next;
            }
            Node<K, V> newNode = new Node<>(key, value);
            if (prev == null) {
                buckets[stripe] = newNode;
            } else {
                prev.next = newNode;
            }
        } finally {
            locks[stripe].unlock();
        }
    }

    public V get(K key) {
        int stripe = stripeIndex(key);
        locks[stripe].lock();
        try {
            Node<K, V> node = buckets[stripe];
            while (node != null) {
                if (node.key.equals(key)) {
                    return node.value;
                }
                node = node.next;
            }
            return null;
        } finally {
            locks[stripe].unlock();
        }
    }

    private static class Node<K, V> {
        final K key;
        volatile V value;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
            this.next = null;
        }
    }
}
```

This design:

- Uses multiple locks protecting independent partitions (“stripes”) of a map.
- Allows concurrent operations on different stripes without waiting for a global mutex.
- Trades off complexity for better concurrency performance.
  
Alternatively, one could use `ConcurrentHashMap` in Java, which implements similar striping internally.

## Key follow-up questions
- What concurrency issues arise when using a global mutex?
- How does lock granularity affect performance and complexity?
- Can you explain the trade-offs of lock striping versus a concurrent data structure like Java’s `ConcurrentHashMap`?
- How would you handle read-heavy workloads versus write-heavy workloads in concurrency?
- What are potential deadlocks or starvation scenarios in fine-grained locking?
- How can lock-free algorithms improve concurrency, and what challenges do they bring?
- Explain when you would choose immutable data structures over locks for concurrency.

## Takeaways
- Using a global mutex is a simple but suboptimal way to ensure thread safety, limiting scalability and performance.
- Concurrency-sensitive designs benefit from finer lock granularity or lock-free techniques.
- Lock striping is a common pattern to reduce lock contention by partitioning the state.
- Built-in Java utilities like `ConcurrentHashMap` abstract these concurrency complexities.
- Understanding concurrency trade-offs and failure modes (deadlocks, contention, starvation) is crucial during LLD interviews.
- Always strive for designs that maximize concurrent access while ensuring correctness and minimizing complexity.
