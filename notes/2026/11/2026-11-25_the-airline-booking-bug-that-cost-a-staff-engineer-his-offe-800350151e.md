# The Airline Booking Bug That Cost a Staff Engineer His Offe

> Automatically generated interview-preparation note.

## Original problem

✈️ His Booking System Held a Global Lock While Building the Seat Heap

## Interview-ready answer

## Problem understanding
The scenario describes a critical bug in an airline booking system where a global lock was held during the construction of a data structure ("seat heap"), causing performance bottlenecks and potentially system unavailability. The problem is a classic concurrency design issue in a high-throughput, time-sensitive domain like airline seat booking, where locks need to be minimal or fine-grained to avoid blocking other operations such as concurrent bookings or cancellations. 

Constraints:
- The booking system must support concurrent seat allocation requests.
- Data integrity must be maintained (no double bookings, consistent seat availability).
- Performance is critical; holding global locks during heavy computation or data structure building is unacceptable.
- The system should be scalable, fault-tolerant, and responsive under load.

Design Goals:
- Minimize lock contention to maximize throughput.
- Use concurrent data structures or lock splitting.
- Implement consistent snapshot or read-copy-update patterns if needed to build complex data structures without blocking.

## Interview answer
To design a high-performance airline seat booking system without global locking during seat heap construction:

1. **Data Structure Choice:**  
   Instead of using a single global lock guarding the entire seat data, use fine-grained locks at the segment or row level, or leverage concurrent data structures like `ConcurrentHashMap` for seat statuses.

2. **Lock Minimization:**  
   Avoid holding locks while performing expensive operations such as building or sorting the "seat heap." If the heap represents seat priority or availability, maintain it incrementally or rebuild it asynchronously.

3. **Snapshot or Versioning:**  
   Use immutable snapshots or copy-on-write techniques for building the heap outside of any critical section. The heap can then be swapped atomically once ready.

4. **Thread-Safe Heap Building:**  
   Use a separate thread or service to build the heap without blocking booking requests. Readers can access the current immutable heap, while the builder prepares the next version.

5. **Optimistic Concurrency Control:**  
   Use atomic compare-and-swap or version checks to apply booking changes, reducing lock duration and retry instead of blocking.

6. **Trade-offs:**  
   - Using finer-grained locks or lock-free data structures increases complexity but improves scalability.
   - Immutable snapshots add memory overhead but permit non-blocking reads.
   - Asynchronous rebuilding introduces eventual consistency windows.

## Java implementation
Below is an example illustrating a thread-safe seat booking system that avoids global locks while building a seat heap for prioritizing available seats. The heap is maintained as an immutable snapshot that is updated asynchronously.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicReference;

public class AirlineBookingSystem {

    enum SeatStatus { AVAILABLE, BOOKED }

    static class Seat implements Comparable<Seat> {
        final String seatNumber;
        final int priority; // lower priority value = higher preference
        final SeatStatus status;

        Seat(String seatNumber, int priority, SeatStatus status) {
            this.seatNumber = seatNumber;
            this.priority = priority;
            this.status = status;
        }

        Seat withStatus(SeatStatus newStatus) {
            return new Seat(seatNumber, priority, newStatus);
        }

        @Override
        public int compareTo(Seat other) {
            return Integer.compare(this.priority, other.priority);
        }
    }

    // Thread-safe map for seat states
    private final ConcurrentHashMap<String, Seat> seats = new ConcurrentHashMap<>();

    // Atomic reference to an immutable heap snapshot
    private final AtomicReference<List<Seat>> seatHeapSnapshot = new AtomicReference<>(Collections.emptyList());

    // Executor for asynchronous heap rebuilding
    private final ScheduledExecutorService rebuildExecutor = Executors.newSingleThreadScheduledExecutor();

    public AirlineBookingSystem(List<Seat> initialSeats) {
        for (Seat seat : initialSeats) {
            seats.put(seat.seatNumber, seat);
        }
        rebuildSeatHeap();

        // Periodically rebuild heap asynchronously every 5 seconds
        rebuildExecutor.scheduleAtFixedRate(this::rebuildSeatHeap, 5, 5, TimeUnit.SECONDS);
    }

    // Non-blocking seat booking method
    public boolean bookSeat(String seatNumber) {
        return seats.computeIfPresent(seatNumber, (k, seat) -> {
            if (seat.status == SeatStatus.AVAILABLE) {
                return seat.withStatus(SeatStatus.BOOKED);
            } else {
                return seat; // already booked
            }
        }).status == SeatStatus.BOOKED;
    }

    // Get the current snapshot of the seat heap
    public List<Seat> getSeatHeapSnapshot() {
        return seatHeapSnapshot.get();
    }

    // Rebuild the heap snapshot asynchronously without locking booking operations
    private void rebuildSeatHeap() {
        // Build a heap of only available seats sorted by priority
        List<Seat> availableSeats = new ArrayList<>();
        for (Seat seat : seats.values()) {
            if (seat.status == SeatStatus.AVAILABLE) {
                availableSeats.add(seat);
            }
        }
        Collections.sort(availableSeats); // natural order defined by priority

        // Atomically update the snapshot
        seatHeapSnapshot.set(Collections.unmodifiableList(availableSeats));
    }

    public void shutdown() {
        rebuildExecutor.shutdown();
    }
}
```

### Explanation: 
- We use a `ConcurrentHashMap` to allow concurrent seat state updates without global locking.
- The seat heap is represented as an immutable list sorted by priority, stored in an `AtomicReference`, rebuilt asynchronously.
- Booking operations use `computeIfPresent` to atomically update seat status.
- The asynchronous rebuild prevents booking request blocking.
- This approach trades eventual consistency for concurrency — the booking system is always responsive, and booking state is consistent, but the seat priority snapshot may lag behind a few seconds.

## Key follow-up questions

1. **Q:** How do you ensure no double booking in a concurrent booking scenario?  
   **A:** Using `ConcurrentHashMap.computeIfPresent` or atomic compare-and-swap operations ensures seat status updates happen atomically, preventing double booking.

2. **Q:** What if the seat heap needs to be real-time accurate?  
   **A:** In that case, rebuilding the heap synchronously for each booking is possible but will degrade performance. Alternatively, fine-grained locking, or lock-free skip lists could be used.

3. **Q:** Can you explain the trade-off between locking and eventual consistency here?  
   **A:** Minimizing locking improves throughput and reduces blocking but means the heap snapshot might not reflect the latest states instantly — the system tolerates slight delay in data freshness.

4. **Q:** How would you handle failure in the asynchronous rebuild thread?  
   **A:** Use try-catch around rebuild logic, log failures, and implement backoff retries. The system remains functional since booking modifies the seat map independently.

5. **Q:** How scalable is this design for a global booking system?  
   **A:** This design is scalable within a single node. For global scalability, consider partitioning seat data by flight segments or regions and replicating state with consensus protocols.

## Takeaways
- Avoid holding global locks during expensive computations in concurrent systems.
- Use fine-grained concurrency control and atomic operations to maintain data integrity.
- Immutable snapshots and asynchronous updates provide non-blocking reads and updated views.
- Design trade-offs between consistency and availability must align with business requirements.
- Careful data structure and concurrency design is crucial in time-critical booking systems preventing catastrophic failures caused by locking bottlenecks.
