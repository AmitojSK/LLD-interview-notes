# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
Design a system to handle time-limited discount offers or promotions, such as a "500 Rs discount available for the last few hours." Key constraints and goals include:

- Offer eligibility must be verifiable based on timing (start and end period).
- Users may have personalized eligibility (e.g., only some users qualify).
- The system must ensure the offer is applied correctly and only within the valid window.
- Scalability to handle many users querying and redeeming offers concurrently.
- Ability to send notifications or reminders about expiring offers.

## Interview answer
At a high level, the design must revolve around:

1. **Offer model**: An offer is characterized by a unique ID, start timestamp, end timestamp, discount value, and possibly eligibility conditions.

2. **User eligibility**: Eligibility can be a rule or attribute checked against the user profile or activity.

3. **Offer application logic**:
   - When a user attempts to apply the offer, verify current time is within the offer period.
   - Check user eligibility.
   - Ensure the offer is not applied multiple times if limited to one usage.

4. **Concurrency considerations**: Multiple users may apply offers simultaneously. We need atomic checks and updates to prevent race conditions (for example, if the offer is limited to certain redemptions). Use optimistic or pessimistic locking depending on scalability.

5. **Notifications and reminders**: Optionally, a scheduler can notify users of the offer expiry period approaching.

6. **Persistence and caching**: Store offer metadata and user redemptions in a persistent store (DB). Frequently accessed data can be cached for performance.

Trade-offs include:

- Storing all offer metadata centrally improves consistency but might limit flexibility.
- Caching improves response time but adds complexity to cache invalidation.
- Deciding how strictly to enforce user eligibility can affect UX and system complexity.

## Java implementation

```java
import java.time.Instant;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Represents a limited time offer.
 */
public class LimitedTimeOffer {
    private final String offerId;
    private final Instant startTime;
    private final Instant endTime;
    private final int discountAmount; // in Rs

    public LimitedTimeOffer(String offerId, Instant startTime, Instant endTime, int discountAmount) {
        this.offerId = offerId;
        this.startTime = startTime;
        this.endTime = endTime;
        this.discountAmount = discountAmount;
    }

    public String getOfferId() {
        return offerId;
    }

    public Instant getStartTime() {
        return startTime;
    }

    public Instant getEndTime() {
        return endTime;
    }

    public int getDiscountAmount() {
        return discountAmount;
    }

    public boolean isActive() {
        Instant now = Instant.now();
        return !now.isBefore(startTime) && !now.isAfter(endTime);
    }
}

/**
 * Simple user representation.
 */
class User {
    private final String userId;
    // Additional user attributes can be added here (e.g., subscription status)
    public User(String userId) {
        this.userId = userId;
    }
    public String getUserId() {
        return userId;
    }
}

/**
 * Service to handle offers and user redemptions.
 */
public class OfferService {

    // Store offers by ID
    private final Map<String, LimitedTimeOffer> offers = new ConcurrentHashMap<>();

    // Track user IDs who have redeemed an offer
    private final Map<String, Set<String>> offerRedemptions = new ConcurrentHashMap<>();

    /**
     * Adds a new offer to the system.
     */
    public void addOffer(LimitedTimeOffer offer) {
        offers.put(offer.getOfferId(), offer);
        offerRedemptions.put(offer.getOfferId(), ConcurrentHashMap.newKeySet());
    }

    /**
     * Checks if a user is eligible for the given offer.
     * Placeholder logic - can be extended.
     */
    private boolean isUserEligible(User user, LimitedTimeOffer offer) {
        // For example, check if user is in a whitelist or meets other conditions
        return true; // allowing all users here for demo
    }

    /**
     * Attempts to redeem the offer for the user. Returns discount amount if successful, else 0.
     */
    public int redeemOffer(String offerId, User user) {
        LimitedTimeOffer offer = offers.get(offerId);
        if (offer == null) {
            throw new IllegalArgumentException("Invalid offer ID");
        }
        if (!offer.isActive()) {
            System.out.println("Offer expired or not started yet");
            return 0;
        }
        if (!isUserEligible(user, offer)) {
            System.out.println("User not eligible for offer");
            return 0;
        }
        Set<String> redeemedUsers = offerRedemptions.get(offerId);

        // Atomic check and add to prevent multiple redemptions
        synchronized (redeemedUsers) {
            if (redeemedUsers.contains(user.getUserId())) {
                System.out.println("Offer already redeemed by user");
                return 0;
            }
            redeemedUsers.add(user.getUserId());
        }
        return offer.getDiscountAmount();
    }
}
```

**Notes:**

- This design supports multiple offers and tracks user redemptions.
- `synchronized` on the redemption user set prevents race conditions while redeeming.
- This is an in-memory simplistic example; a real system would persist redemptions and offer data.
- Eligibility logic is simplified and can depend on user attributes or external rules.
- Time checks use `Instant` for clarity and timezone safety.

## Key follow-up questions

1. **Q:** How would you handle multiple concurrent requests from the same user trying to redeem the same offer?
   
   **A:** Synchronizing on the redemption set or using atomic database operations like SELECT FOR UPDATE or unique constraints would ensure only one redemption. Another way is to use distributed locks or atomic compare-and-set operations in the database or cache.

2. **Q:** How would you extend the system to allow offers with limited total redemptions (e.g., only 100 users can redeem)?

   **A:** We would maintain a counter for redemptions and check if the maximum is reached before allowing redemption. This requires atomic increments and concurrency control, possibly via database transactions or atomic counters in Redis.

3. **Q:** How can you ensure offer time validity in a distributed environment with multiple servers that might have clock skew?

   **A:** Use a centralized time service or rely on a common trusted time source like an NTP server. Alternatively, validate offer times in a single authoritative backend service and have other services delegate to that.

4. **Q:** If offers need to be personalized per user (e.g., based on user history), how would you design eligibility checks?

   **A:** Eligibility logic can be encapsulated as separate strategy classes or rule engines that operate on user data and offer parameters. This promotes extensibility and separation of concerns.

5. **Q:** What about notifying users that an offer is about to expire?

   **A:** Implement a scheduled job that checks offers nearing expiry and sends notifications through email, push, or in-app messaging to targeted users.

## Takeaways

- Time-bound offers require careful handling of timing and eligibility constraints.
- Concurrency in redeeming offers necessitates atomic operations or locking mechanisms to prevent double spends.
- Design must account for extensibility in eligibility rules and offer attributes.
- Distributed systems need strategies to handle clock synchronization and data consistency.
- Offer expiry notifications require scheduled background tasks and user targeting mechanisms.
- Persistent backing is required for durability; caching can improve performance but adds complexity.
