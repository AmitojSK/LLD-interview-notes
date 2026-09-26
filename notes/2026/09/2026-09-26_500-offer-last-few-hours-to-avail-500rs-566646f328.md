# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding

Design a **Low-Level Design (LLD)** system for managing promotional offers, specifically for handling limited-time discount offers such as "500rs discount available for last few hours." The system should support:

- Activation and expiration of offers.
- Tracking users who are eligible or have availed the offers.
- Sending timely notifications/reminders.
- Handling concurrent access and scalability.
- Robustness against failures (e.g., network issues when sending notifications).
  
Design goals include:

- Efficient management of offer lifecycle (activation, expiry).
- Support for multiple offers with overlapping validity periods.
- Thread-safe concurrent operations, as promotions may be accessed and updated by multiple backend threads.
- Extensibility for different types of offers and notification modes.

## Interview answer

A good LLD for a promotional offer system requires modular components:

1. **Offer Management**:
   - `Offer` entity captures attributes: offerId, description, discountAmount, startTime, endTime, eligibilityCriteria.
   - An `OfferService` handles CRUD and lifecycle operations, including activation and expiration checks.
2. **User Eligibility and Tracking**:
   - Track users who qualify for an offer and those who have redeemed it.
   - Eligibility can be dynamically evaluated via strategies (e.g., user downpayment, purchase history).
3. **Notification System**:
   - Schedule notifications for "last few hours" or reminders.
   - Use event queues or schedulers to send asynchronous notifications.
4. **Concurrency and Consistency**:
   - Use proper concurrency controls or thread-safe data structures (e.g., ConcurrentHashMap).
   - For expiration, use scheduled tasks or delay queues.
5. **Extensibility**:
   - Use interfaces and strategies for eligibility and notification logic.
6. **Failure Handling**:
   - Retry mechanisms on failed notifications.
   - Graceful degradation, e.g., log without crashing.
   
**Trade-offs:**

- Storing state in-memory improves speed but risks data loss on crash; persistent storage is more reliable but slower.
- Complex eligibility criteria could be offloaded to a rules engine for flexibility.
- Notification delivery can be synchronous or asynchronous; async improves throughput but adds complexity.

## Java implementation

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

// Offer Entity
public class Offer {
    private final String offerId;
    private final String description;
    private final double discountAmount;
    private final LocalDateTime startTime;
    private final LocalDateTime endTime;
    private final EligibilityCriteria eligibilityCriteria;

    public Offer(String offerId, String description, double discountAmount,
                 LocalDateTime startTime, LocalDateTime endTime, EligibilityCriteria eligibilityCriteria) {
        this.offerId = offerId;
        this.description = description;
        this.discountAmount = discountAmount;
        this.startTime = startTime;
        this.endTime = endTime;
        this.eligibilityCriteria = eligibilityCriteria;
    }

    public boolean isActive() {
        LocalDateTime now = LocalDateTime.now();
        return !now.isBefore(startTime) && !now.isAfter(endTime);
    }

    public boolean isEligible(User user) {
        return eligibilityCriteria.isEligible(user);
    }
    
    // Getters ...
}

// User Entity
class User {
    private final String userId;
    private final Map<String, Object> attributes; // General user attributes
   
    public User(String userId) {
        this.userId = userId;
        this.attributes = new HashMap<>();
    }
    
    // Getters and setters for attributes
}

// Eligibility Criteria Interface
interface EligibilityCriteria {
    boolean isEligible(User user);
}

// Example EligibilityCriteria: Always Eligible
class AlwaysEligible implements EligibilityCriteria {
    public boolean isEligible(User user) {
        return true;
    }
}

// OfferService manages offers' lifecycle and tracking
public class OfferService {
    private final ConcurrentMap<String, Offer> offers = new ConcurrentHashMap<>();
    private final ConcurrentMap<String, Set<String>> redeemedOffersByUser = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
    private final NotificationService notificationService;

    public OfferService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void addOffer(Offer offer) {
        offers.put(offer.getOfferId(), offer);
        scheduleExpiry(offer);
        scheduleLastHourNotification(offer);
    }

    // Schedule offer expiration - remove after endTime
    private void scheduleExpiry(Offer offer) {
        long delay = java.time.Duration.between(LocalDateTime.now(), offer.getEndTime()).toMillis();
        if (delay > 0) {
            scheduler.schedule(() -> offers.remove(offer.getOfferId()), delay, TimeUnit.MILLISECONDS);
        }
    }

    // Notify users of last few hours (example: notify when 1 hour left)
    private void scheduleLastHourNotification(Offer offer) {
        long notifyDelay = java.time.Duration.between(LocalDateTime.now(), offer.getEndTime().minusHours(1)).toMillis();
        if (notifyDelay > 0) {
            scheduler.schedule(() -> notificationService.notifyOfferEndingSoon(offer), notifyDelay, TimeUnit.MILLISECONDS);
        }
    }

    public boolean redeemOffer(String offerId, User user) {
        Offer offer = offers.get(offerId);
        if (offer == null || !offer.isActive() || !offer.isEligible(user)) return false;

        redeemedOffersByUser.putIfAbsent(user.getUserId(), ConcurrentHashMap.newKeySet());
        Set<String> redeemedOffers = redeemedOffersByUser.get(user.getUserId());

        synchronized (redeemedOffers) {
            if (redeemedOffers.contains(offerId)) {
                return false; // Already redeemed
            }
            redeemedOffers.add(offerId);
        }
        // Additional logic to apply the discount can be done here
        return true;
    }
}

// NotificationService interface with simple console implementation
interface NotificationService {
    void notifyOfferEndingSoon(Offer offer);
}

class ConsoleNotificationService implements NotificationService {
    public void notifyOfferEndingSoon(Offer offer) {
        // In real system, send emails/push notifications asynchronously
        System.out.println("Offer " + offer.getOfferId() + " ends in less than 1 hour! Hurry up!");
    }
}
```

### Explanation:

- **Concurrency:**
  - `ConcurrentHashMap` ensures thread-safe access to offers and redeemed tracking.
  - Redemption per user is synchronized on the per-user redeemedOffers set to prevent duplicate redemptions concurrently.
- **Scheduling:**
  - ScheduledExecutorService handles delayed tasks for offer expiration and notifications.
- **Failure modes:**
  - Currently, failures in sending notifications are not retried; in a production system, failures should be logged and retried.
- **Extensibility:**
  - Eligibility criteria is an interface, supporting different implementations.
  - Notification service can have varied implementations (email, SMS, push).

## Key follow-up questions

1. **Q:** How would you handle persistence in this design?
   **A:** We would integrate a database (e.g., relational or NoSQL) to persist offers, user redemption data, and notification logs. The in-memory state would be a cache updated from persistent storage. Transactional consistency could be ensured via ACID transactions or eventual consistency depending on requirements.

2. **Q:** How do you ensure scalability if millions of users access offers concurrently?
   **A:** Scale horizontally with stateless backend services behind a load balancer. Use distributed caching (e.g., Redis) for offer data, partition user redemption data by user ID. Use asynchronous messaging for notifications and distributed schedulers, possibly leveraging cloud services.

3. **Q:** How would you design the notification system for handling retries and failures?
   **A:** Implement a message queue (Kafka, RabbitMQ) for notifications. Each notification delivery can have retry policies with exponential backoff. Failed attempts logged and monitored. Dead Letter Queue (DLQ) used for problematic messages to prevent blocking.

4. **Q:** How would you design the system to support different types of offers (percentage discounts, buy-one-get-one)?
   **A:** Use the Strategy pattern for offer calculation logic. Define an `OfferType` interface or class hierarchy that encapsulates computation of discount or benefit. The `Offer` class will reference the specific type/strategy.

5. **Q:** How would you handle clock skew issues in a distributed system for offer expiration?
   **A:** Use a centralized and reliable time source (e.g., NTP servers) across systems. In distributed components, rely on event timestamps explicitly rather than local clock where possible. Use TTL and expiration based on database timestamps or token expiry handled at a single source of truth.

## Takeaways

- Clean separation of concerns: offer management, eligibility, redemption, and notification.
- Use of concurrency-safe data structures and synchronizations to prevent race conditions.
- Scheduling mechanisms to handle limited-time events and reminders.
- Extensibility with interfaces and strategies for eligibility and offer types.
- Consider persistence, scaling, and failure-resilient infrastructure in production-ready designs.
