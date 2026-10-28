# 🎬 Two Users Clicked "Book Now" at the Same Millisecond — Both Got Seat 5A

> Automatically generated interview-preparation note.

## Original problem

I designed the entities — Theater, Screen, Show, Seat, Booking — and walked through the flow.

## Interview-ready answer

## Problem understanding

The problem involves designing a seat booking system for shows in a theater, where multiple users may try to book the same seat concurrently. The specific issue highlighted is that two users clicked "Book Now" at the exact same millisecond and both were allocated the same seat, leading to a double booking conflict. 

**Constraints and design goals:**

- Prevent double booking of the same seat for the same show.
- Handle concurrent requests safely, ensuring consistency.
- Maintain a clear domain model: Theater, Screen, Show, Seat, Booking.
- Provide a scalable, robust solution with clear transactional boundaries.
- Ensure performance does not degrade excessively under high concurrency.

## Interview answer

### Core design

1. **Domain Entities:**
   - **Theater**: Collection of screens.
   - **Screen**: Physical screen with seats.
   - **Show**: A movie/event at a specific time tied to a Screen.
   - **Seat**: Identified by row and number in the Screen.
   - **Booking**: Reservation of one or more seats for a Show by a user.

2. **Booking Flow:**

   - User selects a show and one or more seats.
   - Request to book those seats concurrently can occur.
   - When booking, the system validates if the requested seats are available.
   - If available, system marks seats as booked atomically.
   - Confirm booking or reject if seats unavailable.

3. **Concurrency Control:**

   The root cause of double booking is concurrent seat allocation without proper synchronization or atomic checks.

   - **Optimistic Locking**: Store a version or timestamp field on Seat or Booking record. On booking, try to update only if version unchanged.
   - **Pessimistic Locking / DB Transactions**: Use DB locks (`SELECT ... FOR UPDATE`) to lock seat rows during booking.
   - **Atomic Update**: Use a single atomic DB query that attempts to book the seat only if it is free.
   - **In-memory locking or distributed locks** (like Redis locks) can also work in distributed systems.

4. **Data Model Suggestions:**

   - Represent seat availability per show (seat-show mapping).
   - Do not permanently mark seats as booked on the Screen entity since seats are reused for different shows.
   - Maintain a `SeatAvailability` entity with composite keys (show_id, seat_id) and a booking status.

5. **Handling Failure Cases:**

   - If booking fails due to concurrency, inform the client immediately to reselect seats.
   - Use retries or graceful degradation to handle concurrent contention.

6. **Trade-offs:**

   - **Pessimistic locking**: Simple but can reduce concurrency.
   - **Optimistic locking**: High concurrency but requires retry logic.
   - **Distributed locks**: Useful for horizontally scaled systems but add complexity.

## Java implementation

Below is an idiomatic example demonstrating concurrency-safe seat booking using optimistic locking with JPA/Hibernate annotations. Assume a relational DB and JPA for persistence:

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "seat_availability", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"show_id", "seat_id"})
})
public class SeatAvailability {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "show_id", nullable = false)
    private Long showId;

    @Column(name = "seat_id", nullable = false)
    private Long seatId;

    @Version
    private Long version; // Optimistic locking version field

    @Enumerated(EnumType.STRING)
    private Status status;

    private LocalDateTime bookedAt;

    private Long bookingId; // Link to Booking entity

    public enum Status {
        AVAILABLE,
        BOOKED
    }

    // getters/setters omitted for brevity
}
```

**Booking Service Example:**

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import jakarta.persistence.EntityManager;
import jakarta.persistence.OptimisticLockException;

@Service
public class BookingService {

    private final EntityManager em;

    public BookingService(EntityManager em) {
        this.em = em;
    }

    @Transactional
    public boolean bookSeat(Long showId, Long seatId, Long bookingId) {
        try {
            SeatAvailability seatAvailability = em.createQuery(
                "SELECT s FROM SeatAvailability s WHERE s.showId = :showId AND s.seatId = :seatId", 
                SeatAvailability.class)
                .setParameter("showId", showId)
                .setParameter("seatId", seatId)
                .getSingleResult();

            if (seatAvailability.getStatus() == SeatAvailability.Status.AVAILABLE) {
                seatAvailability.setStatus(SeatAvailability.Status.BOOKED);
                seatAvailability.setBookedAt(LocalDateTime.now());
                seatAvailability.setBookingId(bookingId);
                em.merge(seatAvailability); // Will check optimistic lock via @Version

                return true; // Successfully booked
            } else {
                return false; // Already booked
            }
        } catch (OptimisticLockException e) {
            // Someone else booked concurrently
            return false;
        } catch (NoResultException e) {
            // Seat not found
            return false;
        }
    }
}
```

**Explanation:**

- The `SeatAvailability` entity models the booking state per seat per show.
- The `@Version` attribute enables optimistic locking; if two concurrent transactions try to modify the record, one will fail with `OptimisticLockException`.
- The `BookingService.bookSeat()` method wraps the booking inside a transaction, reads seat availability, checks status, and tries to book it. Conflicts cause transaction rollback and failure.

**Concurrency and edge cases:**

- On failure, the client should retry or choose alternate seats.
- The database must support transactions and optimistic locking.
- Careful error handling, logging, and user messaging are important.
- Blocking scopes should be minimal to reduce contention.

## Key follow-up questions

1. **Q: What alternatives to optimistic locking can we use to prevent double booking?**  
   A: Pessimistic locking (e.g., `SELECT ... FOR UPDATE`) can prevent concurrent transactions from acquiring locks simultaneously, though it reduces concurrency. We can also use atomic queries to update seat status only if available or use distributed locks if the system is distributed.

2. **Q: How would you handle booking multiple seats in a single transaction?**  
   A: Wrap the booking logic for all requested seats inside a single transaction. Check and update seat availability for each seat atomically. If any seat booking fails, roll back the entire transaction to avoid partial bookings.

3. **Q: How do you scale this solution for distributed, high throughput systems?**  
   A: Use distributed locking mechanisms like Redis-based locks to synchronize seat booking across instances. Also, caching seat availability for reads, partitioning shows/seats, and efficient DB indexing help improve performance.

4. **Q: How do you handle the case where a user holds a seat but never completes booking?**  
   A: Implement locking timeouts or seat reservations with expiration. Temporarily mark seats as “held” with a TTL, and release the hold if booking is not completed within a timeframe.

5. **Q: What schema or data modeling changes would help improve concurrency control?**  
   A: Splitting seat availability into a separate entity with clear show and seat relationships is crucial. Introducing status enums for seat states (AVAILABLE, HELD, BOOKED) and timestamp fields for expiration helps manage lifecycle more effectively.

## Takeaways

- **Concurrency in seat booking is a classic race condition problem** that requires transactional consistency.
- Optimistic locking with version fields is often the best trade-off for balancing concurrency and correctness.
- Designing the domain model to represent seat availability per show avoids state conflicts.
- Transactions and locking strategies must be clearly defined and tested.
- Handling partial failures, retries, and user feedback is essential for a good UX.
- Consider real-world issues like seat holding timeouts and distributed locking for scalable system design.
