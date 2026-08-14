# 📦 His Dock System Held a Global Lock While Every Worker Waited in Line

> Automatically generated interview-preparation note.

## Original problem

My design was solid — Dock, Truck, Package, TruckState, Worker. The interviewer said 'Walk me through load(). What locks do you hold?' I said 'I synchronize on the Dock instance, check if the package fits, place it on the truck, and if the truck is full I dispatch it and create a new one — all under one lock.

## Interview-ready answer

## Problem understanding

The candidate designed a system managing loading packages onto trucks at a dock, involving these main entities: Dock, Truck, Package, TruckState, Worker. The key method `load()` handles placing packages onto trucks. During the interview, the candidate explained they used a single synchronized lock on the Dock instance to:

- Check if the package fits on the current truck
- Place the package on the truck
- Dispatch the truck if full and create a new one

This implies a coarse-grained global lock on the Dock object that serializes all loading operations by multiple workers, causing contention and waiting.

## Interview answer

The design as described ensures correctness (mutual exclusion when modifying shared state like the current truck and dock) but at a high concurrency cost. Holding a global lock on the Dock while each worker loads a package means only one worker can load at a time, turning the system into effectively a single-threaded bottleneck.

To improve this design for scalability, consider:

- Using finer-grained locking, e.g., lock per Truck rather than the entire Dock.
- Allow multiple workers to load concurrently as long as they operate on different trucks or packages.
- Use concurrent data structures (e.g., `ConcurrentLinkedQueue`) to accept packages or maintain waiting trucks.
- Employ lock-free or optimistic concurrency patterns if possible.
- Minimize the critical section: only synchronize on the minimum necessary state mutation.
- Use an AtomicInteger or similar for tracking truck fill level to avoid locking.
- Dispatching a truck and creating a new one can be done asynchronously or outside the main lock.

A practical approach might be:

- Synchronize only when checking/modifying the current truck.
- When the current truck is full, switch to another truck safely while dispatching happens separately.
- Workers waiting to load do not block each other unnecessarily.

This reduces contention, increases throughput, and makes the system more responsive under concurrency.

## Java implementation

Here is a simplified example showcasing a finer-grained design avoiding a global lock on Dock:

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.atomic.AtomicInteger;

class Package {
    int size;
    // other fields
}

class Truck {
    private final int capacity;
    private final AtomicInteger filledSize = new AtomicInteger(0);
    private final ReentrantLock lock = new ReentrantLock();

    public Truck(int capacity) {
        this.capacity = capacity;
    }

    public boolean tryLoadPackage(Package pkg) {
        lock.lock();
        try {
            if (filledSize.get() + pkg.size <= capacity) {
                filledSize.addAndGet(pkg.size);
                // Add package to truck's internal package list
                return true;
            }
            return false; // No space
        } finally {
            lock.unlock();
        }
    }

    public boolean isFull() {
        return filledSize.get() >= capacity;
    }

    // dispatch logic could be asynchronous
    public void dispatch() {
        System.out.println("Dispatching truck...");
        // dispatch work here
    }
}

class Dock {
    private volatile Truck currentTruck;
    private final int truckCapacity;

    public Dock(int truckCapacity) {
        this.truckCapacity = truckCapacity;
        this.currentTruck = new Truck(truckCapacity);
    }

    public void load(Package pkg) {
        while (true) {
            Truck truck = currentTruck;
            if (truck.tryLoadPackage(pkg)) {
                if (truck.isFull()) {
                    synchronized (this) {
                        if (truck == currentTruck && truck.isFull()) {
                            truck.dispatch();
                            currentTruck = new Truck(truckCapacity);
                        }
                    }
                }
                break;
            } else {
                // Current truck can't fit the package, try to switch trucks
                synchronized (this) {
                    if (truck == currentTruck) {
                        truck.dispatch();
                        currentTruck = new Truck(truckCapacity);
                    }
                }
            }
        }
    }
}
```

### Explanation:
- The `Truck` class uses a lock only for loading packages, allowing concurrent attempts to load on different trucks (if we managed a pool).
- The dock manages the single current truck, but synchronizes only when switching trucks and dispatching.
- Workers busy-wait retry if a truck is full or package can't fit, reducing global lock time.
- This pattern increases concurrency and reduces waiting.

## Key follow-up questions

- How do you handle truck dispatching asynchronously to avoid blocking the worker threads?
- What if multiple workers detect the truck is full simultaneously? How do you handle race conditions for creating a new truck?
- Can packages arrive at higher rates than loading speed? How do you handle backpressure or queue packages?
- How would you test this design under high concurrency?
- Would you consider using `StampedLock` or other advanced concurrency mechanisms? What are their advantages?
- How do you handle failure modes, e.g., package loading failed midway or truck dispatch failed?
- How does the design scale with thousands of workers and packages?

## Takeaways

- Coarse synchronization (a global lock for the whole Dock) simplifies correctness but severely penalizes concurrency and throughput.
- Fine-grained locking or lock-free designs enable better parallelism by reducing contention.
- Synchronization should cover only the absolute minimum critical section needed.
- Volatile or atomic variables can help coordinate state changes safely.
- Thoughtfully isolate areas requiring exclusivity (e.g., switching trucks) and separate concerns like dispatching.
- Design for concurrency must balance complexity and performance while ensuring correct behavior.
- Always consider thread contention and bottleneck points during design and be ready to explain trade-offs in interviews.
