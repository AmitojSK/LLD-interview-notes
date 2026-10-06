# ⏱️ A User Selects Seats, Goes to Make Tea, and Your Entire Show Loses 4 Premium Seats Forever

> Automatically generated interview-preparation note.

## Original problem

My design was solid. Theater → Screen → Show → SeatMap → Seat. I had ReentrantLock per seat.

## Interview-ready answer

## Problem understanding

The problem involves designing a seat reservation system for a theater show, where users select seats and hold them temporarily before confirming payment. The design must prevent race conditions and double-booking, particularly with concurrent users selecting the same seats. A key issue arises when a user selects seats but delays confirmation (e.g., goes to make tea), causing premium seats to remain locked and unavailable indefinitely, leading to poor utilization and customer frustration.

Constraints and goals:
- Seats are organized by Theater → Screen → Show → SeatMap → Seat.
- Concurrency control is needed to handle multiple users selecting seats concurrently.
- Locks per seat (e.g., ReentrantLock) ensure thread safety but may cause seats to be locked for too long if users delay.
- The system should ensure fairness and availability, preventing indefinite seat locking.
- Must support seat "holds" with expiration or timeout to release locked seats if not confirmed.

## Interview answer

**Core design:**
- Model the domain hierarchy: Theater contains Screens, Screens contain Shows, Shows have SeatMaps, Seats belong to SeatMaps.
- Seats are the smallest unit, identified uniquely.
- Introduce the concept of a **Seat Hold** to represent a temporary reservation.
- When a user selects a seat, the system should:
  - Check if the seat is available (not sold or held).
  - Temporarily lock the seat with a hold, associating it with the user and a timestamp.
  - Prevent others from selecting the same seat during the hold.
- Holds have a configurable expiration timeout (e.g., 5 minutes).
  - If a hold expires without confirmation (payment), the seat is released back to available.
- Use ReentrantLocks or similar concurrency primitives **per show or seat block**, rather than per seat, to limit lock contention and complexity.
- Use a background job or scheduler to periodically clear expired holds.
- Confirming purchase releases the hold and marks the seat as sold.

**Trade-offs and reasoning:**
- Per-seat ReentrantLock is granular but has pitfalls:
  - If a user holds the lock and delays action indefinitely, other sessions block or cannot acquire the lock.
  - Deadlocks can occur if multiple seats are locked inconsistently.
- Using a logical "hold" state with timestamps allows for eventual expiration without relying on continuous locking.
- A distributed or shared cache system (like Redis) with TTL can handle seat holds to scale better.
- Communicating a hold timer in the UI prevents user frustration ("You have 4 minutes to complete booking").
- Releasing seats after a timeout increases seat availability and fairness.
- Locks should guard seat status updates, not lock duration while user is idle.

## Java implementation

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

public class SeatReservationSystem {

    public static class Seat {
        private final String seatId;
        // Possible states: AVAILABLE, HELD, SOLD
        private SeatState state = SeatState.AVAILABLE;
        private String heldByUserId;
        private Instant holdTimestamp;
        private final ReentrantLock lock = new ReentrantLock();

        public Seat(String seatId) {
            this.seatId = seatId;
        }

        public boolean tryHold(String userId) {
            lock.lock();
            try {
                if (state == SeatState.AVAILABLE) {
                    state = SeatState.HELD;
                    heldByUserId = userId;
                    holdTimestamp = Instant.now();
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        public boolean confirmHold(String userId) {
            lock.lock();
            try {
                if (state == SeatState.HELD && userId.equals(heldByUserId)) {
                    state = SeatState.SOLD;
                    heldByUserId = null;
                    holdTimestamp = null;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        public boolean releaseHoldIfExpired(long holdTimeoutMillis) {
            lock.lock();
            try {
                if (state == SeatState.HELD) {
                    Instant now = Instant.now();
                    if (holdTimestamp.plusMillis(holdTimeoutMillis).isBefore(now)) {
                        state = SeatState.AVAILABLE;
                        heldByUserId = null;
                        holdTimestamp = null;
                        return true;
                    }
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        public SeatState getState() {
            lock.lock();
            try {
                return state;
            } finally {
                lock.unlock();
            }
        }
    }

    enum SeatState {
        AVAILABLE,
        HELD,
        SOLD
    }

    public static class Show {
        private final String showId;
        private final Map<String, Seat> seats = new ConcurrentHashMap<>();
        private final long holdTimeoutMillis = 5 * 60 * 1000; // 5 minutes

        public Show(String showId, List<String> seatIds) {
            this.showId = showId;
            for (String id : seatIds) {
                seats.put(id, new Seat(id));
            }
        }

        public boolean holdSeats(List<String> seatIds, String userId) {
            // Try to hold all requested seats atomically
            // This implementation attempts all seats, returning false if any fail
            List<Seat> seatObjects = new ArrayList<>();
            for (String id : seatIds) {
                Seat seat = seats.get(id);
                if (seat == null) return false; // seat invalid
                seatObjects.add(seat);
            }

            // Sort seats to prevent deadlocks when locking multiple seats
            seatObjects.sort(Comparator.comparing(s -> s.seatId));
            
            // Lock all seats in order
            List<ReentrantLock> locks = new ArrayList<>();
            for (Seat seat : seatObjects) {
                seat.lock.lock();
                locks.add(seat.lock);
            }

            try {
                // Check all seats are AVAILABLE
                for (Seat seat : seatObjects) {
                    if (seat.state != SeatState.AVAILABLE) {
                        return false;  // One seat unavailable
                    }
                }
                // Hold all seats
                Instant now = Instant.now();
                for (Seat seat : seatObjects) {
                    seat.state = SeatState.HELD;
                    seat.heldByUserId = userId;
                    seat.holdTimestamp = now;
                }
                return true;
            } finally {
                // Unlock all
                for (ReentrantLock lock : locks) {
                    lock.unlock();
                }
            }
        }

        public boolean confirmSeats(List<String> seatIds, String userId) {
            for (String id : seatIds) {
                Seat seat = seats.get(id);
                if (seat == null) return false;
                if (!seat.confirmHold(userId)) {
                    return false;
                }
            }
            return true;
        }

        public void expireHolds() {
            for (Seat seat : seats.values()) {
                seat.releaseHoldIfExpired(holdTimeoutMillis);
            }
        }
    }

    // A background service could call show.expireHolds() at regular intervals
}
```

**Explanation:**

- Each `Seat` maintains its own lock guarding its state.
- `Show` manages a collection of seats.
- The `holdSeats` method attempts to hold multiple seats atomically to prevent partial holds. It locks all seats in a fixed order to prevent deadlocks.
- Holds expire after 5 minutes via `expireHolds`, which should be invoked periodically by a scheduled task.
- `confirmSeats` finalizes reservations.
- This design avoids indefinite locking by not holding locks during user think-time, instead using timestamps and periodic cleanup.

## Key follow-up questions

1. **Q:** How do you handle concurrency when multiple users try to hold the same seat simultaneously?  
   **A:** By locking the seat(s) when transitioning states and checking availability inside the lock, only one user can hold a seat at a time. Attempt to acquire locks in a consistent order when holding multiple seats to avoid deadlocks.

2. **Q:** What if a user never confirms their seat hold?  
   **A:** Holds have an expiration timeout after which they are automatically released. A background job periodically releases expired holds, freeing seats for others.

3. **Q:** Why not use a single global lock for all seats?  
   **A:** A global lock would create a performance bottleneck, preventing scalability and parallelism. Using finer-grained per-seat locks or per-show locks reduces contention.

4. **Q:** How would you scale this design if multiple instances of your service are running?  
   **A:** Use a distributed locking mechanism or an external cache (e.g., Redis with SETNX and TTL features) to manage holds centrally. Periodic expiration and atomic updates should be maintained at the shared state level.

5. **Q:** Could you improve user experience to prevent seats being locked unintentionally?  
   **A:** Show a countdown timer for the hold expiration in the UI, notify users before expiration, and allow explicit hold cancellation to release seats.

6. **Q:** What are risks of deadlocks in your design and how do you mitigate them?  
   **A:** Deadlocks can occur when multiple threads lock multiple seats in different orders. To prevent deadlocks, always lock seats in a globally consistent order (e.g., sorted seat ID order).

## Takeaways

- Avoid long-lived locks held during user idle time; instead use hold states with expiration.
- Use fine-grained locking coupled with atomic state transitions for thread safety.
- Always design for concurrency, with deadlock avoidance strategies in multi-lock scenarios.
- Expiration mechanisms and cleanup jobs are essential to restore availability of held resources.
- User experience considerations (timers, notifications) align system behavior with expectations.
- Distributed systems require coordination beyond in-memory locks—think about external consistent storage or locking.
