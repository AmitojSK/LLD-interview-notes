# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
The task is to design a system capable of managing limited-time promotional offers (e.g., a "500rs discount") targeted at users who engage with an online platform focused on Low-Level Design (LLD) education. The core requirements involve:

- Defining and applying promotional offers with specific constraints (such as limited availability and expiration).
- Tracking user eligibility and offer redemption to prevent misuse.
- Ensuring scalability and fault tolerance given potentially high user volume.
- Supporting notifications or reminders for users regarding limited-time offers.

Key constraints:

- The offer has a limited time window.
- The offer is limited in quantity (e.g., only a certain number of users can avail it).
- Users should only be able to redeem the offer once.
- The system should handle concurrency safely.

Design goals:

- High availability and consistency in offer redemption.
- Clear modeling of offers, users, and redemption states.
- Scalability to support growing user base.
- Extensible for multiple simultaneous offers.

## Interview answer
To design this promotional offer system, I would focus on three main components: Offer Management, User Eligibility and Redemption Tracking, and Notification.

1. **Offer Management:**

   - Model an `Offer` with attributes: `offerId`, `discountAmount`, `startTime`, `endTime`, `maxRedemptions`, and `redemptionsCount`.
   - Implement constraints for validating if the offer is active (time-based) and if the maximum number of redemptions is not exceeded.

2. **User Eligibility and Redemption Tracking:**

   - Track user redemptions via a `UserOfferStatus` entity, which records offer usage per user.
   - Ensure idempotency so that a user cannot redeem the same offer multiple times.
   - Use atomic operations or transactions to safely increment global redemption counts and track individual user redemptions under concurrent load.

3. **Notification System:**

   - Notify users about available offers and expiration deadlines.
   - Allow reminders before the offer expires.

**Trade-offs:**

- Using strong consistency guarantees (e.g., database transactions) ensures correctness but may limit scalability under very high load.
- For very high scale, eventual consistency with conflict detection can be considered but adds complexity.
- Simpler locking or optimistic concurrency controls could be employed depending on system needs.

## Java implementation

```java
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class OfferService {
    // Represents an offer with limited quantity and time validity
    static class Offer {
        final String offerId;
        final int discountAmountRs;
        final Instant startTime;
        final Instant endTime;
        final int maxRedemptions;
        final AtomicInteger redemptionsCount = new AtomicInteger(0);

        public Offer(String offerId, int discountAmountRs, Instant startTime, Instant endTime, int maxRedemptions) {
            this.offerId = offerId;
            this.discountAmountRs = discountAmountRs;
            this.startTime = startTime;
            this.endTime = endTime;
            this.maxRedemptions = maxRedemptions;
        }

        boolean isActive() {
            Instant now = Instant.now();
            return now.isAfter(startTime) && now.isBefore(endTime);
        }

        boolean canRedeem() {
            return redemptionsCount.get() < maxRedemptions;
        }

        boolean incrementRedemptions() {
            while (true) {
                int current = redemptionsCount.get();
                if (current >= maxRedemptions) {
                    return false;
                }
                if (redemptionsCount.compareAndSet(current, current + 1)) {
                    return true;
                }
            }
        }
    }

    // Tracks user's redemption status per offer
    private final Map<String, Offer> offers = new ConcurrentHashMap<>();
    private final Map<String, Map<String, Boolean>> userOfferRedemptions = new ConcurrentHashMap<>();

    // Add an offer to the system
    public void addOffer(Offer offer) {
        offers.put(offer.offerId, offer);
    }

    // User redeems an offer (returns true if successful)
    public boolean redeemOffer(String userId, String offerId) {
        Offer offer = offers.get(offerId);
        if (offer == null || !offer.isActive()) {
            return false; // offer invalid or expired
        }
        userOfferRedemptions.putIfAbsent(userId, new ConcurrentHashMap<>());
        Map<String, Boolean> userRedemptions = userOfferRedemptions.get(userId);

        // Idempotent check
        if (userRedemptions.getOrDefault(offerId, false)) {
            return false; // already redeemed
        }

        // Attempt to increment global redemption count atomically
        if (!offer.canRedeem()) {
            return false; // no more redemptions left
        }

        boolean incremented = offer.incrementRedemptions();
        if (!incremented) {
            return false; // failed concurrency check
        }

        // Mark user as redeemed
        userRedemptions.put(offerId, true);
        return true;
    }
}
```

**Explanation:**

- The `Offer` class models the offer logic including atomic redemptions count updates.
- `OfferService` manages offers and user redemptions.
- Concurrency is handled safely using `AtomicInteger` and concurrent data structures.
- The redemption method ensures a user cannot redeem the same offer twice and that the total usage count respects the limit.
- For production, persistence layers (databases), distributed locks, or queue-driven processing might be necessary for durability and scaling.

## Key follow-up questions

1. **Q:** How would you handle scalability when millions of users attempt to redeem the offer simultaneously?
   **A:** We'd move persistence into a distributed database with support for atomic increments and uniqueness constraints, use message queues for processing redemptions asynchronously, and add caching layers. Rate limiting and horizontal scaling of services would be critical.

2. **Q:** How to prevent race conditions where two users redeem the last available offer simultaneously?
   **A:** Using atomic operations (like `compareAndSet` on counters) or database-level transactions ensures only one succeeds. Distributed locks could be used if necessary.

3. **Q:** How would you extend the system to support multiple simultaneous offers and complex eligibility rules?
   **A:** Abstract the offer model to include eligibility predicates or rules, store offer-user relations, and implement a rules engine. The redemption logic would check these rules dynamically.

4. **Q:** What failure modes might affect this system and how would you mitigate them?
   **A:** Potential failures include lost updates, double redemptions, and clock drift causing incorrect time checks. Using reliable storage with ACID transactions, time synchronization (e.g., NTP), and idempotent operations can mitigate these.

5. **Q:** How can you notify users effectively about upcoming offer expirations?
   **A:** Integrate with a notification service or scheduler to send email or push notifications ahead of offer expiration, based on user activity logs.

## Takeaways

- Modeling limited-time, limited-quantity offers requires careful concurrency control to avoid over-redemption.
- Atomic operations and idempotent user tracking are crucial for correctness.
- Time-bound logic must be accurate and resilient to clock discrepancies.
- Scalability concerns demand distributed systems design and eventual consistency considerations.
- Extensibility can be achieved via abstraction and modular design, preparing for complex rules and multiple offer types.
