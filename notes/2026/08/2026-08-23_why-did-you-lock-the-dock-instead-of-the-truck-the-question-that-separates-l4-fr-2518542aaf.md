# 🏭 "Why Did You Lock the Dock Instead of the Truck?" — The Question That Separates L4 from L6

> Automatically generated interview-preparation note.

## Original problem

The interviewer gave me a loading dock problem — workers, trucks, packages. I started with the naive synchronized solution, then immediately said....

## Interview-ready answer

## Problem understanding

In a loading dock system involving workers, trucks, and packages, the challenge is to design a concurrent system that manages resource access efficiently and correctly. Specifically, the problem often involves coordinating multiple workers loading packages onto trucks at several docks without running into concurrency issues such as race conditions or deadlocks.

A common naive approach might be to synchronize (lock) on the truck objects themselves when accessing or modifying their state during the loading process. However, this can lead to various problems in terms of system throughput, granularity of locking, or contention.

The key insight, and what separates strong candidates (L6) from good ones (L4), is the distinction between locking the shared resource (the truck) versus the access point or boundary (the dock). Locking the dock instead of the truck allows better concurrency control and more natural synchronization semantics aligned with the physical workflow.

## Interview answer

When designing synchronization for the loading dock system, the initial idea might be to synchronize directly on the truck instance to ensure only one worker loads a given truck at a time. However, this can be inefficient and can lead to overly coarse locking.

Instead, I would lock on the dock because the dock is the critical section where work happens — only one truck can be at a dock at any time, and only one worker can load a truck at a given dock concurrently. By using the dock as the lock, we better model the real-world constraints and reduce contention across trucks that are at different docks.

This approach also decouples truck lifecycle management from the concurrency control logic. Trucks might move to different docks or be reused; the dock remains the synchronization anchor point.

In addition, this enables us to handle trucks waiting in a queue for a dock — only when a dock is free can a truck enter, simplifying the concurrency model.

## Java implementation

Below is an idiomatic Java implementation sketch demonstrating locking at the dock level rather than the truck level:

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Truck {
    private final String id;

    public Truck(String id) {
        this.id = id;
    }

    public String getId() {
        return id;
    }

    // Additional truck state and methods here...
}

class Dock {
    private final String id;
    private final Lock lock = new ReentrantLock();

    public Dock(String id) {
        this.id = id;
    }

    /**
     * Attempts to load the truck by acquiring the dock lock.
     * Only one truck can be loaded at this dock at a time.
     */
    public void loadTruck(Truck truck, Runnable loadTask) {
        lock.lock();
        try {
            System.out.println("Loading truck " + truck.getId() + " at dock " + id);
            loadTask.run(); // Simulate loading work
            System.out.println("Finished loading truck " + truck.getId() + " at dock " + id);
        } finally {
            lock.unlock();
        }
    }
}

class Worker implements Runnable {
    private final Dock dock;
    private final Truck truck;

    public Worker(Dock dock, Truck truck) {
        this.dock = dock;
        this.truck = truck;
    }

    @Override
    public void run() {
        dock.loadTruck(truck, () -> {
            try {
                // Simulate loading packages
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
}
```

**Notes:**

- The `Dock` class owns a `ReentrantLock` representing its availability.
- Workers synchronize on the dock, not the truck.
- Trucks can be passed around and scheduled independently.
- The `loadTruck` method serializes loading at a dock.

## Key follow-up questions

- What if multiple docks can handle the same truck simultaneously? How would the locking strategy change?
- How to handle trucks waiting for an available dock (a queue or semaphore)?
- How do we ensure fairness among trucks/workers in accessing the dock?
- What failure modes arise if a worker crashes or is interrupted while holding the dock lock?
- Could we improve throughput with finer-grained locking or lock-free data structures?
- How does the dock locking scale when the number of docks or trucks increases drastically?
- How would you design the system to handle partial loading from multiple workers concurrently on the same truck (if that is allowed)?

## Takeaways

- Synchronization based on real-world concurrency boundaries (the dock) rather than the resource being accessed (the truck) leads to cleaner and more scalable designs.
- Locking coarse-grained objects (the dock) fits naturally with physical resource contention logic.
- Java's `ReentrantLock` or `synchronized` can be used to lock critical regions safely.
- It's crucial to think through the workflows, resource contention points, and lifecycle of entities before deciding on locking strategies.
- Candidates who recognize the difference between locking the dock versus the truck demonstrate a deeper understanding of concurrency control and system design.
