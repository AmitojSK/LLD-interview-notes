# The Airline Booking Bug That Cost a Staff Engineer His Offe

> Automatically generated interview-preparation note.

## Original problem

✈️ His Booking System Held a Global Lock While Building the Seat Heap

## Interview-ready answer

## Problem understanding
The discussed problem involves an airline booking system where a global lock is held while constructing a heap or priority structure representing available seats. This design likely causes performance bottlenecks and concurrency issues, especially in a highly concurrent, real-time environment like flight seat booking. The main constraints include ensuring correct seat allocation without race conditions while maximizing throughput and minimizing lock contention. The goal is to design a system that supports concurrent bookings efficiently and correctly without compromising data consistency.

## Interview answer
To design a scalable and concurrency-friendly airline seat booking system, avoid using a global lock for managing seat allocation. Instead, use fine-grained locking or lock-free data structures to reduce contention.

**Core components:**

1. **Seat Heap or Priority Queue:**
   - Based on priority (e.g., price, class, or preferred seating).
   - Should be thread-safe and support concurrent access.

2. **Concurrency Control:**
   - Avoid global locks that block all booking threads.
   - Use fine-grained locks or striped locks per flight, seat section, or row.
   - Or use concurrent data structures like `ConcurrentSkipListSet` or `PriorityBlockingQueue`.
   - Optionally, use optimistic concurrency control with retries.

3. **Booking Flow:**
   - Read the current seat availability snapshot.
   - Attempt to reserve a seat atomically.
   - Handle booking failures due to race conditions with retries.
   - Persist commitments asynchronously.

4. **Failure Modes:**
   - Contention leading to retries.
   - Partial updates or stale reads—avoid by transactional boundaries or atomic operations.

5. **Trade-offs:**
   - More complex concurrency control improves throughput but complicates coding and reasoning.
   - Lock-free structures reduce locking delays but impose complexity.
   - Simplicity with global lock is easier but not scalable for production with many users.

## Java implementation
```java
import java.util.Comparator;
import java.util.PriorityQueue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Thread-safe flight seat booking manager using fine-grained locking.
 */
public class FlightSeatBookingManager {

    // Represents a seat with priority (lower number means better seat)
    static class Seat {
        final String seatId;
        final int priority;
        volatile boolean booked = false;

        Seat(String seatId, int priority) {
            this.seatId = seatId;
            this.priority = priority;
        }
    }

    // Lock per flight to reduce contention
    private final ConcurrentHashMap<String, FlightSeats> flights = new ConcurrentHashMap<>();

    /**
     * Manages seats within a flight
     */
    static class FlightSeats {
        private final PriorityQueue<Seat> seatHeap;
        private final ReentrantLock lock = new ReentrantLock();

        FlightSeats() {
            this.seatHeap = new PriorityQueue<>(Comparator.comparingInt(s -> s.priority));
        }

        void addSeat(Seat seat) {
            lock.lock();
            try {
                seatHeap.add(seat);
            } finally {
                lock.unlock();
            }
        }

        /**
         * Attempts to book the best available seat atomically
         * @return booked seat or null if none available
         */
        Seat bookBestAvailableSeat() {
            lock.lock();
            try {
                while (!seatHeap.isEmpty()) {
                    Seat seat = seatHeap.poll();
                    if (!seat.booked) {
                        seat.booked = true;
                        return seat;
                    }
                }
                return null; // no seats available
            } finally {
                lock.unlock();
            }
        }
    }

    public void addFlight(String flightId) {
        flights.putIfAbsent(flightId, new FlightSeats());
    }

    public void addSeatToFlight(String flightId, String seatId, int priority) {
        FlightSeats flightSeats = flights.get(flightId);
        if (flightSeats == null) {
            throw new IllegalArgumentException("Flight not found");
        }
        flightSeats.addSeat(new Seat(seatId, priority));
    }

    /**
     * Book the best seat on a flight
     */
    public Seat bookSeat(String flightId) {
        FlightSeats flightSeats = flights.get(flightId);
        if (flightSeats == null) return null;
        return flightSeats.bookBestAvailableSeat();
    }
}
```

**Details:**
- Each flight has its own seat heap and lock — avoiding a global lock.
- `ReentrantLock` used for critical section to maintain heap integrity.
- Seats marked `booked` to avoid double booking.
- Returns null if no seats available.
- This approach supports concurrent bookings on different flights without global contention.

**Limitations & extensions:**
- Further concurrency can be improved by partitioning seats by rows or sections with separate locks.
- Use transactional or persistent storage layers to commit bookings.
- Support cancellation, refunds by updating `booked` status accordingly.
- Handle failures gracefully via retries or compensating actions.

## Key follow-up questions

1. **Q: Why is holding a global lock during seat allocation problematic?**  
   **A:** Because it serializes all booking requests even if they operate on different flights, causing poor scalability and high contention, leading to increased latency and potential timeouts in a high-concurrency environment.

2. **Q: How can the system be scaled further beyond flight-level locking?**  
   **A:** By introducing finer-grained locking or lock-free structures at the seat row, section, or individual seat level. Alternatively, employ concurrent data structures or use optimistic concurrency with conflict detection and retries.

3. **Q: What concurrency data structures from Java can help optimize seat booking?**  
   **A:** Classes like `ConcurrentSkipListSet`, `ConcurrentLinkedQueue`, or `PriorityBlockingQueue` can offer concurrent prioritized or ordered access with less locking overhead.

4. **Q: How do you handle race conditions where two threads try to book the same seat?**  
   **A:** Use atomic operations or locks to ensure only one thread can mark the seat as booked. The above implementation locks during seat selection and booking to prevent races.

5. **Q: What happens if the system crashes after reserving a seat but before persisting the booking?**  
   **A:** Requires transactional or durable persistence to ensure atomicity and consistency. Use a database transaction or two-phase commit to guarantee bookings are committed fully or rolled back.

## Takeaways
- Global locking severely limits scalability in concurrent booking systems.
- Fine-grained locking or lock-free concurrency improves throughput by reducing contention, especially in real-time systems.
- Priority queues managing seat availability need concurrency-safe access; mutable shared data structures require synchronization.
- Balance complexity and performance: introducing concurrency primitives improves scalability but complicates reasoning.
- Consider failure modes carefully; durable persistence and transactional boundaries are essential for correctness.
- Testing concurrency scenarios thoroughly prevents subtle race conditions and bugs that can cause critical failures in production systems.
