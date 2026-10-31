# 🏭 "Why Did You Lock the Dock Instead of the Truck?" — The Question That Separates L4 from L6

> Automatically generated interview-preparation note.

## Original problem

The interviewer gave me a loading dock problem — workers, trucks, packages. I started with the naive synchronized solution, then immediately said....

## Interview-ready answer

## Problem understanding

The problem involves designing a system to manage loading docks, trucks, and workers handling packages. The key challenge is synchronization and concurrency control in a multi-threaded environment where multiple trucks arrive at multiple docks, workers load or unload packages concurrently, and resources must be efficiently utilized without race conditions or deadlocks.

Constraints and goals include:

- Multiple docks and trucks may operate concurrently.
- Workers can be assigned to load/unload packages.
- Synchronization must prevent inconsistent states (e.g., two trucks at the same dock simultaneously).
- The solution should maximize throughput (avoid unnecessary blocking) and maintain correctness.
- Trade-offs include coarse-grained (e.g., locking the whole dock area) vs fine-grained locking (per dock or truck) and their impact on concurrency.

## Interview answer

### Core design

1. **Resource modeled explicitly:** Model docks and trucks as separate resource entities, each potentially lockable. 
2. **Lock acquisition granularity matters:** 
   - Locking the dock is intuitive since trucks occupy docks to load.
   - However, locking only docks might cause bottlenecks if multiple trucks and workers contend on the same dock.
   - Locking trucks independently allows greater concurrency, especially if workers can load packages in parallel once a truck is assigned.
3. **Prevent deadlocks:** Acquire locks in a strict global order (e.g., always acquire dock lock then truck lock) or use a single composite lock for the pair.
4. **Workers assignation:** Once a truck has locked a dock, multiple workers can safely load/unload packages concurrently if package containers are independent — no package-level locking needed or limited synchronization per package.
5. **State consistency:** Use concurrent data structures or synchronization when modifying shared states such as package lists or load counts.
6. **Use high-level concurrency primitives:** Java’s `ReentrantLock`, `Semaphore`, or higher abstractions like `Phaser` or `CountDownLatch` for coordinating worker threads loading a given truck.

### Why lock the truck instead of the dock?

- Dock represents physical location, while trucks are movable resources that can be queued or assigned.
- Locking truck ensures that operations on the truck are consistent and concurrency-safe.
- Dock locks can be coarse, blocking the entire dock area and reducing parallelism.
- Locking the truck allows multiple docks to operate independently.

### Trade-offs

- Coarse locking (e.g., locking the dock) is simpler but causes poor scalability.
- Fine-grained locking (truck-level) increases concurrency but requires careful deadlock prevention.
- Complexity vs. throughput balance is necessary based on system scale and expected load.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

class Dock {
    private final int id;
    private final Lock lock = new ReentrantLock();

    public Dock(int id) {
        this.id = id;
    }

    public boolean tryLock() {
        return lock.tryLock();
    }

    public void unlock() {
        lock.unlock();
    }

    public int getId() {
        return id;
    }
}

class Truck {
    private final int id;
    private final Lock lock = new ReentrantLock();
    private final List<Package> packages = new ArrayList<>();

    public Truck(int id, List<Package> packages) {
        this.id = id;
        this.packages.addAll(packages);
    }

    public boolean tryLock() {
        return lock.tryLock();
    }

    public void unlock() {
        lock.unlock();
    }

    public void loadPackage(Package pkg, Worker worker) {
        lock.lock();
        try {
            // loading simulation: modifying package or truck state here
            System.out.printf("Worker %d loading package %d on Truck %d%n", worker.getId(), pkg.getId(), id);
            // e.g., package.setLoaded(true);
        } finally {
            lock.unlock();
        }
    }

    public int getId() {
        return id;
    }

    public List<Package> getPackages() {
        return packages;
    }
}

class Package {
    private final int id;
    public Package(int id) {
        this.id = id;
    }
    public int getId() {
        return id;
    }
}

class Worker implements Runnable {
    private final int id;
    private final Truck assignedTruck;

    public Worker(int id, Truck truck) {
        this.id = id;
        this.assignedTruck = truck;
    }

    public int getId() {
        return id;
    }

    @Override
    public void run() {
        for (Package pkg : assignedTruck.getPackages()) {
            assignedTruck.loadPackage(pkg, this);
            try {
                Thread.sleep(50); // simulate work delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        }
        System.out.printf("Worker %d finished loading Truck %d%n", id, assignedTruck.getId());
    }
}

public class LoadingDockSystem {
    private final List<Dock> docks;
    private final Queue<Truck> waitingTrucks = new ConcurrentLinkedQueue<>();
    private final ExecutorService workerPool = Executors.newCachedThreadPool();

    public LoadingDockSystem(int dockCount) {
        docks = new ArrayList<>();
        for (int i = 0; i < dockCount; i++) {
            docks.add(new Dock(i));
        }
    }

    public void submitTruck(Truck truck) {
        waitingTrucks.offer(truck);
        processTrucks();
    }

    private void processTrucks() {
        // Try to assign waiting trucks to free docks
        for (Dock dock : docks) {
            if (dock.tryLock()) {
                try {
                    Truck truck = waitingTrucks.poll();
                    if (truck == null) return;

                    // Lock truck to load packages exclusively on it
                    if (truck.tryLock()) {
                        try {
                            System.out.printf("Truck %d assigned to Dock %d%n", truck.getId(), dock.getId());

                            // Assign workers to load packages concurrently
                            int workerCount = Math.min(truck.getPackages().size(), 3); // limit number of workers
                            List<Future<?>> futures = new ArrayList<>();
                            for (int i = 1; i <= workerCount; i++) {
                                Worker worker = new Worker(i, truck);
                                futures.add(workerPool.submit(worker));
                            }

                            // Wait for all workers to finish loading packages
                            for (Future<?> f : futures) {
                                f.get();
                            }

                            System.out.printf("Truck %d loaded and leaving Dock %d%n", truck.getId(), dock.getId());
                        } catch (Exception e) {
                            e.printStackTrace();
                        } finally {
                            truck.unlock();
                        }
                    } else {
                        // truck locked by other process, put it back
                        waitingTrucks.offer(truck);
                    }
                } finally {
                    dock.unlock();
                }
            }
        }
    }

    public void shutdown() {
        workerPool.shutdown();
    }

    public static void main(String[] args) {
        LoadingDockSystem system = new LoadingDockSystem(2);
        List<Package> packages = Arrays.asList(new Package(1), new Package(2), new Package(3));
        system.submitTruck(new Truck(101, packages));
        system.submitTruck(new Truck(102, packages));

        system.shutdown();
    }
}
```

### Explanation

- `Dock` and `Truck` are lockable resource classes to manage synchronization.
- `LoadingDockSystem` assigns waiting trucks to docks as docks become available.
- Workers are created per truck and load packages concurrently.
- Locks prevent multiple trucks from occupying a dock or being loaded simultaneously incorrectly.
- Proper unlock and exception handling avoids deadlock and resource leaks.
- `ConcurrentLinkedQueue` for thread-safe truck waiting queue.
- Used thread pool to simulate multiple workers.

### Edge cases and failure modes

- Deadlocks if multiple locks requested out of order — handled by locking dock first, then truck.
- Starvation if trucks keep arriving and some trucks never get assigned. Could be improved with prioritization or fairness policies.
- Worker interruptions safely handled by thread interruption.
- Partial loading failures require rollback logic in a production system (not included for simplicity).

## Key follow-up questions

1. **Q:** Why not just use synchronized methods on dock or truck objects?
   **A:** Synchronized methods can cause coarse-grained locking, limiting concurrency and throughput. Using explicit locks with tryLock enables non-blocking attempts and better fine-grained control.

2. **Q:** How would you prevent deadlocks if trucks try to lock multiple docks or packages?
   **A:** Enforce a global lock ordering (e.g., always lock dock first, then truck), or use lock timeouts and retry mechanisms. Alternatively, use transactional memory or lock-free data structures.

3. **Q:** How to extend this to handle priority trucks or urgent packages?
   **A:** Introduce priority queues for trucks, prioritize locks granting based on urgency, or use priority-aware scheduling algorithms to assign docks and workers.

4. **Q:** How would you handle worker failure or interruption during package loading?
   **A:** Implement retry mechanisms, heartbeat checks, and transactional updates to roll back partial changes. Also, carefully handle thread interruptions and cleanup.

5. **Q:** Can you design a monitoring system to observe dock/truck/worker states?
   **A:** Yes, add state variables with thread-safe visibility (volatile, atomic) and expose metrics via monitoring APIs or logging. Possibly use event-driven design for real-time updates.

## Takeaways

- Fine-grained locking (truck-level) allows increased concurrency over coarse-grained dock locking.
- Deadlock prevention requires careful lock ordering and consistent locking strategy.
- Use high-level concurrency constructs to coordinate multiple worker threads efficiently.
- System design must balance simplicity vs performance and scalability.
- Always consider failure modes and recovery strategies in concurrent system design.
- Demonstrating both naive and advanced locking techniques shows maturity in low-level design interviews.
