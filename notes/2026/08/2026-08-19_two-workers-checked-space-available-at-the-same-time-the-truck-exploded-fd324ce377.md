# 🚛 Two Workers Checked "Space Available" at the Same Time — The Truck Exploded

> Automatically generated interview-preparation note.

## Original problem

I designed the entities — Dock, Truck, Package, TruckState — and walked through the loading flow. The interviewer said....

## Interview-ready answer

## Problem understanding

This scenario describes a concurrency problem in a logistics system where two workers simultaneously check if there is space available on a truck, and then proceed to load packages. The race condition here is that both observe space availability, proceed to load, and this overbooking causes an error—likened to a "truck explosion."

Key domain concepts likely involved:

- **Dock**: Location where trucks are loaded.
- **Truck**: Has a capacity and a current load state.
- **Package**: Item to be loaded onto a truck.
- **TruckState**: Tracks current load, available space, and loading status.

The interviewer’s concern is probably about managing concurrent access and updates to the truck state to prevent inconsistent reads/updates—an archetypal concurrency control problem in backend systems.

## Interview answer

This is a classic example of a race condition due to concurrent state checks and updates. To solve it, your design must:

- Prevent multiple workers from simultaneously loading packages that exceed truck capacity.
- Ensure atomic check-and-update of truck space availability.
- Maintain consistent TruckState under concurrency.

Potential solutions include:

1. **Synchronized/blocking critical section**

   Use Java synchronization or locks around checking space and loading the package to ensure only one worker performs the operation at a time. This would serialize access to the truck state, preventing race conditions.

2. **Optimistic locking with versioning**

   Use a version number or timestamp with TruckState and employ compare-and-swap semantics. Workers read the state and version, perform their logic, then during update check if version has changed. On conflict, retry the operation.

3. **Database locks / transactions**

   If the truck state is stored in DB, leveraging row-level locking or transactional isolation ensures that only one transaction can load packages that fit in the capacity at once.

4. **Use atomic operations**

   Use atomic integer/counter representing available capacity and employ atomic decrement operation that checks for capacity overflow before decrementing.

*Tradeoffs:*

- Synchronization is simplest and easy to reason about but blocks threads.
- Optimistic locking scales well under low contention but involves retries.
- Database transactions are often the most reliable but can have performance overhead.
- Atomic operations are good if you only manage numeric capacity and simple decrement checks.

**In summary**, your design needs to encapsulate the check-and-load operation atomically so that two workers do not produce inconsistent state that leads to overloading.

## Java implementation

Below is a sample idiomatic Java implementation utilizing synchronization to protect the critical section of checking space and loading a package atomically.

```java
public class Truck {
    private final int capacity;
    private int loadedPackagesCount;

    public Truck(int capacity) {
        this.capacity = capacity;
        this.loadedPackagesCount = 0;
    }

    // Synchronized method to ensure atomic check and load operation
    public synchronized boolean loadPackage(Package pkg) {
        if (hasSpace()) {
            loadedPackagesCount++;
            // Additional logic to associate package with truck omitted for brevity
            return true; // Successfully loaded
        }
        return false; // No space available
    }

    public synchronized boolean hasSpace() {
        return loadedPackagesCount < capacity;
    }

    public synchronized int getAvailableSpace() {
        return capacity - loadedPackagesCount;
    }
}

public class Package {
    private final String id;

    public Package(String id) {
        this.id = id;
    }

    public String getId() {
        return id;
    }
}
```

**Usage in multithreaded worker context:**

```java
Truck truck = new Truck(10);

Runnable worker = () -> {
    Package pkg = new Package("pkg_" + Thread.currentThread().getId());
    if (truck.loadPackage(pkg)) {
        System.out.println("Loaded package " + pkg.getId());
    } else {
        System.out.println("No space available to load package " + pkg.getId());
    }
};

Thread worker1 = new Thread(worker);
Thread worker2 = new Thread(worker1);
worker1.start();
worker2.start();
```

This guarantees only one worker at a time checks and increments `loadedPackagesCount`, preventing concurrent overbooking.

## Key follow-up questions

- How would you handle scenarios where packages have different sizes, not just counts?
- How would you design the system for distributed concurrency (e.g., multiple app servers)?
- What are the trade-offs between optimistic vs pessimistic locking here?
- How can you avoid deadlocks or reduced throughput due to locking?
- How would you scale the system for hundreds of concurrent workers?
- How to handle failure scenarios, such as a worker failing mid-loading after reserving space?
- Can you extend the design to support reservation and confirmation steps?

## Takeaways

- Concurrency control is crucial in systems managing shared mutable state under concurrent access.
- The atomicity of check-and-update operations prevents race conditions and inconsistent states.
- Synchronized blocks or locks are simplest to implement but may have performance impacts.
- Optimistic locking offers better throughput under low contention but requires retries.
- Real-world systems often rely on transactional databases or distributed locking mechanisms.
- Detailed requirements (package size, distributed environment, failure recovery) influence concurrency strategies.
- Always design entities and workflows to expose clearly atomic operations for critical state changes.
