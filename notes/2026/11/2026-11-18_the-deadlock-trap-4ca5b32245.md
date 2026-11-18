# 🛑 The Deadlock Trap

> Automatically generated interview-preparation note.

## Original problem

The Bug That Doesn't Crash — It Freezes.

## Interview-ready answer

## Problem understanding
Deadlocks occur in concurrent systems when two or more threads are each waiting for resources held by the others, causing an indefinite wait or freeze. This problem is critical in backend systems as it causes application freezes without crashing, making them difficult to detect and troubleshoot. The design goal is to prevent or detect deadlocks in a multi-threaded environment to maintain system responsiveness and correctness.

## Interview answer
To address deadlocks, first understand the four Coffman conditions necessary for a deadlock:

1. Mutual Exclusion: Only one thread can use a resource at a time.
2. Hold and Wait: Threads hold resources while waiting to acquire others.
3. No Preemption: Resources can only be released voluntarily.
4. Circular Wait: A cycle of threads exists where each thread waits for a resource held by the next.

To prevent deadlocks:

- **Resource Ordering**: Impose a global order in which threads acquire locks. For example, always acquire Lock A before Lock B.
- **Lock Timeout or TryLock**: Use timed lock acquisition attempts and abort or rollback if locks can't be obtained within a threshold.
- **Avoid Nested Locks**: Minimize holding multiple locks or nested locking.
- **Use Concurrency Utilities**: Opt for higher-level concurrency utilities (e.g., `java.util.concurrent` locks) that support features like `tryLock`.

To detect deadlocks at runtime, use JVM tools (e.g., `jstack`) or programmatically use `ThreadMXBean` to detect cycles in lock graphs.

Trade-offs:

- Resource ordering and avoiding locks reduce concurrency.
- Timed lock attempts add complexity.
- Detection adds overhead and is reactive rather than preventive.

## Java implementation

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class DeadlockPreventionExample {
    private final Lock lockA = new ReentrantLock();
    private final Lock lockB = new ReentrantLock();

    // Acquire locks in a fixed global order to prevent deadlock
    public void safeMethod() {
        // Always lock lockA before lockB
        try {
            if (lockA.tryLock(1, TimeUnit.SECONDS)) {
                try {
                    if (lockB.tryLock(1, TimeUnit.SECONDS)) {
                        try {
                            // Critical section: both locks held
                            doWork();
                        } finally {
                            lockB.unlock();
                        }
                    } else {
                        // Could not acquire lockB, release lockA and retry or abort
                        System.out.println("Timeout waiting for lockB, releasing lockA");
                    }
                } finally {
                    lockA.unlock();
                }
            } else {
                System.out.println("Timeout waiting for lockA");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void doWork() {
        // Business logic here
        System.out.println("Executing critical section");
    }
}
```

### Explanation:
- Uses `ReentrantLock` with `tryLock` and timeout to avoid indefinite waiting.
- Locks are always acquired in a fixed order: `lockA` then `lockB`.
- If lock acquisition fails, the method releases held locks and can retry or exit.
- This pattern avoids circular wait and indefinite blocking, preventing deadlocks.
- Proper exception handling and releasing locks are critical to avoid lock starvation or other concurrency issues.

## Key follow-up questions

1. Question: What are the main causes of deadlock in Java multithreading?
   Answer: Deadlocks mainly arise when threads acquire multiple locks out of order, hold locks while waiting for others, or use blocking synchronization mechanisms without timeout or release policies. The four Coffman conditions must be present simultaneously.

2. Question: How can you detect deadlocks programmatically in Java?
   Answer: Java’s `ThreadMXBean` in the `java.lang.management` package can detect deadlocks by analyzing thread and lock ownership graphs, enabling you to find if any threads form cycles in lock dependencies.

3. Question: What are the trade-offs between deadlock prevention by lock ordering vs detection and recovery?
   Answer: Lock ordering prevents deadlocks proactively but reduces flexibility and concurrency. Detection allows maximum concurrency but reacts after a deadlock occurs, requiring recovery strategies like thread interruption or restart, which can complicate system state management.

4. Question: Can using synchronized blocks cause deadlocks? How to avoid it?
   Answer: Yes, nested synchronized blocks on multiple objects can cause deadlocks if threads acquire locks in different orders. Avoid by establishing a strict global lock acquisition order and by minimizing scope and nesting of synchronized blocks.

5. Question: What role do `tryLock` and `lockInterruptibly()` play in deadlock prevention?
   Answer: `tryLock` allows a thread to attempt a lock without waiting indefinitely, enabling program logic to handle failure gracefully, like rolling back or retrying. `lockInterruptibly` allows a thread to be interrupted while waiting for a lock, preventing indefinite blocking.

## Takeaways
- Deadlocks freeze the system by threads waiting indefinitely for locks held by each other.
- Prevention strategies like global lock ordering and timed lock attempts avoid deadlocks proactively.
- Detection tools are useful for diagnosing deadlocks but are reactive.
- Use advanced concurrency utilities and design carefully to minimize nested locking.
- Proper lock release and exception handling prevent resource starvation and unexpected freezes.
- Writing deadlock-free code requires discipline in lock acquisition and comprehensive concurrency design.
