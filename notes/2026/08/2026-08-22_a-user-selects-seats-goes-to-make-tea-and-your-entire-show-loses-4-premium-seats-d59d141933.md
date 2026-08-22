# ⏱️ A User Selects Seats, Goes to Make Tea, and Your Entire Show Loses 4 Premium Seats Forever

> Automatically generated interview-preparation note.

## Original problem

My design was solid. Theater → Screen → Show → SeatMap → Seat. I had ReentrantLock per seat.

## Interview-ready answer

## Problem understanding

The scenario describes an online theater seat reservation system where users select seats for a show, but while the user delays finalizing the purchase (e.g., taking a break), those seats remain locked and unavailable to others, effectively losing premium seats "forever" during high contention. The user’s initial design model is:

- **Theater** → contains Screens
- **Screen** → contains Shows
- **Show** → has a SeatMap for that specific showtime
- **SeatMap** → manages Seats
- **Seat** → individual seat entity, locked by a `ReentrantLock`

Each seat uses a `ReentrantLock` to provide concurrency control for seat booking at an individual seat level.

The problem is related to how concurrency and locking are implemented for seat reservations, leading to resource starvation when locks are held for too long without timeout or expiration.

## Interview answer

The main issue with using a `ReentrantLock` per seat as a locking mechanism for seat reservations is that it only synchronizes concurrent access but does not define any timeout or expiration semantics on the lock holding. If a customer selects seats and then delays their purchase, these locks remain held, blocking other users from booking the same seats. This leads to unavailable seats and a poor customer experience.

### Analysis:

- **Locks hold on seats indefinitely:** When a user locks seats to reserve, those locks remain as long as the user session or operation holds them.
- **No automatic release or expiration:** Without a timeout/expiry, these locks are effectively permanent until explicitly released.
- **Scale and granularity:** Locking per seat is fine for fine-grained concurrency but requires additional logic for session or reservation timeouts.
- **Locking primitives alone are insufficient:** Using plain `ReentrantLock` (or similar) is a concurrency tool, but not a higher-level reservation management strategy.

### Better design approach:

- Use a **seat reservation state machine** with explicit seat states such as:  
  - **AVAILABLE** (free),  
  - **HELD** (temporarily reserved but not sold),  
  - **SOLD** (booked and paid).
  
- When a user selects seats, the system marks these seats as **HELD** and starts a timer/lease/timestamp.
- Implement a **lease expiration mechanism** that automatically releases (reverts to AVAILABLE) seats held for more than a configured timeout (e.g., 5-10 minutes).
- The locking mechanism should protect concurrent state transitions but not act as a "lock forever" semaphore.
- Consider centralizing locking or using optimistic concurrency with atomic updates (e.g., database transactions or versioning) rather than per-seat locks.
- Provide users with clear feedback about the remaining hold time.
- Use atomic compare-and-set operations or transactional updates to handle seat state changes safely.

Such a design prevents seats from being locked indefinitely and increases fairness and responsiveness.

## Java implementation

Below is a simplified implementation example illustrating the **Seat** entity with state and expiry management. This example abstracts locking with `synchronized` on seat state transitions and uses a scheduled mechanism to expire seat holds.

```java
import java.time.Instant;
import java.util.concurrent.*;

public class Seat {
    public enum State {
        AVAILABLE, HELD, SOLD
    }

    private final String seatId;
    private State state = State.AVAILABLE;
    private Instant holdExpiryTime; // when the hold expires
    private String holdingUserId;

    private static final long HOLD_DURATION_SECONDS = 300; // 5 minutes

    public Seat(String seatId) {
        this.seatId = seatId;
    }

    public synchronized boolean hold(String userId) {
        if (state != State.AVAILABLE) {
            return false;
        }
        state = State.HELD;
        holdingUserId = userId;
        holdExpiryTime = Instant.now().plusSeconds(HOLD_DURATION_SECONDS);
        return true;
    }

    public synchronized boolean confirmSale(String userId) {
        if (state == State.HELD && userId.equals(holdingUserId)) {
            state = State.SOLD;
            holdingUserId = null;
            holdExpiryTime = null;
            return true;
        }
        return false;
    }

    public synchronized boolean releaseHold(String userId) {
        if (state == State.HELD && userId.equals(holdingUserId)) {
            state = State.AVAILABLE;
            holdingUserId = null;
            holdExpiryTime = null;
            return true;
        }
        return false;
    }

    // Method to check if hold expired and clear it
    public synchronized void checkAndExpireHold() {
        if (state == State.HELD && Instant.now().isAfter(holdExpiryTime)) {
            state = State.AVAILABLE;
            holdingUserId = null;
            holdExpiryTime = null;
        }
    }

    public String getSeatId() {
        return seatId;
    }

    public synchronized State getState() {
        checkAndExpireHold(); // update state before returning
        return state;
    }
}

// Manager class to run hold expirations periodically
class SeatExpiryManager {
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private final ConcurrentMap<String, Seat> seats;

    public SeatExpiryManager(ConcurrentMap<String, Seat> seats) {
        this.seats = seats;
    }

    public void startExpiryTask() {
        scheduler.scheduleAtFixedRate(() -> {
            for (Seat seat : seats.values()) {
                seat.checkAndExpireHold();
            }
        }, 1, 1, TimeUnit.MINUTES);
    }

    public void shutdown() {
        scheduler.shutdown();
    }
}
```

### Key implementation details:

- Seat state tracked atomically inside synchronized blocks to ensure concurrency safety.
- `hold()` method lets a seat be reserved temporarily, only if it’s currently AVAILABLE.
- `confirmSale()` finalizes the booking.
- `releaseHold()` cancels the reservation voluntarily.
- A timed expiration automatically frees seats if the hold is not confirmed within timeout.
- A periodic task scans seats to expire any timed-out holds.

This contrasts with using a `ReentrantLock` per seat where locks may be held indefinitely.

## Key follow-up questions

- How do you ensure fairness among concurrent users competing for the same seats?
- How would this design change if the seat inventory is updated in a distributed system?
- How do you handle partial failures, e.g., user closes browser in the middle of seat holding?
- What data storage model fits best: relational DB with transactions, in-memory distributed cache, or event-based state machine?
- How would you implement refunds or seat re-availability if payment fails?
- What API design would you use for client seat selection and confirmation?
- How do you monitor and alert for hot or blocked seats or system performance?

## Takeaways

- Fine-grained locking like per-seat `ReentrantLock` handles basic concurrency but does not solve seat hold expiration or reservation life-cycle.
- A robust seat reservation system needs explicit state management with time-based holds and automatic expiration.
- Combining concurrency controls with business logic state machines provides better resilience.
- Timeouts and expiry prevent resource starvation and improve user experience under contention.
- Centralized or distributed coordination must consider latency, partial failure, and consistency trade-offs.
- Thoughtful design is required to balance consistency, performance, and user fairness in seat booking systems.
