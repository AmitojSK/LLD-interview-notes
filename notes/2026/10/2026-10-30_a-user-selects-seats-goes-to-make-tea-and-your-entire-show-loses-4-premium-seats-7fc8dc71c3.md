# ⏱️ A User Selects Seats, Goes to Make Tea, and Your Entire Show Loses 4 Premium Seats Forever

> Automatically generated interview-preparation note.

## Original problem

My design was solid. Theater → Screen → Show → SeatMap → Seat. I had ReentrantLock per seat.

## Interview-ready answer

## Problem understanding

The problem is designing a seat reservation system for a theater booking platform, where users select seats for a show. The main constraints involve preventing double booking and handling concurrency robustly since multiple users can attempt to book the same seat simultaneously. The platform should prevent a scenario where reserved seats become unusable or locked indefinitely due to improper concurrency control or timeout handling.

Key design goals:
- Model entities: Theater, Screen, Show, SeatMap, Seat.
- Concurrency control: prevent multiple users from booking the same seat.
- Avoid deadlocks or indefinite locking.
- Efficient handling of reservation hold periods before payment confirmation.
- Support high request throughput and fast seat availability feedback to users.

## Interview answer

### Core design

1. **Domain Modeling**:
   - **Theater**: has one or more Screens.
   - **Screen**: represents a physical room with multiple Shows.
   - **Show**: a scheduled movie/performance with a SeatMap.
   - **SeatMap**: a collection of Seats for the Show.
   - **Seat**: individual seat with state (available, held, reserved).

2. **Concurrency Control**:
   - Avoid locking at the granularity of a single seat with a dedicated lock (e.g., ReentrantLock per seat). Holding locks for long times (e.g., user delay) will block other threads and reduce system throughput.
   - Instead, implement a system where seat reservation is a two-step process:
     - **Hold step**: when user selects seats, temporarily mark them as “held” with a timestamp. This should be a quick transactional update to a data store.
     - **Confirmation step**: hold expires after a timeout (e.g., 15 minutes) if user does not complete booking (payment).
   - Use optimistic concurrency control or atomic compare-and-swap (CAS) semantics to avoid race conditions.
   - Use background tasks or scheduled jobs to expire holds.

3. **Data Structures & Storage**:
   - Seats modeled with state enum: AVAILABLE, HELD, RESERVED.
   - Seat holds stored in a distributed cache or database with hold expiration metadata.
   - Using locks (e.g., Java's ReentrantLocks) in memory is risky in distributed systems and can cause permanent deadlocks or resource wastage.

4. **Failure Modes**:
   - User selects seats but abandons the session → seats remain held forever.
     - Handle by enforcing hold expiration.
   - Two users simultaneously try to hold the same seat → only one succeeds via CAS.
   - System crash or restart: persist holds in durable storage for crash recovery.

### Trade-offs and reasoning

- Using ReentrantLock per seat can cause seats to be locked indefinitely if a user pauses. Locks held in memory are unsuitable for a distributed environment.
- Optimistic concurrency with timestamped holds provides better scalability and user experience.
- Background task or TTL on holds ensures seats don’t remain blocked.
- The design with Theater → Screen → Show → SeatMap → Seat is good entity-wise but locking has to be reconsidered.
- Storage selection depends on scale: in-memory caches with TTL work well for fast expiration, backed by persistent DB for durability.

## Java implementation

Below is a simplified design illustrating hold and confirm logic with optimistic concurrency and expiration, avoiding long-lived locks:

```java
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class SeatReservationSystem {

    enum SeatState { AVAILABLE, HELD, RESERVED }

    static class Seat {
        private volatile SeatState state = SeatState.AVAILABLE;
        private volatile String holdUserId = null;
        private volatile Instant holdExpiry = null;

        public synchronized boolean holdSeat(String userId, Instant now, long holdDurationSeconds) {
            if(state == SeatState.AVAILABLE ||
               (state == SeatState.HELD && holdExpiry != null && holdExpiry.isBefore(now))) {
                // Can place hold (new or expired hold)
                state = SeatState.HELD;
                holdUserId = userId;
                holdExpiry = now.plusSeconds(holdDurationSeconds);
                return true;
            }
            return false;
        }

        public synchronized boolean confirmSeat(String userId, Instant now) {
            if(state == SeatState.HELD && holdUserId.equals(userId) && holdExpiry.isAfter(now)) {
                state = SeatState.RESERVED;
                holdExpiry = null;
                return true;
            }
            return false;
        }

        public synchronized void expireHoldIfNeeded(Instant now) {
            if(state == SeatState.HELD && holdExpiry != null && holdExpiry.isBefore(now)) {
                state = SeatState.AVAILABLE;
                holdUserId = null;
                holdExpiry = null;
            }
        }

        public SeatState getState() {
            return state;
        }
    }

    static class SeatMap {
        private final Map<String, Seat> seats = new ConcurrentHashMap<>();
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        public SeatMap(Map<String, Seat> seats) {
            this.seats.putAll(seats);
            // Schedule hold expiry cleanup every minute
            scheduler.scheduleAtFixedRate(() -> {
                Instant now = Instant.now();
                seats.values().forEach(seat -> seat.expireHoldIfNeeded(now));
            }, 1, 1, TimeUnit.MINUTES);
        }

        public boolean holdSeat(String seatId, String userId, long holdDurationSeconds) {
            Seat seat = seats.get(seatId);
            if(seat == null) return false;
            return seat.holdSeat(userId, Instant.now(), holdDurationSeconds);
        }

        public boolean confirmSeat(String seatId, String userId) {
            Seat seat = seats.get(seatId);
            if(seat == null) return false;
            return seat.confirmSeat(userId, Instant.now());
        }

        public SeatState getSeatState(String seatId) {
            Seat seat = seats.get(seatId);
            return seat != null ? seat.getState() : null;
        }

        public void shutdown() {
            scheduler.shutdown();
        }
    }

    // Example use-case, test seats etc. would be implemented in unit tests.
}
```

### Explanation
- `Seat` uses synchronized methods to avoid races per seat.
- Holds expire via scheduled task running every minute.
- Holds recorded with expiry timestamps; confirmation checks validity.
- No permanent locks; concurrency handled by synchronized blocks which are short-lived.
- Extensible to distributed caching and persistent store with comparable atomic updates.

## Key follow-up questions

1. **Q: Why not use ReentrantLocks for seats as initially proposed?**  
   A: Locks held during user think time cause indefinite blocking and poor scalability. Long-held locks waste resources and are unsafe in distributed systems where processes die or disconnect.

2. **Q: How do you handle seat hold expiry across servers in a distributed environment?**  
   A: Store hold info with timestamps in a distributed cache or database with TTL support, letting the store expire holds automatically or have a cleanup service.

3. **Q: What happens if the system crashes while seats are held?**  
   A: Persist holds to durable storage before confirming. On restart, recover hold states and their expiration times or clear them based on elapsed time.

4. **Q: How to handle concurrency when multiple users attempt to hold the same seat simultaneously?**  
   A: Use atomic compare-and-set or transactional updates that only succeed if seat is AVAILABLE; otherwise fail and notify user.

5. **Q: Can a user extend their hold duration?**  
   A: Yes, design can support extending hold by updating holdExpiry if the user requests and the seat is still held by them.

## Takeaways

- Avoid locking seats for long user think time; it causes seats to be lost indefinitely.
- Use hold-and-expire semantics with timestamps, either via scheduled expiry tasks or TTL caches.
- Model seat states clearly and use atomic operations to avoid race conditions.
- Consider distributed environment challenges, including failure recovery and hold expiration coordination.
- The domain model (Theater → Screen → Show → SeatMap → Seat) suits the design, but concurrency control must be decoupled from coarse-grained locks.
