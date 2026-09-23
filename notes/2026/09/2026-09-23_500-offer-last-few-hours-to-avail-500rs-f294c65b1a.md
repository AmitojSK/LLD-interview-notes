# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
The task involves designing a backend system component to handle promotional offers in an e-commerce or learning platform context. Key requirements include managing limited-time offers (like a 500 INR discount), ensuring users can avail the offer only within a specified timeframe, and perhaps limiting the number of redemptions per user or globally. Constraints often include preventing duplicated redemptions, handling concurrency during offer validation and redemption, and ensuring data consistency and scalability.

Design goals:
- Represent offers with validity period and rules.
- Allow users to query available offers.
- Safely redeem an offer, enforcing constraints (time, validity, availability).
- Handle concurrency to avoid race conditions or overselling.
- Provide audit or logs for offer usage.
- Scalable and maintainable design.

## Interview answer
To design a "Limited-Time Offer" feature:

1. **Core components**:
   - **Offer**: Represents the promotion with attributes like ID, description, discount amount, start and end times, max redemptions (global/user), and current redemption count.
   - **UserOfferRedemption**: Tracks redemptions to ensure per-user limits.
   - **OfferService**: Business logic to validate and redeem offers.
   - **OfferRepository**: Data access layer that persists offers and redemptions.

2. **Flow**:
   - When a user requests available offers, query active offers (current time within the validity window).
   - When redeeming:
     - Check if offer is active by timestamp.
     - Check if user is eligible (not exceeded per-user limit).
     - Check if global redemption limit is not exceeded.
     - Atomically increment redemption counters and mark the redemption.
   - Return success or failure accordingly.

3. **Concurrency and consistency**:
   - Use optimistic locking or database transactions to prevent race conditions.
   - For high concurrency, consider distributed locks or atomic counters (e.g., Redis INCR with Lua scripts).
   - Use caching to reduce DB load but always confirm availability on redemption.

4. **Edge cases**:
   - Redemption near expiry — ensure atomic check and update.
   - Failures during redemption (rollback counters).
   - Multiple concurrent redemptions by same user.

5. **Trade-offs**:
   - Strong consistency with transaction locking vs higher throughput with eventual consistency.
   - Simplicity in design assuming low concurrency vs scalability for millions of users.
   - Data schema complexity vs flexibility for future offer types.

## Java implementation
Below is a simplified implementation focusing on the core offer redemption flow with concurrency control using `synchronized` blocks for thread-safety at the service level. In production, this should be managed via DB transactions or distributed locks.

```java
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

class Offer {
    private final String offerId;
    private final String description;
    private final int discountRs;
    private final Instant startTime;
    private final Instant endTime;
    private final int maxGlobalRedemptions;
    private final int maxPerUserRedemptions;

    private int currentGlobalRedemptions = 0;

    public Offer(String offerId, String description, int discountRs,
                 Instant startTime, Instant endTime,
                 int maxGlobalRedemptions, int maxPerUserRedemptions) {
        this.offerId = offerId;
        this.description = description;
        this.discountRs = discountRs;
        this.startTime = startTime;
        this.endTime = endTime;
        this.maxGlobalRedemptions = maxGlobalRedemptions;
        this.maxPerUserRedemptions = maxPerUserRedemptions;
    }

    public synchronized boolean canRedeem() {
        Instant now = Instant.now();
        return now.isAfter(startTime) && now.isBefore(endTime)
            && currentGlobalRedemptions < maxGlobalRedemptions;
    }

    public synchronized boolean redeem() {
        if (!canRedeem()) return false;
        currentGlobalRedemptions++;
        return true;
    }

    public int getDiscountRs() {
        return discountRs;
    }

    public int getMaxPerUserRedemptions() {
        return maxPerUserRedemptions;
    }
}

class OfferService {
    private final Map<String, Offer> offers = new ConcurrentHashMap<>();
    private final Map<String, Map<String, Integer>> userRedemptions = new ConcurrentHashMap<>();
    // Map<OfferId, Map<UserId, Count>>

    public void addOffer(Offer offer) {
        offers.put(offer.offerId, offer);
    }

    /**
     * Attempt to redeem an offer for a user.
     * @param offerId Offer id
     * @param userId User id
     * @return discount amount if success, 0 otherwise
     */
    public synchronized int redeemOffer(String offerId, String userId) {
        Offer offer = offers.get(offerId);
        if (offer == null) return 0;

        if (!offer.canRedeem()) return 0;

        Map<String, Integer> userMap = userRedemptions.computeIfAbsent(offerId, k -> new ConcurrentHashMap<>());
        int redeemedCount = userMap.getOrDefault(userId, 0);
        if (redeemedCount >= offer.getMaxPerUserRedemptions()) {
            return 0; // user max redemption exceeded
        }

        // Attempt to redeem global count
        if (!offer.redeem()) {
            return 0; // global limit reached at redemption
        }

        // Update user redemption count
        userMap.put(userId, redeemedCount + 1);
        return offer.getDiscountRs();
    }
}
```

### Explanation:
- `Offer` encapsulates offer data and controls global redemption state with synchronization.
- `OfferService` manages offers and per-user redemption counts.
- The `redeemOffer` method synchronizes to maintain integrity between global and per-user counts.
- The offer validity is checked by time constraints and redemption limits.
- Concurrency control is basic and suitable for single JVM. For distributed systems, replace with DB transactions or distributed locks.

## Key follow-up questions
1. **Q:** How would you handle concurrency in a distributed environment to prevent overselling offers?  
   **A:** Use distributed locks (e.g., Redis RedLock), database atomic transactions with row locking, or atomic counters with CAS operations in highly scalable stores. Another option is to enqueue redemption requests and process them serially in a worker to ensure strict limits.

2. **Q:** How do you manage offer expirations and clean up expired offers?  
   **A:** Periodic background jobs can scan offers and expire or archive them. Alternatively, TTL (time-to-live) can be used in caches or data stores. The system should reject redemptions for expired offers by checking timestamps.

3. **Q:** How do you support different types of offers (e.g., percentage discount, buy-one-get-one, free shipping)?  
   **A:** Use an offer type enum and polymorphism — create an abstract `Offer` base class and subclass for each offer type encapsulating specific discount logic. This design encourages extensibility.

4. **Q:** How do you ensure offer redemption auditability and traceability?  
   **A:** Log every redemption event with metadata (user, offer, timestamp, IP address). Store records in a persistence layer for compliance and analytics.

5. **Q:** How would you handle partial failures, such as the global counter increments but user redemption fails?  
   **A:** Use transactions to atomically update all relevant states. If transaction management is unavailable, implement compensating actions or rollback steps to maintain consistency.

## Takeaways
- Manage offer state carefully with clear validation on time and redemption limits.
- Concurrency control is critical to avoid over-redemption, requiring locking or transactions.
- Design should be extensible for multiple offer types and scalable for many users.
- Handle edge cases such as race conditions near expiry and failure recovery.
- Logging and audit trail ensure traceability and debugging capability.
- Simple synchronized approaches work for prototype or single-node deployments but need robust distributed solutions for production-scale.
