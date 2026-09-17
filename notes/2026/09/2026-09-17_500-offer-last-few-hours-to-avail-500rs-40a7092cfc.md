# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding

Design a system to manage and send promotional offers to users of an online learning platform. Key constraints and goals include:

- Deliver personalized email offers to users who are engaged with the platform (e.g., consuming content, practicing problems).
- Ensure timely delivery of limited-time offers (e.g., "last few hours to avail 500rs discount").
- Handle high volume of emails efficiently without overloading the system.
- Manage offer expiration and prevent duplicate or expired offers from being sent.
- Maintain user engagement data to personalize offers.
- Track email delivery and user redemption metrics.

This is a classic case of designing a promotion campaign email system with user-targeting and offer time constraints.

## Interview answer

To design a promotional email system with limited-time offers, focus on these components:

1. **User engagement tracking**: Maintain and update user activity logs (blog readings, problem attempts, etc.) to qualify users for offers.

2. **Offer management**:
   - Model offers with metadata including offer id, discount amount, valid from/to timestamps.
   - Maintain offer lifecycle states: scheduled, active, expired.

3. **Email campaign scheduler**:
   - Periodically run a job to identify active offers and eligible users.
   - Compose personalized emails including dynamic offer details and expiry warnings.
   - Use queues to decouple email preparation and sending, improving scalability.

4. **Email sending infrastructure**:
   - Interface with an SMTP/email provider via async APIs.
   - Implement retry mechanisms on failures.
   - Monitor bounced or undelivered emails.

5. **Offer redemption tracking**:
   - Track if users have redeemed offers or clicked email links.
   - Avoid resending offers to users who have already used them or unsubscribed.

6. **Scalability and fault tolerance**:
   - Use distributed message queues (e.g., Kafka, RabbitMQ) for email jobs.
   - Persist all state changes to databases with transactional guarantees.
   - Use caching to reduce repeated database hits for user engagement data.

7. **Security**:
   - Safeguard sensitive user info.
   - Validate email inputs to avoid injection attacks.
   - Ensure replays or fraudulent redemptions are detected.

**Trade-offs**:

- Using a push model where emails are sent real-time upon triggers gives low latency but higher system load.
- Using periodic batch jobs simplifies architecture but may delay time-sensitive offers.
- Over-personalization risks privacy and complexity.

The proposed design balances timely delivery with scalability by combining scheduled offer activation and queued email sending.

## Java implementation

Below is a simplified Java backend design for managing offer campaigns and sending promotional emails.

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;

class User {
    final String userId;
    final String email;
    boolean unsubscribed;

    // User engagement metrics, updated elsewhere
    boolean isActiveEngagedUser;

    public User(String userId, String email) {
        this.userId = userId;
        this.email = email;
        this.unsubscribed = false;
        this.isActiveEngagedUser = false;
    }
}

class Offer {
    final String offerId;
    final int discountRs;
    final Instant validFrom;
    final Instant validTo;

    public Offer(String offerId, int discountRs, Instant validFrom, Instant validTo) {
        this.offerId = offerId;
        this.discountRs = discountRs;
        this.validFrom = validFrom;
        this.validTo = validTo;
    }

    public boolean isActive() {
        Instant now = Instant.now();
        return now.isAfter(validFrom) && now.isBefore(validTo);
    }
}

class EmailService {
    // Simulate async email sending with retry logic
    private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(10);

    public void sendEmail(String to, String subject, String body) {
        executor.submit(() -> {
            try {
                // Simulate sending email, with manual retry on failure
                System.out.println("Sending email to " + to);
                // Call to actual SMTP provider API here
                // throw new RuntimeException("fail") to simulate failure

            } catch (Exception e) {
                System.err.println("Failed to send email to " + to + ". Retrying...");
                executor.schedule(() -> sendEmail(to, subject, body), 5, TimeUnit.SECONDS);
            }
        });
    }

    public void shutdown() {
        executor.shutdown();
    }
}

class OfferCampaignManager {
    private final Map<String, Offer> offers = new ConcurrentHashMap<>();
    private final List<User> users = new CopyOnWriteArrayList<>(); // Ideally DB-driven
    private final EmailService emailService = new EmailService();

    // Keep track of which user received which offer to prevent duplicates
    private final Map<String, Set<String>> sentOffersByUser = new ConcurrentHashMap<>();

    public void addUser(User user) {
        users.add(user);
    }

    public void addOffer(Offer offer) {
        offers.put(offer.offerId, offer);
    }

    public void runCampaign() {
        Instant now = Instant.now();
        for (Offer offer : offers.values()) {
            if (!offer.isActive()) continue;

            for (User user : users) {
                if (user.unsubscribed) continue;
                if (!user.isActiveEngagedUser) continue;
                // Check if offer already sent to user
                sentOffersByUser.putIfAbsent(user.userId, ConcurrentHashMap.newKeySet());
                if (sentOffersByUser.get(user.userId).contains(offer.offerId)) continue;

                // Compose personalized email
                String subject = "Exclusive Offer: Save " + offer.discountRs + " Rs - Hurry, ending soon!";
                String body = String.format("Hi %s,\n\nThank you for being active on our platform! " +
                                "Enjoy a special discount of %d Rs valid till %s.\nDon't miss out!\n\nBest,\nLLDcoding Team",
                        user.userId, offer.discountRs, offer.validTo);

                emailService.sendEmail(user.email, subject, body);
                sentOffersByUser.get(user.userId).add(offer.offerId);
            }
        }
    }

    public void shutdown() {
        emailService.shutdown();
    }
}

// Example usage (would be in main or a scheduler trigger):

// OfferCampaignManager manager = new OfferCampaignManager();
// manager.addUser(new User("user1", "user1@example.com"));
// manager.addOffer(new Offer("offer500", 500, Instant.now().minusSeconds(3600), Instant.now().plusSeconds(3600)));
// manager.runCampaign();
```

**Notes:**
- User engagement (`isActiveEngagedUser`) should be updated via other subsystems based on user activity.
- Production would use persistent databases and external message queues to handle scale.
- The email service uses a thread pool executor to simulate asynchronous sending and retries.
- `CopyOnWriteArrayList` used for thread-safe iteration of users; in reality, use a persistent store.
- Failure modes include email sending failures, user unsubscribes, and offer expiry.

## Key follow-up questions

1. **Question:** How would you ensure scalability when the user base increases to millions?
   **Answer:** Use distributed message queues to batch and buffer email jobs, shard user databases, and use cloud email delivery services (e.g., AWS SES). Employ asynchronous processing and horizontal scaling of campaign workers.

2. **Question:** How would you handle user unsubscribes to comply with regulations like GDPR?
   **Answer:** Maintain unsubscribe flags per user; check before sending emails. Provide easy unsubscribe links in email. Securely remove or anonymize user data upon requests.

3. **Question:** How can you personalize offers beyond sending the same discount to all active users?
   **Answer:** Use user activity and purchase history data to segment users. Deploy rule-based or ML models to tailor discounts dynamically, e.g., higher discounts for more engaged or long-term users.

4. **Question:** How would you test this system end-to-end?
   **Answer:** Use unit tests for components (email sending, offer validity), mock external SMTP services, and run integration tests with sample user data. Perform load testing for batch campaigns.

5. **Question:** What failure modes exist and how do you mitigate them?
   **Answer:** Email failures — add retries and fallbacks. Expired offers — check validity at send time. Data inconsistencies — use transactions and eventual consistency mechanisms.

## Takeaways

- Designing a promotional email system requires careful user targeting, offer lifecycle management, and scalable email delivery.
- Decoupling via asynchronous queues is critical for handling large volumes and retries.
- Validity windows of offers must be strictly enforced to avoid sending stale promotions.
- User privacy and unsubscribe handling are legal and trust-critical aspects.
- Testing all edge cases and failure scenarios is essential for production readiness.
