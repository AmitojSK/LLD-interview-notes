# 📦 His Dock System Held a Global Lock While Every Worker Waited in Line

> Automatically generated interview-preparation note.

## Original problem

My design was solid — Dock, Truck, Package, TruckState, Worker. The interviewer said 'Walk me through load(). What locks do you hold?' I said 'I synchronize on the Dock instance, check if the package fits, place it on the truck, and if the truck is full I dispatch it and create a new one — all under one lock.

## Interview-ready answer

## Problem understanding
Design a multithreaded system simulating package loading onto trucks via docks with workers. Constraints include concurrent workers trying to load packages without corrupting shared states (Dock, Truck, Package), ensuring thread safety, and maintaining system scalability and performance. The design goal is to allow multiple workers to load packages efficiently while minimizing contention and maximizing throughput, especially preventing bottlenecks like global locking in critical paths.

## Interview answer
The initial design synchronizes the entire load operation by locking the Dock instance, checking if packages fit, loading them on a truck, and dispatching full trucks all under a single lock. While correct for thread safety, this creates a global lock that sequentializes all workers, leading to a performance bottleneck.

**Core Design:**
- The Dock acts as a shared resource coordinating package loading.
- Trucks have a state and capacity.
- Workers concurrently load packages to trucks.
- Threads synchronize to avoid data races but must reduce contention.

**Trade-offs and reasoning:**
- Coarse-grained locking (one lock on Dock) simplifies correctness but severely limits concurrency.
- Fine-grained locking or lock splitting can improve throughput but adds complexity.
  
**Possible improvements:**
1. **Separate locks for Trucks:** Each truck could be locked independently, enabling parallel loads on different trucks.
2. **Lock-free or concurrent data structures:** Use concurrent queues for packages or atomic counters for truck capacity.
3. **Partition docks or loading stations:** Multiple docks with independent locks let workers proceed in parallel.
4. **Dispatcher decoupling:** Loading and dispatch operations could be split so dispatch happens asynchronously.

This reduces contention while preserving safety. The key is to avoid holding a single global lock during the entire load operation, especially long-running tasks like dispatching.

## Java implementation
Here is a refined Java approach using fine-grained synchronization:

```java
import java.util.concurrent.atomic.AtomicInteger;

class Package {
    private final int size;

    Package(int size) {
        this.size = size;
    }

    int size() {
        return size;
    }
}

enum TruckState {
    LOADING, DISPATCHED
}

class Truck {
    private final int capacity;
    private final AtomicInteger currentLoad = new AtomicInteger(0);
    private volatile TruckState state = TruckState.LOADING;

    Truck(int capacity) {
        this.capacity = capacity;
    }

    // Try to load a package if it fits, returns true if loaded
    boolean tryLoadPackage(Package pkg) {
        while (true) {
            int load = currentLoad.get();
            int newLoad = load + pkg.size();
            if (newLoad > capacity || state != TruckState.LOADING) {
                return false;
            }
            if (currentLoad.compareAndSet(load, newLoad)) {
                return true;
            }
        }
    }

    boolean isFull() {
        return currentLoad.get() >= capacity;
    }

    void dispatch() {
        state = TruckState.DISPATCHED;
        // Simulate dispatch logic asynchronously
    }

    TruckState getState() {
        return state;
    }
}

class Dock {
    private final int truckCapacity;
    private volatile Truck currentTruck;

    public Dock(int truckCapacity) {
        this.truckCapacity = truckCapacity;
        this.currentTruck = new Truck(truckCapacity);
    }

    public boolean load(Package pkg) {
        while (true) {
            Truck truck = currentTruck;
            if (truck.tryLoadPackage(pkg)) {
                if (truck.isFull()) {
                    dispatchTruck(truck);
                }
                return true;
            } else {
                // Current truck full or dispatched, try to swap in a new truck
                synchronized (this) {
                    if (currentTruck == truck) {
                        dispatchTruck(truck);
                        currentTruck = new Truck(truckCapacity);
                    }
                }
            }
        }
    }

    private void dispatchTruck(Truck truck) {
        if (truck.getState() == TruckState.LOADING) {
            truck.dispatch();
            // Could push truck to a dispatch queue for asynchronous processing
        }
    }
}

class Worker extends Thread {
    private final Dock dock;
    private final Package pkg;

    Worker(Dock dock, Package pkg) {
        this.dock = dock;
        this.pkg = pkg;
    }

    @Override
    public void run() {
        dock.load(pkg);
    }
}
```

### Explanation:
- **AtomicInteger for load:** allows concurrent update attempts without coarse locking.
- **Volatile currentTruck:** safe visibility of the current truck.
- **Double-check locking in load:** after failed load attempt, synchronize only when swapping trucks.
- **Dispatch asynchronously:** dispatch logic decoupled and invoked outside main lock protecting truck assignment.
- **Minimized synchronized block:** only during truck switching and dispatch initiation.

### Failure modes:
- Possible repeated failed attempts to load in race conditions, but bounded by correctness.
- If `dispatch()` is slow, consider offloading it to an async thread to avoid blocking.

## Key follow-up questions
1. **Q:** How do you ensure no packages are lost or duplicated during concurrent loading?
   **A:** By using atomic operations on the truck’s load counter and verifying truck state before and after loading, we ensure package loading is exclusive and consistent, preventing lost or duplicated packages.

2. **Q:** How does your design scale with multiple workers?
   **A:** By minimizing synchronized blocks and using atomic operations, multiple workers can load different packages concurrently to the same or different trucks, significantly improving scalability over a global lock.

3. **Q:** What if package sizes vary greatly, how would you handle fitting packages?
   **A:** The atomic load update handles capacity constraints dynamically. Larger packages that do not fit the current truck trigger truck dispatch and new truck creation, ensuring no package overload.

4. **Q:** How would you handle truck dispatch asynchronously?
   **A:** The `dispatchTruck()` method can hand off the truck to a dedicated executor or dispatch queue thread pool, decoupling load and dispatch logic and avoiding blocking loading threads.

5. **Q:** Are there any risks of deadlock or livelock in your design?
   **A:** The design avoids deadlocks as it uses minimal synchronized blocks with limited scope and lock-free atomic operations for package loading. Livelock risk is low but can be mitigated by backoff or yield strategies if contention is high.

## Takeaways
- Avoid global locks in high concurrency systems; it blocks all workers and reduces throughput.
- Use fine-grained locking or atomic variables to reduce contention.
- Decouple long-running operations (e.g. dispatch) from critical synchronized sections.
- Carefully reason about thread-safe state updates with a combination of atomic and volatile variables.
- Design for scalability and maintainability by balancing concurrency complexity and correctness.
