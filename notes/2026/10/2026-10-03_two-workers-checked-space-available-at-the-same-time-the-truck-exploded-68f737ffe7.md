# 🚛 Two Workers Checked "Space Available" at the Same Time — The Truck Exploded

> Automatically generated interview-preparation note.

## Original problem

I designed the entities — Dock, Truck, Package, TruckState — and walked through the loading flow. The interviewer said....

## Interview-ready answer

## Problem understanding

The problem concerns modeling and handling the concurrent process of loading packages onto trucks at a dock in a warehouse or logistics environment. Key entities include Dock, Truck, Package, and states such as TruckState (e.g., loading, loaded, departed). The critical constraint is ensuring data consistency when multiple workers check and update the "space available" on the same truck concurrently. Without proper synchronization, race conditions can occur, e.g., two workers see space available simultaneously and both load packages, leading to overloading the truck ("truck exploded").

Design goals:

- Model domain entities with clear state transitions.
- Prevent race conditions in loading decisions.
- Maintain consistency and integrity of truck capacity.
- Support concurrent workers safely.
- Provide clear, maintainable, and extensible design.

## Interview answer

**Core Design:**

1. **Entities:**

   - **Truck:** Has capacity (e.g., weight limit or volume), maintains current load state (total loaded packages, current weight), and a `TruckState` field to represent current phase (e.g., `READY`, `LOADING`, `LOADED`, `DEPARTED`).

   - **Package:** Has identifiable attributes such as weight, volume.

   - **Dock:** Coordinates trucks and workers, possibly holds queues of trucks waiting to be loaded.

2. **Concurrency control:**

   Prevent race conditions on truck capacity checking and loading by synchronizing access to relevant truck state.

   Possible approaches:

   - **Pessimistic locking:** Use synchronized blocks or explicit locks when accessing/modifying the truck's available space.

   - **Optimistic locking:** Use versioning and atomic compare-and-set updates; suitable for distributed systems but adds complexity.

3. **Loading process:**

   Worker attempts to load a package onto a truck:

   - Check if truck state is `LOADING`.

   - Acquire lock on truck or synchronize to check available space.

   - If enough space, add the package.

   - Update the loaded weight, packages list.

   - Release lock.

   This prevents multiple workers from simultaneously over-committing capacity.

4. **State transitions:**

   Ensure truck state transitions are well defined and atomic, e.g., only transition from `LOADING` to `LOADED` once loading is complete.

5. **Error handling:**

   If a worker tries to load a package exceeding space, return an appropriate error or retry mechanism.

**Trade-offs:**

- Using coarse-grained synchronization (e.g., one lock per Truck) is simpler and safer but reduces parallelism if many workers load the same truck.

- Fine-grained locks improve concurrency but increase complexity.

- Optimistic locking may improve performance in low-contention, but handling version conflicts requires retries.

- Immutable package data simplifies concurrency since packages themselves do not change.

## Java implementation

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.locks.ReentrantLock;

enum TruckState {
    READY,
    LOADING,
    LOADED,
    DEPARTED
}

class Package {
    private final String id;
    private final double weight;

    public Package(String id, double weight) {
        this.id = id;
        this.weight = weight;
    }

    public double getWeight() {
        return weight;
    }

    public String getId() {
        return id;
    }
}

class Truck {
    private final String id;
    private final double capacity;
    private double currentLoad;
    private TruckState state;
    private final List<Package> packages;
    private final ReentrantLock lock;

    public Truck(String id, double capacity) {
        this.id = id;
        this.capacity = capacity;
        this.currentLoad = 0;
        this.state = TruckState.READY;
        this.packages = new ArrayList<>();
        this.lock = new ReentrantLock();
    }

    public String getId() {
        return id;
    }

    public TruckState getState() {
        return state;
    }

    // Begin loading phase, only if truck is READY
    public boolean startLoading() {
        lock.lock();
        try {
            if (state == TruckState.READY) {
                state = TruckState.LOADING;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    // Adds package atomically if space available and truck is loading
    public boolean loadPackage(Package pkg) {
        lock.lock();
        try {
            if (state != TruckState.LOADING) {
                return false; // Cannot load if not in LOADING state
            }
            double newLoad = currentLoad + pkg.getWeight();
            if (newLoad <= capacity) {
                packages.add(pkg);
                currentLoad = newLoad;
                return true;
            } else {
                return false; // Not enough space
            }
        } finally {
            lock.unlock();
        }
    }

    // Mark truck as fully loaded
    public boolean finishLoading() {
        lock.lock();
        try {
            if (state == TruckState.LOADING) {
                state = TruckState.LOADED;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    // Returns unmodifiable copy of packages to prevent external mutation
    public List<Package> getPackages() {
        lock.lock();
        try {
            return Collections.unmodifiableList(new ArrayList<>(packages));
        } finally {
            lock.unlock();
        }
    }

    public double getAvailableCapacity() {
        lock.lock();
        try {
            return capacity - currentLoad;
        } finally {
            lock.unlock();
        }
    }
}

class Dock {
    // This class may manage multiple trucks and distribute workers, simplified here
    // For a real system, queues and worker coordination would be present
}
```

**Explanation:**

- The `Truck` class uses a `ReentrantLock` to synchronize access to mutable state such as `currentLoad`, `packages`, and `state`.

- All methods that inspect or mutate these fields acquire the lock to prevent race conditions.

- `loadPackage` both checks capacity and modifies package list atomically.

- State transitions are done under lock to ensure consistency.

- Packages themselves are immutable.

- The locking strategy prevents two workers simultaneously checking free space and loading packages that exceed capacity.

**Failure modes & edge cases:**

- If a worker tries to load a package when the truck is not in LOADING state, the operation is rejected.

- If space is insufficient, the load fails gracefully.

- Deadlock risk is minimized by always locking a single truck instance at a time and using reentrant locks properly.

- High concurrency might cause threads to wait longer at the lock, which could be optimized later if needed.

## Key follow-up questions

1. **Question:** How would you handle multiple trucks being loaded concurrently by many workers?

   **Answer:** Each truck has its own lock, allowing concurrent loading on different trucks. Workers synchronize only at the truck level. For managing trucks and workers, the Dock can assign workers to trucks dynamically. Further scalability could involve partitioning trucks among workers or using lock-free data structures.

2. **Question:** What if the loading process spans multiple machines or services?

   **Answer:** Distributed locking mechanisms such as Redis-based locks or database locks can synchronize truck state across machines. The system could use optimistic concurrency control with versions or timestamps saved in the database. Eventual consistency models could apply, but to avoid truck overloading, a strict locking mechanism is often needed.

3. **Question:** What are potential race conditions if no locking is used?

   **Answer:** Two workers can both check space availability concurrently and both decide there is space, then both load packages resulting in exceeding the truck capacity. This inconsistency may cause "truck explosion," unsafe loads, or operational errors.

4. **Question:** Can you optimize locking to reduce contention if many workers frequently try to load?

   **Answer:** One way is to reduce the lock span: only lock the minimal critical section (checking and updating load). Another is to batch loading requests or use lock-free atomic variables for tracking load if possible. Alternatively, a queueing system where workers get exclusive access by assignment or staging area may help.

5. **Question:** How would you handle partial loads or unloading during the loading phase?

   **Answer:** Design the truck state machine to allow intermediate states and model operations like unloading with proper synchronization. Each package operation requires lock protection, and state transitions must carefully consider such scenarios (e.g., revert to `LOADING` from `LOADED` if unloading occurs).

6. **Question:** How to persist truck loading info and ensure consistency after a server crash?

   **Answer:** Persist truck state and loaded packages atomically in durable storage (database) during each update or batch. Transactions ensure atomicity. On crash recovery, reload the last consistent state from the database. Using write-ahead logs or event sourcing also helps resilience.

## Takeaways

- Concurrency control is essential when multiple workers access and mutate shared state (truck capacity).

- Using fine-grained locks per entity (truck) balances safety and concurrency well.

- Clear state machines simplify lifecycle management and concurrency reasoning.

- Thoroughly consider failure modes and partial updates, especially in distributed or fault-prone environments.

- Immutable data structures (packages) reduce shared mutable state complexity.

- Designing APIs that prevent illegal state transitions and enforce business rules (like loading only when allowed) increase robustness.

- Discuss trade-offs between simplicity (coarse locks) and scalability (fine locks, optimistic concurrency, distributed locking).
