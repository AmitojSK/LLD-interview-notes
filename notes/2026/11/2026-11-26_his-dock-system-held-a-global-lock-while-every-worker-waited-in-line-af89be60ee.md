# 📦 His Dock System Held a Global Lock While Every Worker Waited in Line

> Automatically generated interview-preparation note.

## Original problem

My design was solid — Dock, Truck, Package, TruckState, Worker. The interviewer said 'Walk me through load(). What locks do you hold?' I said 'I synchronize on the Dock instance, check if the package fits, place it on the truck, and if the truck is full I dispatch it and create a new one — all under one lock.

## Interview-ready answer

## Problem understanding

Design a concurrent system modeling a dock where workers load packages onto trucks. The system includes these entities:  
- **Dock**: coordinates loading.  
- **Truck**: collects packages until full, then is dispatched.  
- **Package**: an item to be loaded.  
- **TruckState**: tracks truck capacity/filled state.  
- **Worker**: a thread/process that loads packages.

**Constraints and goals:**  
- Multiple workers load packages concurrently.  
- Trucks have capacity limits.  
- Once a truck is full, it must be dispatched and a new truck used.  
- Avoid performance bottlenecks caused by locks held too long.  
- Ensure thread safety and correctness (no lost packages, no overfilling).

## Interview answer

The initial approach — synchronizing the entire `load()` method on the Dock instance — ensures correctness but creates a global lock contention point. All workers block waiting to acquire the Dock lock, resulting in low concurrency and throughput. This design leads to increased wait times, under-utilized worker threads, and potential scaling issues.

**Improved design considerations:**  
1. **Minimize lock scope:** Only protect the critical state updates related to truck capacity and package placement. Avoid holding locks for operations like dispatching trucks if that can be done asynchronously or outside the lock.  
2. **Use finer-grained synchronization:** Protect per-truck state or leverage concurrent data structures.  
3. **Immutable or thread-safe Packages:** Avoid unnecessary synchronization on package instances.  
4. **Worker coordination:** Use queued work or concurrency primitives like semaphores, conditions, or `Lock`/`Condition` to allow workers to wait efficiently when the dock is full or when a truck is being dispatched.

**Trade-offs:**  
- More granular locking introduces complexity and potential for subtle concurrency bugs such as deadlocks or missed updates.  
- Asynchronous dispatching improves throughput but requires thread coordination.  
- Using concurrency utilities (e.g., `java.util.concurrent`) can simplify correctness guarantees.

**Summary:** Synchronizing the whole load method serializes all workers unnecessarily; instead, protect minimal critical sections, separate state updates from long-running operations (like dispatching), and use concurrent structures or coordination mechanisms to achieve higher concurrency and throughput.

## Java implementation

Below is a simplified idiomatic Java sketch showing how to reduce locking, minimize lock scope, and asynchronously dispatch full trucks:

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.ReentrantLock;

class Package {
    private final int size;
    public Package(int size) { this.size = size; }
    public int getSize() { return size; }
}

class Truck {
    private final int capacity;
    private int usedCapacity = 0;
    private final List<Package> packages = new ArrayList<>();
    private final ReentrantLock lock = new ReentrantLock();

    public Truck(int capacity) {
        this.capacity = capacity;
    }

    /**
     * Try to add package if it fits.
     * @return true if package was added, false otherwise
     */
    public boolean tryLoadPackage(Package pkg) {
        lock.lock();
        try {
            if (usedCapacity + pkg.getSize() <= capacity) {
                packages.add(pkg);
                usedCapacity += pkg.getSize();
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    public boolean isFull() {
        lock.lock();
        try {
            return usedCapacity >= capacity;
        } finally {
            lock.unlock();
        }
    }

    public List<Package> getPackagesSnapshot() {
        lock.lock();
        try {
            return new ArrayList<>(packages);
        } finally {
            lock.unlock();
        }
    }
}

class Dock {
    private final int truckCapacity;
    private Truck currentTruck;
    private final Object lock = new Object();

    public Dock(int truckCapacity) {
        this.truckCapacity = truckCapacity;
        this.currentTruck = new Truck(truckCapacity);
    }

    /**
     * Loads a package onto a truck.
     * Returns true if package loaded, false if package doesn't fit on current truck.
     */
    public boolean load(Package pkg) {
        Truck truckToUse;
        boolean loaded;
        synchronized (lock) {
            loaded = currentTruck.tryLoadPackage(pkg);
            if (!loaded) {
                // Dispatch full truck asynchronously
                dispatchTruck(currentTruck);
                // Prepare a new truck
                currentTruck = new Truck(truckCapacity);
                // Try to load again
                loaded = currentTruck.tryLoadPackage(pkg);
            }
            truckToUse = currentTruck;
        }
        return loaded;
    }

    /**
     * Dispatch truck asynchronously (e.g., send to transport).
     * In production, this might be offloaded to a separate thread pool.
     */
    private void dispatchTruck(Truck truck) {
        // For demo, just print asynchronously
        new Thread(() -> {
            List<Package> packages = truck.getPackagesSnapshot();
            System.out.println("Dispatching truck with " + packages.size() + " packages.");
            // Simulate dispatch delay, etc.
        }).start();
    }
}

class Worker implements Runnable {
    private final Dock dock;
    private final List<Package> packagesToLoad;

    public Worker(Dock dock, List<Package> pkgs) {
        this.dock = dock;
        this.packagesToLoad = pkgs;
    }

    @Override
    public void run() {
        for (Package pkg : packagesToLoad) {
            while (!dock.load(pkg)) {
                // Optional backoff or wait
                Thread.yield();
            }
        }
    }
}
```

### Explanation:

- The **Truck** class uses a `ReentrantLock` to guard its capacity and package list. This allows multiple trucks to be manipulated concurrently without a global dock lock.
- The **Dock** uses a lightweight `synchronized(lock)` block only to:  
  - Check and swap the current truck reference atomically.  
  - Dispatch the full truck asynchronously, outside the critical section.
- The **dispatchTruck** method runs asynchronously to avoid blocking workers.
- Workers try to load packages, retrying if the truck was full and a new truck is being created.
- This approach reduces the global synchronization contention by:  
  - Minimizing the critical section in Dock — just truck swapping instead of entire load operation.  
  - Moving dispatching out of the lock.  
  - Allowing concurrent truck operations if the design were extended for multiple docks or trucks.

## Key follow-up questions

1. **Q:** Why not make the entire load method synchronized on the Dock instance?  
   **A:** Doing so serializes all package loading because only one worker at a time can load, causing contention and scaling bottlenecks in high concurrency.

2. **Q:** How do you handle the case where a package is larger than the truck capacity?  
   **A:** That package can never fit — the method should detect this early and reject the package or route it differently. This is a design constraint to handle in validation.

3. **Q:** What if dispatching the truck takes a long time — could that impact loading?  
   **A:** By dispatching asynchronously outside the synchronized block, we prevent blocking other workers. A thread pool can be used for dispatching to control concurrency and resource usage.

4. **Q:** Could multiple trucks be loaded in parallel?  
   **A:** This design currently models a single dock with a single current truck for simplicity. If multiple trucks or docks are supported, then concurrency can be improved by assigning different trucks to different workers without locking globally.

5. **Q:** How do you ensure no race conditions between checking truck full state and loading?  
   **A:** The truck operations (`tryLoadPackage` and `isFull`) are protected by internal `ReentrantLock`, ensuring atomic state updates and preventing race conditions.

## Takeaways

- Avoid coarse-grained locking in concurrent systems; it hurts scalability.  
- Minimize lock scopes to only the critical shared state updates.  
- Move potentially slow or blocking operations outside of locks (e.g., dispatching).  
- Use thread-safe primitives (`ReentrantLock`, concurrent collections) to protect fine-grained state.  
- Design for edge cases: packages that don’t fit, trucks dispatch timing, and worker backoff.  
- Always reason about potential contention points and test under load/concurrency to identify bottlenecks.
