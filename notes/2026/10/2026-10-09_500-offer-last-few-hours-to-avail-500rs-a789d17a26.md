# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
Design a low-level system to manage and deliver promotional offers via email, specifically focused on handling limited-time discount offers (e.g., "Last few hours to avail 500rs off"). The system must handle user segmentation, email scheduling with expiration, and ensure scalability under potentially large user bases. The design should also consider compliance (e.g., not sending to unsubscribed users), retry/failure handling, and metrics tracking.

Constraints and goals:
- Target a subset of users based on engagement or other criteria.
- Send timely emails with expiration or urgency cues ("last few hours").
- Handle concurrent or bulk email sends efficiently.
- Support retries and backoff on failures.
- Provide monitoring and tracking of email delivery and user engagement.
- Ensure compliance with opt-out policies.
- Think about extensibility to other promotional offers and templates.

## Interview answer
To design a promotional email sending system for time-bound offers:

1. **User Selection and Segmentation**:
   - Use a criteria-based filtering system to select recipients (e.g., active users who read blogs, practiced problems).
   - Store user profiles with metadata such as subscription status, engagement level, and preferences.

2. **Offer Management**:
   - Model offers with fields like `offerId`, `description`, `discountAmount`, `validityPeriod`, and `targetSegment`.
   - Maintain a catalog service exposing current offers and their details.

3. **Email Scheduling and Dispatch**:
   - Employ a scheduler or queue that triggers email sends before the offer expiration.
   - Emails should embed the offer info, expiration time, and urgency messaging.
   - Use a reliable email delivery provider or SMTP gateway.
   
4. **Failure Handling & Retry Logic**:
   - Maintain delivery statuses, with retries on transient failures.
   - Use exponential backoff and limit retry counts.

5. **Compliance & Opt-out**:
   - Maintain a suppression list and filter on sending.
   - Handle unsubscribe requests promptly.

6. **Monitoring and Feedback Loop**:
   - Track delivery, opens, clicks, and coupon redemptions.
   - Use these insights to improve targeting and engagement.

7. **Scalability & Performance**:
   - Batch processing for emails to handle scale.
   - Distribute workload using message queues and worker pools.

Trade-offs:
- Real-time sending vs batch scheduling (batch preferred for scale).
- Complex targeting logic vs performance overhead.
- Storing state for retries vs stateless approaches.

## Java implementation

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

// Represents a user profile with subscription & engagement info
class User {
    private final String userId;
    private final boolean isSubscribed;
    private final int engagementScore; // e.g., frequency of blog reads or problem practice

    public User(String userId, boolean subscribed, int engagementScore) {
        this.userId = userId;
        this.isSubscribed = subscribed;
        this.engagementScore = engagementScore;
    }

    public String getUserId() { return userId; }
    public boolean isSubscribed() { return isSubscribed; }
    public int getEngagementScore() { return engagementScore; }
}

// Represents promotional offer details
class Offer {
    private final String offerId;
    private final String description;
    private final int discountAmount;
    private final Instant expiresAt;

    public Offer(String offerId, String description, int discountAmount, Instant expiresAt) {
        this.offerId = offerId;
        this.description = description;
        this.discountAmount = discountAmount;
        this.expiresAt = expiresAt;
    }

    public String getOfferId() { return offerId; }
    public String getDescription() { return description; }
    public int getDiscountAmount() { return discountAmount; }
    public Instant getExpiresAt() { return expiresAt; }
}

// Service to select eligible users for the offer
class UserSegmentationService {
    public List<User> selectEligibleUsers(List<User> allUsers, int engagementThreshold) {
        return allUsers.stream()
                .filter(u -> u.isSubscribed() && u.getEngagementScore() >= engagementThreshold)
                .collect(Collectors.toList());
    }
}

// Email service abstraction
interface EmailSender {
    void sendEmail(String recipient, String subject, String body) throws EmailSendingException;
}

class EmailSendingException extends Exception {
    public EmailSendingException(String msg) {
        super(msg);
    }
}

// Concrete email sender (stub for demonstration)
class SmtpEmailSender implements EmailSender {
    @Override
    public void sendEmail(String recipient, String subject, String body) throws EmailSendingException {
        // Integrate with SMTP or external provider
        System.out.printf("Sending email to %s: %s - %s%n", recipient, subject, body);
        // Simulate success/failure randomly or based on logic
    }
}

// Email job for retry management
class EmailJob implements Runnable {
    private static final int MAX_RETRIES = 3;
    private final String userId;
    private final String recipientEmail;
    private final String subject;
    private final String body;
    private final EmailSender emailSender;
    private int retryCount = 0;

    public EmailJob(String userId, String recipientEmail, String subject, String body, EmailSender sender) {
        this.userId = userId;
        this.recipientEmail = recipientEmail;
        this.subject = subject;
        this.body = body;
        this.emailSender = sender;
    }

    @Override
    public void run() {
        while (retryCount < MAX_RETRIES) {
            try {
                emailSender.sendEmail(recipientEmail, subject, body);
                System.out.printf("Email sent to %s successfully%n", recipientEmail);
                return;
            } catch (EmailSendingException e) {
                retryCount++;
                System.err.printf("Failed to send email to %s, retry %d/%d%n", recipientEmail, retryCount, MAX_RETRIES);
                try {
                    Thread.sleep(1000 * retryCount); // Exponential backoff
                } catch (InterruptedException ignored) {}
            }
        }
        System.err.printf("Giving up on sending email to %s after %d retries%n", recipientEmail, MAX_RETRIES);
    }
}

// Coordinator to create/send offer emails
class OfferEmailService {
    private final EmailSender emailSender;
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public OfferEmailService(EmailSender sender) {
        this.emailSender = sender;
    }

    public void sendOfferEmails(List<User> users, Offer offer) {
        String subject = String.format("%d Rs Offer | Last few Hours to Avail %d Rs", offer.getDiscountAmount(), offer.getDiscountAmount());
        Instant now = Instant.now();
        long hoursLeft = Math.max(0, (offer.getExpiresAt().getEpochSecond() - now.getEpochSecond()) / 3600);
        String urgencyMessage = hoursLeft > 0 ? String.format("Hurry! Only %d hours left.", hoursLeft) : "Offer expired!";
        
        for (User user : users) {
            String body = String.format("Dear user,\n\nYou've been an amazing part of our community.\n%s\n%s\n", offer.getDescription(), urgencyMessage);
            EmailJob job = new EmailJob(user.getUserId(), user.getUserId()+"@example.com", subject, body, emailSender);
            executor.submit(job);
        }
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

**Explanation:**
- `User` class holds subscription and engagement data.
- `Offer` encapsulates the discount offer and expiry.
- `UserSegmentationService` filters eligible users.
- `EmailSender` interface abstracts email dispatch.
- `EmailJob` runnable handles sending with retries.
- `OfferEmailService` coordinates email generation and delivery using a thread pool.
- Edge cases like offer expiration are handled by expiry checks before sending.
  
Failure modes include email send failures (handled by retry), delays in expiration checks, or users unsubscribed after selection (can be rechecked before send).

## Key follow-up questions
1. **Q:** How would you ensure that emails are not sent to unsubscribed users after selection?  
   **A:** Add a final filtering step just before sending emails to exclude users who unsubscribed after the initial segmentation. Also, keep a suppression list indexed for quick lookups to prevent sending.

2. **Q:** How does the system handle scalability when millions of emails are to be sent?  
   **A:** Use distributed message queues (e.g., Kafka) for load leveling, scale email sender workers horizontally, batch processing, and rate-limit per provider. Also, leverage cloud email services (SES, SendGrid) that handle scaling.

3. **Q:** How would you track email opens and clicks to measure engagement?  
   **A:** Embed tracking pixels (tiny one-pixel images) for opens, and track links with unique identifiers redirecting through analytics endpoints that record clicks before redirecting to final URLs.

4. **Q:** How would you schedule sending close to the "last few hours"?  
   **A:** Maintain offer expiry timestamps and schedule email jobs via scheduled executors or cron jobs that activate a few hours before expiry, ensuring users receive timely urgent emails.

5. **Q:** How can you improve deliverability and avoid spam filters?  
   **A:** Follow email best practices: use verified sender domains, DKIM/SPF records, avoid spammy content, stagger email sending rates, and monitor bounce rates.

## Takeaways
- Clear separation of concerns: user segmentation, offer management, email sending.
- Retry logic is crucial for transient failures; exponential backoff reduces load.
- Real-time filtering for unsubscribed users prevents compliance issues.
- Scalability involves batch processing and distributed systems.
- Employ tracking and monitoring to refine targeting and engagement effectiveness.
- The design is extensible to other promotion types and communications with a flexible templating approach.
- Always consider regulatory constraints and user experience when sending promotional emails.
