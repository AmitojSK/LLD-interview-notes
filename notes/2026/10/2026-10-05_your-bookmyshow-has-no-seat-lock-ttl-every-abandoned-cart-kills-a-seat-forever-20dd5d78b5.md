# 🎟️ "Your BookMyShow Has No Seat Lock TTL — Every Abandoned Cart Kills a Seat Forever"

> Automatically generated interview-preparation note.

## Original problem

The Amazon SDE3 round where the candidate designed 10 entities but missed the 3 design patterns that make them strong

## Interview-ready answer

## Problem understanding

The problem involves designing a robust seat reservation system, like in a ticket booking platform (e.g., BookMyShow), that prevents indefinite seat blocking due to abandoned carts. Specifically, when a user selects (locks) seats to purchase, these seats should not be reserved indefinitely if the user does not complete the transaction. A Time-To-Live (TTL) or expiration mechanism is critical to free locked seats back to availability after a timeout period. The design should handle concurrency, consistent state transitions, and scalability.

Key constraints and goals:
- Avoid permanent seat blocking from abandoned carts.
- Ensure strong consistency in seat allocation to prevent double booking.
- Implement TTL for seat locks so seats expire and are released automatically.
- Support high concurrency and handle failure modes like transaction rollback.
- Design is extensible and maintainable with clear boundaries and patterns.

## Interview answer

**Core Design:**

1. **Entities Design:**
   - `Seat`: Represents each seat with states: AVAILABLE, LOCKED, BOOKED.
   - `Lock`: Represents a temporary hold on a seat by a user, with expiration timestamp.
   - `Cart`: User's selected seats collection.
   - `Booking`: Confirmed purchase of seats.
   - `User`: The customer entity.
   - `Event` and `Venue`: Context for seats.

2. **Key Components:**
   - **SeatManager:** Handles seat availability and locking logic.
   - **LockService:** Manages locking and TTL expiration.
   - **BookingService:** Final seat booking and payment handling.
   - **ExpirationScheduler:** Background job to release expired locks.

3. **Design Patterns:**
   - **State Pattern:** To represent seat states cleanly and transitions (AVAILABLE -> LOCKED -> BOOKED).
   - **Observer Pattern:** For expiration of locks notifying system components to release seats.
   - **Command Pattern:** Encapsulate user commands (lock seat, book seat) supporting retries and rollbacks.

4. **Concurrency and Consistency:**
   - Use optimistic or pessimistic locking at the database level for seat entities to avoid double bookings.
   - Distributed locks or atomic operations in a caching system (like Redis) for performance.
   - Idempotent APIs to handle retries gracefully.

5. **TTL Implementation:**
   - Store lock expiration timestamps.
   - Use a scheduled job or reactive event-driven mechanisms (messaging queue with delayed messages) to free expired seats.
   - In-memory TTL-based caches could be leveraged, but persistent expiry tracking is essential for crash recovery.

6. **Failure Handling:**
   - If booking fails, release seat locks immediately.
   - On service restart, reconcile seat states and release expired locks.
   - Ensure eventual consistency between seat locks and seat booking state.

**Trade-offs:**

- Using pessimistic DB locks simplifies correctness but may reduce scalability.
- Using in-memory caches for locks is fast but risks inconsistency without persistent backup.
- Scheduled TTL jobs introduce delay in freeing seats but are simpler to implement at scale versus real-time expiration.

## Java implementation

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;

enum SeatState {
    AVAILABLE, LOCKED, BOOKED
}

class Seat {
    private final String seatId;
    private volatile SeatState state;
    private volatile Instant lockExpiry;
    private volatile String lockedByUserId;
    private final ReentrantLock lock = new ReentrantLock();

    Seat(String seatId) {
        this.seatId = seatId;
        this.state = SeatState.AVAILABLE;
    }

    // Attempt to lock seat for a user with TTL
    boolean tryLock(String userId, Instant expiryTime) {
        lock.lock();
        try {
            if (state == SeatState.AVAILABLE) {
                state = SeatState.LOCKED;
                lockedByUserId = userId;
                lockExpiry = expiryTime;
                return true;
            }
            // If already locked by same user and not expired, refresh expiry time
            if (state == SeatState.LOCKED && lockedByUserId.equals(userId) && Instant.now().isBefore(lockExpiry)) {
                lockExpiry = expiryTime;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    // Confirm booking, only allowed if locked by user and not expired
    boolean confirmBooking(String userId) {
        lock.lock();
        try {
            if (state == SeatState.LOCKED && lockedByUserId.equals(userId) && Instant.now().isBefore(lockExpiry)) {
                state = SeatState.BOOKED;
                lockedByUserId = null;
                lockExpiry = null;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    // Release seat lock if expired or requested by user owning the lock
    void releaseLockIfExpiredOrByUser(String userId) {
        lock.lock();
        try {
            if (state == SeatState.LOCKED) {
                if (Instant.now().isAfter(lockExpiry) || lockedByUserId.equals(userId)) {
                    state = SeatState.AVAILABLE;
                    lockedByUserId = null;
                    lockExpiry = null;
                }
            }
        } finally {
            lock.unlock();
        }
    }

    boolean isLockedAndExpired() {
        return state == SeatState.LOCKED && Instant.now().isAfter(lockExpiry);
    }

    public String getSeatId() {
        return seatId;
    }

    public SeatState getState() {
        return state;
    }
}

class SeatManager {
    private final Map<String, Seat> seats;
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    // TTL for seat locks in seconds
    private final long lockTtlSeconds;

    SeatManager(List<String> seatIds, long lockTtlSeconds) {
        this.lockTtlSeconds = lockTtlSeconds;
        seats = new ConcurrentHashMap<>();
        for (String seatId : seatIds) {
            seats.put(seatId, new Seat(seatId));
        }
        startExpiryTask();
    }

    // Try to lock multiple seats for a user atomically if all available
    boolean lockSeats(String userId, List<String> seatIds) {
        List<Seat> seatList = new ArrayList<>();
        Instant expiryTime = Instant.now().plusSeconds(lockTtlSeconds);

        // Basic approach to avoid deadlocks: lock seats in order of seatId
        List<String> sortedSeatIds = new ArrayList<>(seatIds);
        Collections.sort(sortedSeatIds);

        // Acquire locks on all seat objects (ReentrantLock at seat level) to avoid partial locking
        for (String seatId : sortedSeatIds) {
            Seat seat = seats.get(seatId);
            if (seat == null) return false; // seat does not exist
            seat.lock.lock();
            seatList.add(seat);
        }

        try {
            // Check availability
            for (Seat seat : seatList) {
                if (seat.getState() != SeatState.AVAILABLE) return false;
            }
            // Lock all seats
            for (Seat seat : seatList) {
                seat.tryLock(userId, expiryTime);
            }
            return true;
        } finally {
            // Release all locks
            for (Seat seat : seatList) {
                seat.lock.unlock();
            }
        }
    }

    boolean confirmBooking(String userId, List<String> seatIds) {
        // Confirm booking for all seats if locked by user and not expired
        List<Seat> seatList = new ArrayList<>();

        for (String seatId : seatIds) {
            Seat seat = seats.get(seatId);
            if (seat == null) return false;
            seat.lock.lock();
            seatList.add(seat);
        }

        try {
            // Verify all seats locked by user and not expired
            for (Seat seat : seatList) {
                if (!(seat.getState() == SeatState.LOCKED && userId.equals(seat.lockedByUserId) &&
                        Instant.now().isBefore(seat.lockExpiry))) {
                    return false;
                }
            }
            // Confirm booking
            for (Seat seat : seatList) {
                seat.confirmBooking(userId);
            }
            return true;
        } finally {
            for (Seat seat : seatList) {
                seat.lock.unlock();
            }
        }
    }

    void startExpiryTask() {
        scheduler.scheduleAtFixedRate(() -> {
            for (Seat seat : seats.values()) {
                seat.lock.lock();
                try {
                    if (seat.isLockedAndExpired()) {
                        seat.releaseLockIfExpiredOrByUser(null);
                    }
                } finally {
                    seat.lock.unlock();
                }
            }
        }, lockTtlSeconds, lockTtlSeconds, TimeUnit.SECONDS);
    }

    void shutdown() {
        scheduler.shutdownNow();
    }

    // For testing and monitoring
    Map<String, SeatState> getSeatStates() {
        Map<String, SeatState> states = new HashMap<>();
        for (Map.Entry<String, Seat> e : seats.entrySet()) {
            states.put(e.getKey(), e.getValue().getState());
        }
        return states;
    }
}
```

**Explanation:**

- `Seat` encapsulates state transitions with thread safety.
- Seat locking uses a per-seat `ReentrantLock` to avoid concurrent state corruption.
- The `SeatManager` manages multiple seats, locking seats atomically by acquiring locks in sorted order to prevent deadlocks.
- A scheduled expiry task checks for and releases expired seat locks periodically.
- TTL is parameterized and can be tuned for business needs.
- Edge cases like null seats or state changes during booking are handled by locking and state checks.

## Key follow-up questions

1. **Question:** How would you scale this design to support millions of seats and requests?
   
   **Answer:** Introduce sharding of seat data across multiple services or databases by venue or event. Use distributed locking mechanisms (like Redis Redlock) to manage seat locks instead of in-memory locks. Deploy lock expiry using distributed delay queues. Use CQRS to separate read and write models for scalability.

2. **Question:** How do you handle the case when the system crashes during the TTL expiration task?

   **Answer:** Persist seat lock states and their expiry timestamps in durable storage (e.g., database or distributed cache with persistence). On restart, run a recovery process that scans for expired seat locks and releases them. Ensure scheduled tasks persist state to allow seamless recovery.

3. **Question:** Why is locking seats in sorted order important before trying to lock?

   **Answer:** Locking in a consistent global order prevents deadlocks where concurrent threads lock seats in different orders and wait indefinitely on each other.

4. **Question:** Can you handle partial locking success if some seats are unavailable?

   **Answer:** The design prevents partial success by locking seats atomically or returning failure if any seat is unavailable. Otherwise, compensating transactions or rollback logic would be necessary to maintain consistency.

5. **Question:** How do you ensure the idempotency of lock or book operations?

   **Answer:** Operations check current state and lock ownership before transitions. For example, trying to lock a seat already locked by the same user refreshes TTL but doesn't change state. Booking only succeeds if the seat is locked by the user and TTL not expired, making retries safe.

## Takeaways

- Implementing a TTL for seat locks is crucial to prevent indefinite blocking and improve seat availability.
- Clear seat state management and safe concurrency control prevent race conditions and ensure data consistency.
- Use design patterns like State, Observer, and Command to modularize behavior and future-proof design.
- Scheduling expiration tasks and handling failure recovery ensures robustness.
- Scalability considerations often push designs from local locks to distributed locking mechanisms.
- Deadlocks can be avoided by consistent ordering of lock acquisition.
- Idempotency and proper validation at each step maintain API reliability under retries or distributed failures.
