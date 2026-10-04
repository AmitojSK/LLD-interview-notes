# 🎬 Two Users Clicked "Book Now" at the Same Millisecond — Both Got Seat 5A

> Automatically generated interview-preparation note.

## Original problem

I designed the entities — Theater, Screen, Show, Seat, Booking — and walked through the flow.

## Interview-ready answer

## Problem understanding

We need to design a robust seat booking system for a theater show where multiple users can simultaneously try to book the same seat, e.g., seat 5A. The core problem is to ensure **data consistency and correctness under concurrent access**. Specifically, preventing race conditions where two users clicking "Book Now" at the same millisecond both get confirmation for the same seat. Key design goals include:

- Prevent double booking of the same seat for the same show.
- Support high concurrency with minimal user wait and conflicts.
- Maintain system integrity and real-time correctness.
- Provide clear and immediate feedback to users on booking success/failure.

Constraints:
- Bookings are per show (so seats must be tracked per show).
- High concurrent requests are expected.
- The system must be scalable and fault tolerant.

## Interview answer

### Core design

1. **Domain model:**

- Theater: Contains multiple Screens.
- Screen: Has multiple Seats.
- Show: Represents a scheduled time for a movie on a screen, with its own set of Seats.
- Seat: Identified by a seat number (e.g., "5A"), associated with a Screen and show.
- Booking: Represents a finalized reservation of one or more seats for a user at a show.

2. **Booking flow:**

- User selects desired show and seat(s).
- System checks availability of seat(s) for the show.
- If available, system **atomically reserves and confirms** the seat(s).
- Booking is persisted; seat marked as booked for that show.

3. **Concurrency control approaches:**

- **Pessimistic locking:** When a user tries to book, acquire a lock on the seat record in DB to prevent others from booking simultaneously.
- **Optimistic locking:** Use version numbers or timestamps to detect conflicts at commit.
- **Atomic update queries:** Use database constraints and atomic update statements to prevent duplicate bookings.
- **Distributed locks:** For highly concurrent distributed systems, lock seats using a distributed lock manager or Redis locks.
- **In-memory caches with synchronization:** If caching seat availability, ensure correct atomic updates via synchronized code or atomic data structures.

4. **Recommended approach:**

- Implement **optimistic locking** on seat booking records using a version field or status.
- On booking attempt, check if seat is free.
- Execute an atomic update conditional on seat state or version.
- If conflict detected (someone booked simultaneously), reject the booking.
- Optional: Use transactions for consistency.

This balances performance and correctness, avoids long locks, and allows retries.

### Trade-offs

- Pessimistic locking avoids conflicts but may reduce throughput due to blocked requests.
- Optimistic locking scales better but requires retry logic on conflicts.
- Distributed locking adds complexity but benefits multi-instance deployments.
- Relying solely on application-level checks risks race conditions; DB constraints are mandatory.

### Failure modes

- Double booking if concurrency controls missing.
- Deadlocks if locking not properly managed.
- Inconsistent state if partial booking succeeds.
- Starvation for highly concurrent seats if retries not capped.

## Java implementation

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.ConcurrentHashMap;

public class SeatBookingService {

    // In-memory representation for simplicity; in real systems use DB with proper locking
    private final ConcurrentHashMap<String, Seat> seatMap = new ConcurrentHashMap<>();

    public SeatBookingService() {
        // Initialize seats for a show, e.g., seat "5A"
        seatMap.put("5A", new Seat("5A"));
    }

    // Attempts to book a seat by seat number
    public boolean bookSeat(String seatNumber) {
        Seat seat = seatMap.get(seatNumber);
        if (seat == null) {
            throw new IllegalArgumentException("Seat does not exist");
        }
        return seat.tryBook();
    }

    static class Seat {
        private final String seatNumber;

        // 0 = available, 1 = booked
        private final AtomicInteger state = new AtomicInteger(0);

        Seat(String seatNumber) {
            this.seatNumber = seatNumber;
        }

        // Atomic operation to book a seat if available
        public boolean tryBook() {
            return state.compareAndSet(0, 1);
        }

        public boolean isBooked() {
            return state.get() == 1;
        }
    }
}
```

### Explanation

- We use an `AtomicInteger` per seat to store seat status (0 = free, 1 = booked).
- `compareAndSet` atomically transitions seat to booked only if it was free.
- This guarantees only one thread can book a seat at a time.
- This approach models optimistic locking at the application level.
- In production, this logic would be implemented using database transactions and atomic update queries.
- Edge cases like releasing seats and timeout holds require additional logic.

## Key follow-up questions

1. **Q: How would you handle booking multiple seats in a single transaction atomically?**  
   A: Use a single DB transaction locking all requested seat records pessimistically or using atomic conditional updates. Alternatively, implement application-level retry if any seat booking fails. This avoids partial bookings.

2. **Q: What if users select seats but do not confirm immediately? How to prevent others from taking them?**  
   A: Implement seat "holds" or reservations with expiry timestamps. Once held, seats are temporarily blocked for that user. If timeout expires, holds release and seats become available again.

3. **Q: How to scale this booking system for many concurrent users globally?**  
   A: Use distributed locking mechanisms (e.g., Redis RedLock) or rely on atomic consistent database operations with sharding/partitioning. Cache read availability aggressively to reduce DB calls.

4. **Q: What data consistency guarantee does your system provide for concurrent booking attempts?**  
   A: The system provides **serializable consistency for seat booking**, ensuring no double booking by enforcing atomic updates and optimistic concurrency control.

5. **Q: How do you handle failure during booking persistence?**  
   A: Use transactional writes. If persistence fails, rollback seat booking state. Provide idempotent retry logic or compensation to maintain consistency.

## Takeaways

- Handling concurrent booking of unique resources (seats) demands strong concurrency control.
- Lower-level atomic operations or DB constraints are critical to prevent double booking.
- Optimistic locking paired with retries offers good performance under high concurrency.
- Real-world systems use transactions, distributed locks, and timeout-based holds.
- Edge cases include partial bookings, failed transactions, timeout holds, and scaling considerations.
- Always test concurrency with race condition scenarios in unit and integration tests.
