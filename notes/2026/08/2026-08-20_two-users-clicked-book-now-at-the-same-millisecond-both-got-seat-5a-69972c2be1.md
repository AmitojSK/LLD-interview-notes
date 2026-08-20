# 🎬 Two Users Clicked "Book Now" at the Same Millisecond — Both Got Seat 5A

> Automatically generated interview-preparation note.

## Original problem

I designed the entities — Theater, Screen, Show, Seat, Booking — and walked through the flow.

## Interview-ready answer

## Problem understanding

The problem describes a concurrency issue in a movie ticket booking system: two users managed to book the same seat (e.g., seat 5A) at the exact same millisecond. This results in double booking, which is a critical data integrity problem in a real-time booking system. The problem involves ensuring consistent booking of seats with proper synchronization and isolation, avoiding race conditions.

Entities mentioned include:

- Theater: Multiple screens.
- Screen: A physical screen or hall within a theater.
- Show: A scheduled movie screening on a specific screen at a specific time.
- Seat: A physical seat identifier within a Screen.
- Booking: The transactional record of a user reserving a seat for a Show.

The key challenge is enforcing that only one booking transaction can allocate a seat for a particular Show at a time, even under high concurrency.

---

## Interview answer

To handle concurrency and prevent multiple users from booking the same seat simultaneously, you need to ensure **atomicity** of seat reservation. Several strategies can be considered:

1. **Database locking at the booking level:**

   - Use database transactions with locks on seat status rows.
   - For example, pessimistic locking (`SELECT ... FOR UPDATE`) on the seat record to prevent concurrent modification.
   - Or optimistic locking with version numbers; if the record was modified after selection, abort and retry.

2. **Unique constraints:**

   - Have a unique index/constraint on (`show_id`, `seat_id`) in the booking table to prevent duplicate bookings.
   - If two transactions try to book the same seat concurrently, one will fail at commit time due to the constraint violation.

3. **Application-level synchronization:**

   - Use a synchronized booking method or distributed locks (e.g., Redis Redlock) keyed by `show_id` + `seat_id`.
   - Ensures single-threaded booking attempts for that seat at any moment.
   - Requires handling lock expiry and deadlocks carefully.

4. **Seat availability state management:**

   - Maintain a seat availability map with atomic operations.
   - For example, use an `AtomicBoolean` or atomic update flags to mark seat as booked.
   - This is effective in in-memory local caches but harder in distributed systems.

A robust approach combines database constraints with locking mechanisms to avoid race conditions and handle unexpected failures.

**Booking flow sketch:**

1. User selects seat.
2. Backend initiates transaction.
3. Lock seat record for update or check optimistic version.
4. Verify seat availability.
5. Insert booking entry with unique constraint.
6. Commit transaction.
7. Return booking success, or retry/fail if seat unavailable.

This solution intrinsically depends on ACID guarantees of the database and proper isolation levels (e.g., SERIALIZABLE or REPEATABLE_READ with locking hints).

---

## Java implementation

Below is a simplified Java example demonstrating pessimistic locking with JPA/Hibernate and transactions in a Spring Boot backend:

```java
@Entity
public class Seat {
    @Id
    private String id; // e.g., "5A"
    
    @ManyToOne
    private Show show;

    private boolean booked;

    // getters, setters
}

@Entity
@Table(
  uniqueConstraints = @UniqueConstraint(columnNames = {"show_id", "seat_id"})
)
public class Booking {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    private Show show;

    @ManyToOne
    private Seat seat;

    private String userId;

    private LocalDateTime timestamp;

    // getters, setters
}

@Repository
public interface SeatRepository extends JpaRepository<Seat, String> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from Seat s where s.show = :show and s.id = :seatId")
    Optional<Seat> findSeatForUpdate(@Param("show") Show show, @Param("seatId") String seatId);
}

@Service
public class BookingService {

    @Autowired
    private SeatRepository seatRepository;

    @Autowired
    private BookingRepository bookingRepository;

    @Transactional
    public Booking bookSeat(String userId, Show show, String seatId) {
        Seat seat = seatRepository.findSeatForUpdate(show, seatId)
            .orElseThrow(() -> new IllegalArgumentException("Seat not found"));

        if (seat.isBooked()) {
            throw new IllegalStateException("Seat already booked");
        }

        seat.setBooked(true);
        seatRepository.save(seat); // optional if JPA tracks changes

        Booking booking = new Booking();
        booking.setUserId(userId);
        booking.setShow(show);
        booking.setSeat(seat);
        booking.setTimestamp(LocalDateTime.now());

        return bookingRepository.save(booking);
    }
}
```

### Explanation:

- `@Lock(LockModeType.PESSIMISTIC_WRITE)` ensures a row lock on the seat, preventing concurrent transactions from reading/modifying it simultaneously.
- The entire booking is within a transaction so that changes are atomic.
- If two threads call `bookSeat` concurrently for the same seat, one gets the lock first, the other blocks or fails.
- On commit, seat is marked booked and booking record created.

---

## Key follow-up questions

- How do database isolation levels influence this booking scenario?
- How would you handle booking cancellations or seat releases?
- What are pros and cons of optimistic vs pessimistic locking here?
- How to scale the booking system for a huge number of concurrent requests?
- How to handle failures or partial transaction rollbacks?
- How to design the data model to minimize locking while preventing conflicts?
- Could a distributed lock or a message queue be helpful? What are the tradeoffs?

---

## Takeaways

- Seat booking requires strong concurrency guarantees to prevent double bookings.
- Database locks and unique constraints are critical to maintain data integrity.
- Pessimistic locking prevents race conditions but may cause contention under high load.
- Optimistic locking allows more concurrency but requires retry logic.
- Transactions must be used to keep booking steps atomic.
- In distributed systems, additional synchronization mechanisms might be needed.
- The data model and database schema design greatly influence concurrency handling complexity and performance.

This design challenge reflects typical issues in real-world booking systems like airline or cinema seat reservations. Understanding isolation, locking, and transaction management is key to solving it efficiently.
