# 🎟️ "Your BookMyShow Has No Seat Lock TTL — Every Abandoned Cart Kills a Seat Forever"

> Automatically generated interview-preparation note.

## Original problem

The Amazon SDE3 round where the candidate designed 10 entities but missed the 3 design patterns that make them strong

## Interview-ready answer

## Problem understanding

The problem revolves around designing a system for seat booking (like BookMyShow) with a critical design flaw: **no seat lock Time-To-Live (TTL)**. This means that when users add seats to their cart but eventually abandon it (do not complete purchase or release seats), those seats remain locked indefinitely, causing unnecessary blocking and potential revenue loss.

Key challenges to address in the design:
- Avoid permanent locking of seats from abandoned carts.
- Efficiently track and expire seat locks when users do not complete booking.
- Handle concurrent seat selection requests safely.
- Maintain a consistent view of seat availability.

Additionally, an Amazon SDE3 interview scenario mentioned designing 10 entities but missing 3 design patterns that are essential for a robust design. This suggests that proper pattern usage is crucial for maintainability, extensibility, and correctness.

## Interview answer

A well-designed seat booking system should include the following concepts:

1. **Seat Lock with TTL**:  
   Implement a locking mechanism that holds a seat for a limited time once added to the cart. If the user does not confirm within that TTL, the lock is released automatically, freeing the seat.

2. **Entities & Design Patterns**:  
   - **Aggregate Root Pattern** to group related entities like `Show`, `Seat`, `Booking` to maintain consistency boundaries.  
   - **State Pattern** to represent the seat states (`Available`, `Locked`, `Booked`). This enables clear state transitions with encapsulated behavior.  
   - **Observer Pattern** can be used to notify clients or other services when seat states change (e.g., broadcasting seat release events).

3. **Concurrency Control**:  
   Use optimistic or pessimistic locking to prevent race conditions where multiple users try to lock the same seat concurrently.

4. **Expiration Mechanism**:  
   - Use a background job or a scheduled executor to check and release expired seat locks.  
   - Alternatively, use distributed caches or storage with TTL (e.g., Redis) to automate expiry.

5. **Data Model Suggestions**:  
   - Entities: `UserCart`, `SeatLock`, `Seat`, `Show`, `Booking`—each with clear responsibilities and lifecycle.  
   - Use value objects for seat identity.  

6. **Handling Failures**:  
   - Ensure idempotency of booking confirmation.  
   - Handle partial failures (network, service downtime) by persisting state transitions atomically.

## Java implementation

The core of the solution is the `SeatLock` with TTL and state management. Here's a simplified idiomatic Java example demonstrating key parts:

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;

enum SeatState {
    AVAILABLE, LOCKED, BOOKED
}

// Seat entity with state management using State Pattern
class Seat {
    private final String seatId;
    private SeatState state;
    private Instant lockExpiryTime;

    public Seat(String seatId) {
        this.seatId = seatId;
        this.state = SeatState.AVAILABLE;
    }

    public synchronized boolean lock(long ttlSeconds) {
        if (state == SeatState.AVAILABLE) {
            state = SeatState.LOCKED;
            lockExpiryTime = Instant.now().plusSeconds(ttlSeconds);
            return true;
        }
        return false;
    }

    public synchronized void unlock() {
        if (state == SeatState.LOCKED) {
            state = SeatState.AVAILABLE;
            lockExpiryTime = null;
        }
    }

    public synchronized boolean book() {
        if (state == SeatState.LOCKED) {
            state = SeatState.BOOKED;
            lockExpiryTime = null;
            return true;
        }
        return false;
    }

    public synchronized boolean isLockExpired() {
        return state == SeatState.LOCKED && Instant.now().isAfter(lockExpiryTime);
    }

    public synchronized SeatState getState() {
        return state;
    }

    public String getSeatId() {
        return seatId;
    }
}

// SeatManager to manage seat locks & expirations
class SeatManager {
    private final Map<String, Seat> seats = new ConcurrentHashMap<>();
    private final ScheduledExecutorService expiryService = Executors.newSingleThreadScheduledExecutor();

    public SeatManager(List<String> seatIds) {
        for (String id : seatIds) {
            seats.put(id, new Seat(id));
        }
        // Run expiry check every 5 seconds
        expiryService.scheduleAtFixedRate(this::releaseExpiredLocks, 5, 5, TimeUnit.SECONDS);
    }

    public boolean lockSeat(String seatId, long ttlSeconds) {
        Seat seat = seats.get(seatId);
        return seat != null && seat.lock(ttlSeconds);
    }

    public boolean bookSeat(String seatId) {
        Seat seat = seats.get(seatId);
        return seat != null && seat.book();
    }

    private void releaseExpiredLocks() {
        for (Seat seat : seats.values()) {
            if (seat.isLockExpired()) {
                seat.unlock();
                System.out.println("Lock expired and seat released: " + seat.getSeatId());
            }
        }
    }

    public void shutdown() {
        expiryService.shutdown();
    }
}

// Usage example
public class BookingSystemDemo {
    public static void main(String[] args) throws InterruptedException {
        SeatManager manager = new SeatManager(Arrays.asList("A1", "A2", "A3"));

        if (manager.lockSeat("A1", 10)) {
            System.out.println("Seat A1 locked for 10 seconds");
        }

        // Wait 12 seconds, lock should expire
        Thread.sleep(12000);

        // Try to lock again, should succeed if previous lock expired
        if (manager.lockSeat("A1", 10)) {
            System.out.println("Seat A1 locked again after expiry");
        }

        manager.shutdown();
    }
}
```

**Notes**:  
- The `Seat` class encapsulates state and transition behavior.  
- The `SeatManager` runs periodic expiration checks, simulating TTL enforcement.  
- In production, you might use distributed locking or persistent storage with TTL support instead of in-memory approaches.

## Key follow-up questions

- How would you handle concurrent requests to lock the same seat in a distributed environment?  
- How do you ensure consistency when multiple microservices interact with seats (e.g., payments, notifications)?  
- How would you handle partial failures during booking confirmation?  
- What design changes would you make if seats can be locked in bulk (multiple seats at once)?  
- Can caching techniques improve performance without sacrificing consistency?  
- How to ensure scalability when the number of shows and seats grows to millions?  
- How would you implement seat availability notifications to clients in real-time?

## Takeaways

- **Seat lock TTL is critical** to prevent indefinite seat blocking from abandoned carts, improving resource utilization and user experience.  
- The **State Pattern** cleanly models seat lifecycle, making state transitions explicit and manageable.  
- Using **scheduled expiration checks or TTL-based caches** automates lock release without manual intervention.  
- **Concurrency control strategies** must be considered carefully to avoid race conditions and inconsistent seat states.  
- Applying core design patterns (Aggregate Root, State, Observer) brings clarity, scalability, and maintainability to complex domain models.  
- Real-world booking systems require robust failure handling, idempotency, and thoughtful trade-offs between consistency, availability, and scalability.

This problem is a great example of combining domain-driven design and concurrency control with system design best practices.
