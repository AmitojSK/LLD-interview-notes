# 🎟️ "Your BookMyShow Has No Seat Lock TTL — Every Abandoned Cart Kills a Seat Forever"

> Automatically generated interview-preparation note.

## Original problem

The Amazon SDE3 round where the candidate designed 10 entities but missed the 3 design patterns that make them strong

## Interview-ready answer

## Problem understanding

The problem is about designing a robust online ticket booking system (like BookMyShow) focusing on seat locking with Time-To-Live (TTL). The core challenge is to avoid permanent seat locking when users add tickets to cart but do not complete the purchase. Without a seat lock TTL, abandoned or expired carts hold seats indefinitely, reducing inventory availability and user experience.

Key constraints and design goals:
- Support multiple concurrent users selecting seats in real-time.
- Prevent overbooking or race conditions on seat assignment.
- Implement a seat lock mechanism with TTL to auto-release expired locks.
- Ensure high availability and low latency on seat querying and locking.
- Design entities and interactions to model users, events, seats, carts, orders, and locking.
- Use design patterns to simplify, modularize, and strengthen the system (e.g., TTL, Observer for updates, State for seat status).

## Interview answer

To design a ticket booking system preventing permanent seat locks due to abandoned carts, follow these key steps:

1. **Core Entities and Relationships**:
   - **Event**: Contains metadata and a collection of Seats.
   - **Seat**: Holds seat ID, status (AVAILABLE, LOCKED, BOOKED), and locker info.
   - **User**: Identifies customers.
   - **Cart**: Holds selected seats with timestamps.
   - **Order**: Confirmed booking after payment.
   - **SeatLock**: Represents a lock with TTL expiry.

2. **Seat Locking with TTL**:
   - When a user adds a seat to cart, the system *locks* the seat and starts a TTL timer (e.g., 10 minutes).
   - This lock sets the seat status to LOCKED, preventing others from selecting it.
   - If TTL expires or user removes seat, the lock is automatically released.
   - On successful booking, seat status moves to BOOKED.

3. **Design Patterns to Strengthen Design**:
   - **State Pattern**: Manage seat lifecycle transitions (AVAILABLE ⇄ LOCKED ⇄ BOOKED) encapsulated in seat state classes.
   - **Observer Pattern**: Notify UI and other components when seat status changes (to update availability in real-time).
   - **Strategy Pattern or Command Pattern**: Encapsulate different strategies for seat locking logic and TTL expiry handling for easier testing and extensibility.

4. **TTL Enforcement**:
   - Use a background scheduler or delayed queue to expire locks.
   - Alternatively, use a distributed caching system (like Redis) with various expiry strategies.
   - Upon TTL expiry, release seat automatically, reverting seat status to AVAILABLE.

5. **Concurrency Handling**:
   - Synchronize seat locking operations per seat to avoid race conditions.
   - Optimistic locking or database constraints can be used if persistent storage is involved.
   - Use atomic operations or transactions for seat status updates.

6. **Resilience and Scalability**:
   - Use distributed cache or in-memory store consistent across instances.
   - Design APIs for locking/unlocking seats, cart management.
   - Handle failure scenarios — network issues, service crashes — through idempotent operations and eventual consistency.

## Java implementation

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;

// Enums for seat status
enum SeatStatus {
    AVAILABLE, LOCKED, BOOKED
}

// Seat class with State pattern implementation
class Seat {
    private final String seatId;
    private volatile SeatStatus status;
    private volatile String lockedByUserId;
    private volatile Instant lockExpiryTime;

    public Seat(String seatId) {
        this.seatId = seatId;
        this.status = SeatStatus.AVAILABLE;
    }

    public synchronized boolean lock(String userId, long lockDurationSeconds) {
        if (status == SeatStatus.AVAILABLE) {
            this.status = SeatStatus.LOCKED;
            this.lockedByUserId = userId;
            this.lockExpiryTime = Instant.now().plusSeconds(lockDurationSeconds);
            return true;
        }
        return false;
    }

    public synchronized boolean unlock(String userId) {
        if (status == SeatStatus.LOCKED && Objects.equals(this.lockedByUserId, userId)) {
            this.status = SeatStatus.AVAILABLE;
            this.lockedByUserId = null;
            this.lockExpiryTime = null;
            return true;
        }
        return false;
    }

    public synchronized boolean book(String userId) {
        if (status == SeatStatus.LOCKED && Objects.equals(this.lockedByUserId, userId)) {
            this.status = SeatStatus.BOOKED;
            this.lockedByUserId = null;
            this.lockExpiryTime = null;
            return true;
        }
        return false;
    }

    public synchronized boolean isLockExpired() {
        if (status == SeatStatus.LOCKED && Instant.now().isAfter(lockExpiryTime)) {
            return true;
        }
        return false;
    }

    public String getSeatId() {
        return seatId;
    }

    public synchronized SeatStatus getStatus() {
        // Auto unlock if lock expired
        if (status == SeatStatus.LOCKED && isLockExpired()) {
            status = SeatStatus.AVAILABLE;
            lockedByUserId = null;
            lockExpiryTime = null;
        }
        return status;
    }
}

// Scheduler to clean expired seat locks
class SeatLockExpiryTask implements Runnable {
    private final Collection<Seat> seats;

    public SeatLockExpiryTask(Collection<Seat> seats) {
        this.seats = seats;
    }

    @Override
    public void run() {
        for (Seat seat : seats) {
            synchronized (seat) {
                if (seat.isLockExpired()) {
                    seat.unlock(seat.lockedByUserId); // Unlock regardless of who holds it
                    System.out.println("Lock expired for seat " + seat.getSeatId());
                }
            }
        }
    }
}

// BookingSystem class
class BookingSystem {
    private final Map<String, Seat> seats;
    private final ScheduledExecutorService scheduler;

    private static final long LOCK_TTL_SECONDS = 10 * 60; // 10 minutes

    public BookingSystem(Collection<String> seatIds) {
        seats = new ConcurrentHashMap<>();
        for (String id : seatIds) {
            seats.put(id, new Seat(id));
        }
        scheduler = Executors.newSingleThreadScheduledExecutor();
        // Run periodic scan to unlock expired seats every 1 minute
        scheduler.scheduleAtFixedRate(new SeatLockExpiryTask(seats.values()), 1, 1, TimeUnit.MINUTES);
    }

    public boolean lockSeat(String seatId, String userId) {
        Seat seat = seats.get(seatId);
        if (seat == null) return false;
        return seat.lock(userId, LOCK_TTL_SECONDS);
    }

    public boolean unlockSeat(String seatId, String userId) {
        Seat seat = seats.get(seatId);
        if (seat == null) return false;
        return seat.unlock(userId);
    }

    public boolean bookSeat(String seatId, String userId) {
        Seat seat = seats.get(seatId);
        if (seat == null) return false;
        return seat.book(userId);
    }

    public SeatStatus getSeatStatus(String seatId) {
        Seat seat = seats.get(seatId);
        if (seat == null) return null;
        return seat.getStatus();
    }

    public void shutdown() {
        scheduler.shutdownNow();
    }
}
```

**Explanation:**

- Each seat tracks its status and lock owner and expiry.
- Lock acquisition is synchronized per seat to avoid race conditions.
- A scheduled task periodically scans all seats and releases expired locks.
- Booking a seat confirms the lock and marks the seat as BOOKED.
- Unlock explicitly frees a locked seat before TTL.

**Failure Modes & Edge Cases:**
- If the server crashes, in-memory locks are lost; a persistent backing store or distributed lock system would address this.
- Lock expiry precision depends on scheduler frequency.
- Concurrent requests for the same seat may fail for some users due to atomic locking.

## Key follow-up questions

1. **Question:** How would you handle the seat locking in a distributed system with multiple servers?
   **Answer:** Use a distributed locking mechanism (e.g. Redis with SETNX and TTL) to atomically lock seats with expiration. This ensures single seat ownership across multiple service instances. Alternatively, store seat lock state in a consistent distributed datastore with optimistic concurrency control.

2. **Question:** How can you notify users in real-time about seat availability changes?
   **Answer:** Employ WebSockets or Server-Sent Events (SSE) to push seat status updates. The system triggers notifications on the Observer pattern whenever seat state changes. This keeps all clients in sync instantly.

3. **Question:** How do you prevent malicious users from holding seats indefinitely?
   **Answer:** Implement strict TTL policies on locks and invalidate or auto-release seats after timeout. Limit the number of seats a single user can lock per event, and consider CAPTCHA or rate-limiting for abuse.

4. **Question:** What are alternative approaches to seat locking besides TTL?
   **Answer:** You could use optimistic concurrency with version checks at booking time, allowing multiple users to select the same seat but confirming availability at checkout. However, this risks checkout conflicts and poor UX. Another option is blocking payment upfront or holding the card authorization.

5. **Question:** How do you ensure ACID properties during the booking transaction?
   **Answer:** Use transactional storage and atomic state changes for seat booking operations. Transactions guarantee seat atomicity and consistency. Distributed transactions or sagas can be used for multi-service workflows with compensating actions.

## Takeaways

- Implementing seat locking with TTL is essential to prevent indefinite seat holding and lost availability.
- Proper synchronization and concurrency control on seat state is critical; per-seat locking granularity avoids contention.
- Design patterns like State and Observer improve modularity and maintain real-time updates.
- A background expiry mechanism or distributed TTL caches enforce lock expiry.
- Distributed systems require external coordination or caching to keep locks consistent.
- Failure modes include system crashes, concurrent race conditions, and malicious hold scenarios, all must be thoughtfully handled.
- Real-time seat availability visibility greatly improves user experience and reliability.
