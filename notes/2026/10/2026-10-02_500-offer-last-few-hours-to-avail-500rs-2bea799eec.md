# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
The problem involves designing a system that handles discount offers like "500 Offer | Last few Hours to Avail 500rs". The key constraints and goals include:

- Support limited-time promotional offers with fixed discount amounts.
- Handle active/inactive states based on time window (e.g., "last few hours").
- Allow users to avail offers only once or a limited number of times.
- Ensure offers apply correctly during checkout.
- Provide extensibility for different offer types and rules.

The design goal is to build a scalable, maintainable, and extensible system managing time-bound discount offers applicable to users purchasing products or subscriptions.

## Interview answer
A robust design for a discount offer system can be built around the following components:

1. **Offer Entity:** Represents an offer with attributes like ID, description, discountAmount (e.g., 500 Rs), startTime, endTime, usageLimitPerUser, totalUsageLimit, currentUsageCount, etc.

2. **Offer State Management:**
   - The offer is active if the current time is between start and end times and if usage limits have not been exceeded.
   - This state can be checked dynamically during queries.

3. **User Offer Usage Tracking:**
   - Maintain a record of which user has availed which offer and how many times.
   - Prevent users from abusing offers beyond allowed limits.

4. **Offer Application Logic:**
   - When a user attempts to checkout, the system checks for applicable active offers.
   - It verifies usage limits and calculates the discounted price accordingly.

5. **Concurrency & Scalability:**
   - Updating usage counts must be atomic to avoid race conditions.
   - Consider database transactions or distributed locks.
   - Caching active offers can optimize read throughput.

6. **Extensibility:**
   - Use an interface or abstract class for different types of offers, e.g., percentage off, fixed amount off, buy-one-get-one, etc.
   - Supports future enhancements easily.

7. **Failure Modes:**
   - Handle offer expiry gracefully.
   - Rollback usage if transaction fails mid-way.

## Java implementation

```java
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class OfferService {
    private final Map<String, Offer> offers = new ConcurrentHashMap<>();
    private final Map<String, Map<String, Integer>> userOfferUsage = new ConcurrentHashMap<>();

    public void addOffer(Offer offer) {
        offers.put(offer.getId(), offer);
    }

    /**
     * Attempt to apply the offer for the user if valid and active.
     * Returns the discount amount if applied, else 0.
     */
    public synchronized int applyOffer(String offerId, String userId) {
        Offer offer = offers.get(offerId);
        if (offer == null || !offer.isActive()) {
            return 0;
        }

        // Check user usage limit
        userOfferUsage.putIfAbsent(userId, new ConcurrentHashMap<>());
        Map<String, Integer> userUsages = userOfferUsage.get(userId);
        int userUsageCount = userUsages.getOrDefault(offerId, 0);

        if (userUsageCount >= offer.getMaxUsagePerUser()) {
            return 0;
        }

        // Check total usage limit
        if (offer.getCurrentUsageCount() >= offer.getTotalUsageLimit()) {
            return 0;
        }

        // Increment usage counts atomically
        offer.incrementUsage();
        userUsages.put(offerId, userUsageCount + 1);

        return offer.getDiscountAmount();
    }

    // Offer class representing the discount offer
    public static class Offer {
        private final String id;
        private final String description;
        private final int discountAmount; // in Rs
        private final Instant startTime;
        private final Instant endTime;
        private final int maxUsagePerUser;
        private final int totalUsageLimit; 
        private int currentUsageCount;

        public Offer(String id, String description, int discountAmount,
                     Instant startTime, Instant endTime,
                     int maxUsagePerUser, int totalUsageLimit) {
            this.id = id;
            this.description = description;
            this.discountAmount = discountAmount;
            this.startTime = startTime;
            this.endTime = endTime;
            this.maxUsagePerUser = maxUsagePerUser;
            this.totalUsageLimit = totalUsageLimit;
            this.currentUsageCount = 0;
        }

        public boolean isActive() {
            Instant now = Instant.now();
            return now.isAfter(startTime) && now.isBefore(endTime)
                    && currentUsageCount < totalUsageLimit;
        }

        public void incrementUsage() {
            currentUsageCount++;
        }

        // Getters...

        public String getId() {
            return id;
        }

        public int getDiscountAmount() {
            return discountAmount;
        }

        public int getMaxUsagePerUser() {
            return maxUsagePerUser;
        }

        public int getTotalUsageLimit() {
            return totalUsageLimit;
        }

        public int getCurrentUsageCount() {
            return currentUsageCount;
        }
    }
}
```

### Explanation:
- `OfferService` manages offers and user usages.
- Synchronization ensures atomicity when applying offers (in a real system, DB transactions or locks should be used).
- `Offer` checks its time validity and usage limits.
- User usage counts are stored per user and offer.

## Key follow-up questions

1. Question: How would you handle concurrency for the usage counts in a distributed environment?
   Answer: In a distributed system, rely on atomic database operations or distributed locks (e.g., Redis locks, Zookeeper) to update usage counters atomically. An optimistic locking mechanism or transactions ensure no overuse occurs.

2. Question: How would you extend your design to support different offer types like percentage discount or buy-one-get-one-free?
   Answer: Define an abstract `Offer` base class or interface with a method `calculateDiscount(Order order)`. Implement concrete subclasses for each type overriding this method.

3. Question: What if the offer needs to be paused or canceled mid-way?
   Answer: Add an "active" flag in the `Offer` class. The `isActive()` method checks this flag along with the time window. Updates to this flag can pause or cancel the offer.

4. Question: How to handle offer stacking or combining?
   Answer: Maintain rules about combinability in the offer metadata. During checkout, apply only compatible offers in order, or disallow multiple offer applications per order if needed.

5. Question: How would you monitor or analyze offer usage success or failures?
   Answer: Log offer application events with timestamps and user info. Use analytics tools or dashboards to track usage patterns, redemption rates, and failure reasons.

## Takeaways
- Designing a discount offer system involves managing time validity, usage limits, and user tracking.
- Concurrency control is crucial to prevent abuse and data corruption.
- Extensibility using abstract classes/interfaces helps accommodate multiple offer types.
- Clear failure handling and active state controls improve reliability.
- Thoughtful design allows easy future enhancements like stacking, cancellation, and analytics.
