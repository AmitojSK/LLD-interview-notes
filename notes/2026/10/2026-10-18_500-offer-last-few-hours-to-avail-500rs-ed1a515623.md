# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
Design a system to manage and deliver personalized promotional offers via email to the users of a platform focused on low-level design (LLD) education. Constraints include handling large-scale user data securely, ensuring timely and relevant offer delivery, and maintaining user engagement without overwhelming them. Design goals involve supporting personalization, scheduling and throttling email sends, tracking offer usage, and providing analytics on campaign effectiveness.

## Interview answer
The system should consist of the following components:

1. **User Profile and Preferences Service**  
   Maintain detailed profiles capturing user activity, interests, and engagement history with LLD content to personalize offers.

2. **Offer Management Service**  
   Create, update, and manage various promotional offers like a "500rs discount" with metadata such as validity, terms, and target segments.

3. **Email Campaign Scheduler**  
   Schedule and trigger email campaigns based on offer validity duration (e.g., "Last few Hours" notifications), user timezone, and optimal send times.

4. **Personalization Engine**  
   Use user data to customize email content dynamically, enhancing engagement through relevant offers and messaging.

5. **Email Delivery System**  
   Interface with an email delivery provider (e.g., SES, SendGrid), ensuring reliability, handling retries, and tracking delivery statuses.

6. **Analytics and Feedback Loop**  
   Track open rates, click-through, and offer redemption to analyze campaign effectiveness and refine personalization logic.

**Trade-offs and Reasoning**  
- **Batch Processing vs Real-time:** Use batch jobs for periodic campaigns to optimize resources; real-time triggers for critical, time-sensitive offers.  
- **Personalization Depth:** Deeper personalization improves engagement but increases data processing cost and privacy considerations. Balance based on ROI.  
- **Reliability:** Use message queues and retries to ensure email delivery despite transient failures.  
- **Privacy and Compliance:** Handle user data per regulations (GDPR, CAN-SPAM), including unsubscribe options.

## Java implementation

```java
import java.time.*;
import java.util.*;
import java.util.concurrent.*;

public class OfferEmailSystem {

    // Simplified User profile
    static class User {
        final String userId;
        final String email;
        final ZoneId timezone;
        final Set<String> interests; // e.g., "LLD", "Coding"
        boolean optedOut;

        public User(String userId, String email, ZoneId timezone, Set<String> interests) {
            this.userId = userId;
            this.email = email;
            this.timezone = timezone;
            this.interests = interests;
            this.optedOut = false;
        }
    }

    // Offer model
    static class Offer {
        final String offerId;
        final String description;
        final LocalDateTime validFrom;
        final LocalDateTime validUntil;
        final Set<String> targetInterests;

        public Offer(String offerId, String description, LocalDateTime validFrom,
                     LocalDateTime validUntil, Set<String> targetInterests) {
            this.offerId = offerId;
            this.description = description;
            this.validFrom = validFrom;
            this.validUntil = validUntil;
            this.targetInterests = targetInterests;
        }

        public boolean isValid(LocalDateTime now) {
            return (now.isEqual(validFrom) || now.isAfter(validFrom)) &&
                   (now.isBefore(validUntil) || now.isEqual(validUntil));
        }
    }

    // Email sending interface simulating integration with external provider
    interface EmailSender {
        boolean sendEmail(String email, String subject, String body);
    }

    // Simple implementation for demonstration
    static class SimpleEmailSender implements EmailSender {
        public boolean sendEmail(String email, String subject, String body) {
            System.out.println("Sending email to " + email);
            // In production, integrate with email API, handle failures, retries etc.
            return true;
        }
    }

    // Personalization engine example
    static class PersonalizationEngine {
        public String personalizeEmail(User user, Offer offer) {
            // Basic example: include offer description and user name
            return "Hi there!\n\n" +
                   "You've been an amazing part of the LLDcoding community. " +
                   "We have an exclusive offer for you: " + offer.description + ".\n" +
                   "Hurry, valid until " + offer.validUntil.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME) +
                   ".\n\nBest,\nTeam LLDcoding";
        }
    }

    // Campaign Scheduler: schedules email sending tasks
    static class CampaignScheduler {
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
        private final EmailSender emailSender;
        private final PersonalizationEngine personalizationEngine;

        public CampaignScheduler(EmailSender emailSender, PersonalizationEngine personalizationEngine) {
            this.emailSender = emailSender;
            this.personalizationEngine = personalizationEngine;
        }

        public void scheduleCampaign(List<User> users, Offer offer) {
            LocalDateTime now = LocalDateTime.now();

            if (!offer.isValid(now)) {
                System.out.println("Offer not valid for campaign scheduling.");
                return;
            }

            for (User user : users) {
                if (user.optedOut) continue;
                // Check if user interests align with offer target interests
                boolean matchesInterest = !Collections.disjoint(user.interests, offer.targetInterests);
                if (!matchesInterest) continue;

                // Avoid sending multiple emails for the same offer (not implemented here but recommended)
                // Schedule email respecting user timezone, e.g., send at 9 AM user local time
                ZonedDateTime userSendTime = ZonedDateTime.now(user.timezone)
                        .withHour(9).withMinute(0).withSecond(0).withNano(0);
                ZonedDateTime nowUtc = ZonedDateTime.now(ZoneOffset.UTC);
                long delaySeconds = Duration.between(nowUtc, userSendTime.withZoneSameInstant(ZoneOffset.UTC)).getSeconds();
                if (delaySeconds < 0) delaySeconds = 0; // send immediately if time passed

                scheduler.schedule(() -> {
                    String content = personalizationEngine.personalizeEmail(user, offer);
                    boolean sent = emailSender.sendEmail(user.email, "Exclusive Offer Just for You!", content);
                    if (sent) {
                        System.out.println("Email sent to userId:" + user.userId);
                        // Track email sent event for analytics (not implemented here)
                    }
                }, delaySeconds, TimeUnit.SECONDS);
            }
        }

        public void shutdown() {
            scheduler.shutdown();
        }
    }

    public static void main(String[] args) {
        List<User> users = List.of(
            new User("u1", "user1@example.com", ZoneId.of("Asia/Kolkata"), Set.of("LLD", "Java")),
            new User("u2", "user2@example.com", ZoneId.of("Europe/London"), Set.of("Python"))
        );

        Offer offer = new Offer("offer500", "Get 500 Rs off on premium LLD course!",
                LocalDateTime.now().minusHours(1),
                LocalDateTime.now().plusHours(5),
                Set.of("LLD"));

        SimpleEmailSender emailSender = new SimpleEmailSender();
        PersonalizationEngine personalizationEngine = new PersonalizationEngine();
        CampaignScheduler scheduler = new CampaignScheduler(emailSender, personalizationEngine);

        scheduler.scheduleCampaign(users, offer);

        // Add shutdown hook to gracefully terminate scheduler
        Runtime.getRuntime().addShutdownHook(new Thread(scheduler::shutdown));
    }
}
```

**Explanation**:  
- `User` includes timezone for scheduling emails at optimal local times and interests for targeted offers.  
- `Offer` includes validity window and target audience metadata.  
- `CampaignScheduler` uses `ScheduledExecutorService` to delay email sending according to user timezones.  
- Email content is personalized lightly via `PersonalizationEngine`.  
- The implementation demonstrates concurrency handling via scheduled tasks and simulates the email sending provider integration.

**Failure modes and edges**:  
- Email sending can fail; should have retry mechanisms and dead-letter queues in production.  
- User opting out disables emails.  
- Timezone edge cases when scheduling near DST changes should be handled carefully (not shown).  
- Duplicate sends can be avoided by tracking history (not shown).

## Key follow-up questions
1. **Q: How would you handle scaling this system for millions of users?**  
   A: Introduce distributed processing with message queues (Kafka, RabbitMQ) for email tasks, horizontally scale the scheduler, and shard user data. Use cloud email services with high throughput and manage rate limits.

2. **Q: How do you ensure compliance with regulations like GDPR and CAN-SPAM?**  
   A: Store user consent, provide easy unsubscribe mechanisms, honor opt-out flags immediately, avoid sending personal data in payloads, and securely manage user data with encryption and access controls.

3. **Q: How can you improve email deliverability and engagement?**  
   A: Use domain authentication (SPF, DKIM), monitor bounce rates, segment users for relevance, A/B test subject lines and content, and optimize send times via analytics.

4. **Q: How would you track and measure the offer redemption rate?**  
   A: Embed unique tracking tokens or coupon codes in emails, log clicks via redirect URLs, and tie these to backend offer redemption events to calculate conversion rates.

5. **Q: What happens if multiple campaigns target the same user simultaneously?**  
   A: Implement throttling and queuing logic to avoid email spamming, prioritize campaigns based on business rules, or batch multiple offers into a single email.

## Takeaways
- Designing promotional email offer systems requires careful balance of personalization, scalability, and compliance.  
- Timezone-aware scheduling and respecting user preferences enhances user experience and engagement.  
- Leveraging concurrency and scheduled tasks helps distribute load and ensure timely delivery.  
- Key pitfalls include ignoring opt-outs, failure to handle email delivery failures, and poor personalization leading to low conversion.  
- Building a feedback loop with tracking and analytics is vital for continuous improvement.
