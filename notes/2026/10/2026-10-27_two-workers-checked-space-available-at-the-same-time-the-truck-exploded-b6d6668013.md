# 🚛 Two Workers Checked "Space Available" at the Same Time — The Truck Exploded

> Automatically generated interview-preparation note.

## Original problem

I designed the entities — Dock, Truck, Package, TruckState — and walked through the loading flow. The interviewer said....

## Interview-ready answer

## Problem understanding

The scenario involves multiple workers simultaneously checking and updating the "space available" on a truck during a loading process. The entities involved include Dock, Truck, Package, and TruckState. The key problem is to design a system that allows multiple workers (threads) to load packages onto the truck concurrently without causing inconsistent states such as overbooking the truck's capacity, leading to a "truck exploded" scenario (data inconsistency or resource over-allocation).

**Constraints & Design Goals:**
- Correctly maintain and update truck space available to avoid exceeding capacity.
- Support concurrent access by multiple workers (threads).
- Prevent race conditions leading to incorrect state (e.g., two workers see enough space independently and both proceed, cumulatively exceeding capacity).
- Ensure strong consistency of the Truck's state.
- Maintain scalability and reasonable concurrency level.
- Handle failure or rollback scenarios gracefully, if needed.

## Interview answer

**Core design approach:**

1. **Entities:**
   - `Truck` holds capacity and current utilization.
   - `Package` holds size.
   - `Dock` manages trucks and worker interaction.
   - `TruckState` encapsulates current capacity info.

2. **Concurrency considerations:**
   - The main challenge is concurrent modification of the available space.
   - Naive approach (check-then-update without synchronization) leads to race conditions.
   
3. **Synchronization mechanisms:**
   - Use **locks** around the critical section of checking and updating truck space.
   - Options:  
     a) Use `synchronized` methods/blocks in Java.  
     b) Use a `ReentrantLock` for finer control.
   
4. **Atomic state update:**
   - Combine the "check available space" and "reserve space" into one atomic operation inside the lock.
   
5. **Data structure choice:**
   - Truck's available space can be a simple integer.
   - Maintain two fields: `capacity` and `usedSpace`.
   
6. **Failure modes:**
   - What if the reserve operation fails mid-way? Must ensure no partial updates.
   - Consider transactional or rollback mechanism if extended.
   
7. **Performance trade-off:**
   - Lock contention could be a bottleneck for very high concurrency.
   - Migration to optimistic concurrency or compare-and-swap (CAS) logic might be possible, but more complex.
   
8. **Sample flow:**
   - Worker thread requests to load a package of size N.
   - Acquire truck lock.
   - Check `capacity - usedSpace >= N`.
   - If yes, update `usedSpace += N`.
   - Release lock.
   - Return success or failure to worker.

## Java implementation

```java
public class Truck {
    private final int capacity;
    private int usedSpace;
    private final Lock lock = new ReentrantLock();

    public Truck(int capacity) {
        this.capacity = capacity;
        this.usedSpace = 0;
    }

    /**
     * Attempts to load a package of given size atomically.
     * @param packageSize Size of package to load.
     * @return true if package could be loaded, false if insufficient space.
     */
    public boolean tryLoadPackage(int packageSize) {
        lock.lock();
        try {
            if (packageSize <= 0) return false;
            if (capacity - usedSpace >= packageSize) {
                usedSpace += packageSize;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    /**
     * For test/inspection purposes only.
     */
    public int getAvailableSpace() {
        lock.lock();
        try {
            return capacity - usedSpace;
        } finally {
            lock.unlock();
        }
    }
}
```

### Explanation:
- Using a `ReentrantLock` to guard the critical section.
- The `tryLoadPackage` method first locks, checks space, updates space, then releases the lock.
- This ensures atomicity and consistency.
- `getAvailableSpace` is also protected to provide consistent snapshots.
- Packages smaller than or equal to zero rejected immediately.
- Potential extension: unlock early or use optimistic locking for read-mostly scenarios.

## Key follow-up questions

1. **Q: What could happen if you don't synchronize the space check and update?**  
   A: Multiple threads could read the same available space simultaneously and both approve loading packages, exceeding capacity and corrupting the logical state.

2. **Q: How would you scale this if multiple trucks exist and many workers are loading concurrently?**  
   A: Use fine-grained locking per truck instance. Workers loading different trucks can proceed concurrently, and only contention is on individual trucks.

3. **Q: How does this design handle rollback if a package loading fails halfway?**  
   A: Currently, the load operation is atomic and synchronous. For complex transactional scenarios, you'd need compensating actions or a two-phase commit mechanism. Alternatively, make state updates only on confirmed successful package placement.

4. **Q: Could you use atomic variables instead of locks?**  
   A: Possibly. An `AtomicInteger` for `usedSpace` with CAS (compare-and-set) can replace locks if operations are simple increments with checks. But the check-then-update atomicity is tricky and may require loop retry logic.

5. **Q: How would you prevent starvation or deadlocks in this design?**  
   A: Deadlocks are unlikely as we only acquire one lock per operation. For starvation, `ReentrantLock` supports fairness policies (fair lock) at the cost of throughput.

## Takeaways

- Always combine read and write operations that depend on each other into atomic critical sections to avoid race conditions.
- Use appropriate synchronization primitives (`synchronized`, `ReentrantLock`, or `Atomic*`) based on complexity and concurrency needs.
- Design for fine-grained locks to maximize concurrency across independent resources.
- Understand trade-offs between locking overhead and complexity of optimistic concurrency.
- Consider failure and rollback strategies for real-world package or resource allocations.
- Testing concurrent modifications is essential to catch race conditions early.
