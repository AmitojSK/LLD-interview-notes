# 🏭 "Why Did You Lock the Dock Instead of the Truck?" — The Question That Separates L4 from L6

> Automatically generated interview-preparation note.

## Original problem

The interviewer gave me a loading dock problem — workers, trucks, packages. I started with the naive synchronized solution, then immediately said....

## Interview-ready answer

## Problem understanding

The problem involves coordinating multiple entities: workers, trucks, and packages, in a loading dock environment. The main challenge is to model synchronization and resource access such that packages are loaded onto trucks efficiently without deadlocks, race conditions, or resource contention. Constraints include:

- Workers and trucks must be coordinated to avoid conflicts.
- Packages are the units being loaded/unloaded.
- The system must scale and be robust with multiple concurrent workers and trucks.
- The design must consider locking strategies to prevent contention and deadlocks.

Design goals:

- Concurrency safety and correctness.
- Maximize throughput and resource utilization.
- Avoid coarse-grained locking that leads to performance bottlenecks.
- Balance between complexity and correctness.

## Interview answer

### Core design

1. **Entities:**
   - **Truck**: Resource where packages are loaded.
   - **Dock**: A physical location allowing interaction between workers and trucks.
   - **Worker**: Operates to load packages from the dock onto trucks.

2. **Synchronization Approach:**

   - The naive solution is to use a single global lock (e.g., synchronized method/block) to protect all operations involving trucks and packages. This is simple but causes contention, reducing concurrency to effectively sequential processing.
   
   - A better approach is to **lock at the resource level**, e.g., lock the Dock rather than the Truck, reflecting the real-world constraint that only one worker (or truck) can use a dock at a time.

3. **Why lock Dock rather than Truck?**

   - **Dock is a limited resource.** Multiple trucks may exist, but only a few docks. A dock serializes access.
   - Trucks can come and go and often represent clients of the dock resource.
   - By locking the Dock, workers serialize access to the limited physical space, preventing collisions and deadlocks.
   - Locking trucks can cause complex locking hierarchies, leading to potential deadlocks if a worker tries to lock a truck and a dock or vice versa.

4. **Locking Strategy:**

   - Lock on the dock when assigning a truck and loading packages.
   - Inside dock lock, assign or release trucks, and perform package loading/unloading.
   - Trucks themselves may have internal locks if they hold internal state that needs concurrency control unrelated to the dock itself.

5. **Trade-offs:**

   - Dock-level locking restricts concurrency to the number of docks, which is natural.
   - Truck-level locking can lead to deadlocks if multiple trucks are locked by different threads trying to acquire docks.
   - Global locking reduces concurrency significantly.

### Summary:

- Prevent deadlocks by choosing the primary lock on the limited resource (Dock).
- Design for natural serialization points.
- Consider making trucks and docks distinct concurrency units.
- Avoid coarse global locks or fine-grain multi-locking that leads to complex deadlocks.

## Java implementation

Below is a simplified implementation demonstrating locking the Dock rather than the Truck:

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

// Represents a dock where trucks can load packages
class Dock {
    private final Lock dockLock = new ReentrantLock();

    // Only one truck can use a dock at a time
    public void useDock(Truck truck, Worker worker) {
        dockLock.lock();
        try {
            System.out.println(worker.getName() + " starts loading packages to truck " + truck.getId());
            truck.loadPackages(worker.loadPackages());
            System.out.println(worker.getName() + " finished loading packages to truck " + truck.getId());
        } finally {
            dockLock.unlock();
        }
    }
}

// Represents a truck collecting packages
class Truck {
    private final String id;

    public Truck(String id) {
        this.id = id;
    }

    public String getId() {
        return id;
    }

    // For simplicity, load packages directly; could be synchronized if needed
    public void loadPackages(int packageCount) {
        System.out.println("Loading " + packageCount + " packages onto truck " + id);
    }
}

// Represents a worker who loads packages
class Worker {
    private final String name;

    public Worker(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    // The worker loads packages from inventory or dock
    public int loadPackages() {
        // Simulate package loading
        return 5; // Packages loaded
    }
}
```

### Concurrency considerations

- `Dock` uses its lock (`dockLock`) to serialize access to the loading space.
- Multiple docks may exist, allowing parallel processing.
- `Truck` assumes thread-safety or manages its synchronization internally if needed.
- Workers invoke `useDock()` to load packages, ensuring no two workers load on the same dock concurrently.

## Key follow-up questions

1. **Why not lock on the truck?**

   Locking on the truck can cause deadlocks when multiple workers try to load different trucks and compete for docks, creating circular waits. The dock is the constrained resource, so it is safer to lock on dock.

2. **What if there are multiple docks and multiple workers?**

   Each dock can have an independent lock. Workers compete only for dock locks. This scales well to the number of docks available.

3. **How would you avoid starvation?**

   Use fair locks (`new ReentrantLock(true)`) or queue requests explicitly to ensure all workers get access eventually.

4. **Could you use `synchronized` methods instead of `ReentrantLock`?**

   Yes, but `ReentrantLock` provides more flexibility (e.g., fairness, tryLock semantics). For simplicity, synchronized blocks might suffice.

5. **How would you handle trucks arriving and leaving dynamically?**

   Use a concurrent data structure (e.g., `ConcurrentLinkedQueue`) to hold waiting trucks and assign them to docks in a thread-safe way, locking docks as needed.

6. **What happens if a worker crashes while holding a dock lock?**

   With a JVM, locks held by the thread will be released on thread termination, but incomplete loading could leave state inconsistent. Use higher-level transaction or failure handling to recover from partial state.

## Takeaways

- Identify the true contention point in the system (Dock) rather than the object being acted on (Truck).
- Granular, well-focused locking reduces deadlock risk and increases concurrency.
- Using locks on limited resources (docks) reflects real-world constraints and guides cleaner synchronization.
- Always reason about lock acquisition order and resource constraints to avoid deadlocks.
- Avoid coarse-grained global locks; they simplify correctness but limit scalability.
- Consider lock fairness and starvation prevention in production systems.
- Think about failure modes and how partial operations affect consistency in concurrent systems.
