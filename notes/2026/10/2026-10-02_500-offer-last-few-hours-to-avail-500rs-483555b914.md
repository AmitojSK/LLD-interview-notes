# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding

Design a backend service for managing discount offers and promotions, specifically handling limited-time offers such as a "₹500 discount" that users can avail within a certain timeframe. Key requirements include:

- Support creating and updating offers with attributes like discount amount, validity period, and eligibility criteria.
- Efficiently determine if a given user can avail an offer at the time of request.
- Handle possible race conditions when multiple users simultaneously try to claim limited offers.
- Track offer usage to prevent abuse or over-redemption.
- Provide mechanisms for notifying users about offer expirations or reminders.
- Scale with increasing user base and number of concurrent requests.

Constraints:

- Offers may have start and end times.
- Some offers may have a limited total number of redemptions.
- System needs to guarantee data consistency, especially for limited stock (like last few hours to avail implies urgency).
- Should handle failure modes gracefully (e.g., partial failures, retry).

Design goals:

- Maintain consistency and integrity of offer usage.
- Ensure good performance under load.
- Provide extensibility for diverse offer types.
- Simple and clean API for frontend/backend integration.

## Interview answer

**1. Core Design**

We design an Offer Management microservice with the following components:

- **Offer Entity**: Holds static info — offer ID, discount amount, start and end time, total available count (if limited), eligibility criteria.
- **Redemption Entity**: Tracks each redemption event with user ID, timestamp, and offer ID.
- **Offer Service**: API for creating, updating, deleting offers, querying availability, redeeming offers.
- **Offer Eligibility Checker**: Encapsulates logic for checking a user’s eligibility (e.g., if user already redeemed, or user category).
- **Offer Repository**: Interface with persistent storage (e.g., RDBMS or NoSQL).
- **Concurrency Control**: To safely update redemption counts, use atomic operations or distributed locking if multiple instances exist.
- **Notification Service (optional)**: For reminding users when offers are about to expire.

**2. Handling Limited-Time & Limited-Count Offers**

- When a user attempts to redeem an offer:
  - Check current time within offer validity period.
  - Check if the user meets eligibility.
  - Atomically check if available redemption count > 0.
  - If yes, decrement the available count and persist a redemption record.
- To guarantee atomicity, use database transactions with row-level locking or optimistic concurrency control.
- Alternatively, use Redis-based atomic counters with Lua scripting for performance-critical paths.

**3. Data Model**

- Offer (offerId, discountAmount, startTime, endTime, totalCount, remainingCount, conditions)
- Redemption (redemptionId, offerId, userId, redemptionTime)

**4. APIs**

- createOffer(Offer)
- updateOffer(Offer)
- getOfferStatus(offerId, userId)
- redeemOffer(offerId, userId)

**5. Failure Modes**

- Concurrent redemption exceeding limits: Use atomic counters or locking.
- Network failures during redemption: Use idempotent requests identified by user+offer.
- Partial failures during offer updates: Use transactional operations.

**6. Extensibility**

- Conditions and eligibility rules can be pluggable.
- Support multiple discount types (fixed amount, percentage, free shipping).
- Support various offer states (active, paused, expired).

## Java implementation

```java
import java.time.Instant;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class OfferService {

    // In-memory data stores to simplify example
    private final ConcurrentHashMap<String, Offer> offers = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ConcurrentHashMap<String, Boolean>> userRedemptions = new ConcurrentHashMap<>();
    
    // Offer represents the discount offer
    static class Offer {
        final String offerId;
        final int discountAmount;
        final Instant startTime;
        final Instant endTime;
        final int totalCount;
        final AtomicInteger remainingCount;
        
        public Offer(String offerId, int discountAmount, Instant startTime, Instant endTime, int totalCount) {
            this.offerId = offerId;
            this.discountAmount = discountAmount;
            this.startTime = startTime;
            this.endTime = endTime;
            this.totalCount = totalCount;
            this.remainingCount = new AtomicInteger(totalCount);
        }

        boolean isValidNow() {
            Instant now = Instant.now();
            return now.isAfter(startTime) && now.isBefore(endTime);
        }
    }

    // Create or update an offer
    public void createOrUpdateOffer(Offer offer) {
        offers.put(offer.offerId, offer);
        userRedemptions.putIfAbsent(offer.offerId, new ConcurrentHashMap<>());
    }

    // Check if a user is eligible for the offer (basic example: only one redemption per user)
    public boolean isUserEligible(String offerId, String userId) {
        if (!offers.containsKey(offerId)) return false;
        Offer offer = offers.get(offerId);
        if (!offer.isValidNow()) return false;
        return !userRedemptions.get(offerId).containsKey(userId);
    }

    // Attempt to redeem an offer
    public synchronized Optional<Integer> redeemOffer(String offerId, String userId) {
        if (!offers.containsKey(offerId)) return Optional.empty();
        Offer offer = offers.get(offerId);
        if (!offer.isValidNow()) return Optional.empty();

        // Check if user already redeemed
        if (userRedemptions.get(offerId).containsKey(userId)) return Optional.empty();

        // Attempt to decrement available count atomically
        while(true) {
            int current = offer.remainingCount.get();
            if (current <= 0) return Optional.empty(); // no offers left
            if (offer.remainingCount.compareAndSet(current, current - 1)) {
                // Mark user redemption
                userRedemptions.get(offerId).put(userId, true);
                return Optional.of(offer.discountAmount);
            }
            // else retry
        }
    }
}
```

**Explanation:**

- `Offer` uses `AtomicInteger` for thread-safe decrement of available count.
- Redeem operation uses `compareAndSet` loop to avoid race-condition.
- Redemption per user tracked in a nested `ConcurrentHashMap` to prevent multiple redemptions.
- Synchronization only around critical section is done carefully via atomic operations for scalability.
- This is a simplified in-memory implementation; in production a durable persistent store and distributed locks or transactions would be needed.

## Key follow-up questions

1. **Q:** How would you handle offer redemption in a distributed system with multiple service instances?  
   **A:** Use a centralized persistent store with transactions or distributed locks to ensure atomic update of remaining count. Alternatively, use Redis atomic counters with Lua scripting to maintain consistency and performance under concurrency.

2. **Q:** How can you prevent users from abusing multiple accounts to redeem the same offer?  
   **A:** Implement user identity verification, rate limiting, and track redemptions based on unique user identifiers such as phone numbers, emails, or device fingerprints. Also, monitor suspicious patterns or use CAPTCHA.

3. **Q:** How would you design the system to support complex eligibility rules?  
   **A:** Abstract eligibility rules into a separate module or strategy pattern, allowing you to plug in rule engines or scriptable policies evaluated at runtime.

4. **Q:** What would you do if the system needs to support millions of users redeeming offers simultaneously?  
   **A:** Use caching for offer info, distributed atomic counters to manage limits, horizontally scale service instances, and distribute redemption load. Use eventual consistency where strict consistency isn't critical and employ asynchronous processing for notifications.

5. **Q:** How would you handle the scenario where an offer is updated or cancelled after some redemptions?  
   **A:** Clearly define versioning or state transitions for offers (e.g., active, paused, cancelled). Ensure subsequent redemptions are blocked post-cancellation. Provide compensation or rollback mechanisms if needed.

## Takeaways

- Use atomic operations and transactional consistency for handling limited resource counts (like limited offer redemptions).
- Design data models to cleanly separate offer metadata vs redemption records.
- Extensibility and modularity in eligibility checking make evolving business rules manageable.
- Concurrency and scale require careful use of locking or atomic counters in distributed systems.
- Build APIs to gracefully handle failure modes, partial failures, and retries.
- Consider anti-abuse mechanisms and user authentication to prevent exploitation of limited offers.
