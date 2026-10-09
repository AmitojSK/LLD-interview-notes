# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
The problem is to design a system to send promotional emails to a user community offering a time-limited discount (500 Rs off, limited to the last few hours). The system needs to handle:
- Targeting appropriate users (active users engaged in LLD learning)
- Managing campaign timing (last few hours offer window)
- Sending emails reliably and efficiently
- Personalizing or customizing content per user or segment
- Tracking delivery, opens, and possibly conversions to improve future campaigns

Design goals:
- Scalability to large user bases
- Reliability and fault tolerance in email delivery
- Easy scheduling and campaign management
- Compliance with email regulations (e.g., unsubscribe handling)
- Extensibility to support different promotions and audience segments

## Interview answer
For designing a promotional email campaign system with limited-time offers, the core components include:

1. **User Segmentation Service**  
   - Maintains user profiles and activity logs
   - Generates a target list based on engagement (e.g., users reading blogs, solving problems)

2. **Campaign Management Module**  
   - Defines promotion details (discount amount, validity period)
   - Handles scheduling for when the offer should be sent (e.g., last few hours)
   - Allows content customization and template management

3. **Email Sending Service**  
   - Queues email send requests asynchronously
   - Uses SMTP or third-party providers such as SES, SendGrid for scalability and deliverability
   - Supports retry policies for failure handling and backoff
   - Throttles sending rates to prevent spam flags

4. **Tracking and Analytics**  
   - Captures delivery status and user engagement metrics (opens, clicks)
   - Provides insights for campaign effectiveness and future improvements

5. **Compliance and Preferences Management**  
   - Handles unsubscribe requests and user preferences
   - Ensures compliance with GDPR, CAN-SPAM laws

**Trade-offs:**

- Building full email infrastructure vs leveraging third-party services: Third parties reduce operational complexity but increase costs and external dependencies.
- Real-time tracking vs batch processing: Real-time provides timely feedback but requires more infrastructure.
- Personalization depth vs scalability: Deep personalization may slow down campaign rollout but increase engagement.

## Java implementation

Below is a simplified Java design focusing on core classes and concurrency handling for sending batch emails for a time-limited promotion.

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class PromotionalEmailSystem {

    // Represents a user in the system
    static class User {
        final String email;
        final boolean isActiveUser; // e.g., read blogs, practiced problems

        User(String email, boolean isActiveUser) {
            this.email = email;
            this.isActiveUser = isActiveUser;
        }
    }

    // Represents an email campaign
    static class Campaign {
        final String campaignId;
        final String subject;
        final String bodyTemplate;
        final LocalDateTime startTime;
        final LocalDateTime endTime;
        final int discountAmount;

        Campaign(String campaignId, String subject, String bodyTemplate,
                 LocalDateTime startTime, LocalDateTime endTime, int discountAmount) {
            this.campaignId = campaignId;
            this.subject = subject;
            this.bodyTemplate = bodyTemplate;
            this.startTime = startTime;
            this.endTime = endTime;
            this.discountAmount = discountAmount;
        }

        boolean isActive() {
            LocalDateTime now = LocalDateTime.now();
            return now.isAfter(startTime) && now.isBefore(endTime);
        }
    }

    // Email sending service that sends emails asynchronously
    static class EmailSender {
        private final ExecutorService executor;
        private final int maxRetries;
        private final Random failureSimulator = new Random();

        EmailSender(int threadPoolSize, int maxRetries) {
            this.executor = Executors.newFixedThreadPool(threadPoolSize);
            this.maxRetries = maxRetries;
        }

        public void sendEmail(String email, String subject, String body) {
            sendWithRetry(email, subject, body, 0);
        }

        private void sendWithRetry(String email, String subject, String body, int attempt) {
            executor.submit(() -> {
                try {
                    // Simulate random failure for demonstration
                    if (failureSimulator.nextInt(10) < 2) {
                        throw new RuntimeException("Simulated send failure");
                    }
                    // Simulate sending email (here just print)
                    System.out.printf("Sent email to %s with subject: %s%n", email, subject);
                    // In real system, call SMTP or email API here
                } catch (Exception e) {
                    if (attempt < maxRetries) {
                        long backoff = (long) Math.pow(2, attempt) * 100L; // exponential backoff
                        try {
                            Thread.sleep(backoff);
                        } catch (InterruptedException ignored) {
                        }
                        sendWithRetry(email, subject, body, attempt + 1);
                    } else {
                        System.err.printf("Failed to send email to %s after %d attempts%n", email, attempt);
                    }
                }
            });
        }

        public void shutdown() {
            executor.shutdown();
        }
    }

    // Campaign manager that executes a campaign by sending emails to targeted users
    static class CampaignManager {
        private final EmailSender emailSender;

        CampaignManager(EmailSender emailSender) {
            this.emailSender = emailSender;
        }

        public void runCampaign(Campaign campaign, List<User> users) {
            if (!campaign.isActive()) {
                System.out.println("Campaign not active currently.");
                return;
            }
            for (User user : users) {
                if (user.isActiveUser) {
                    String body = personalizeBody(campaign.bodyTemplate, user, campaign.discountAmount);
                    emailSender.sendEmail(user.email, campaign.subject, body);
                }
            }
        }

        private String personalizeBody(String template, User user, int discount) {
            // Very simple placeholder replacement example
            return template.replace("{discount}", String.valueOf(discount));
        }
    }

    // Main method for testing
    public static void main(String[] args) throws InterruptedException {
        List<User> users = Arrays.asList(
            new User("user1@example.com", true),
            new User("user2@example.com", false),  // inactive user
            new User("user3@example.com", true)
        );

        Campaign campaign = new Campaign(
            "c123",
            "Last few hours to avail 500 Rs off!",
            "Dear user, don't miss your chance to get {discount} Rs off on our courses!",
            LocalDateTime.now().minusHours(1),
            LocalDateTime.now().plusHours(1),
            500
        );

        EmailSender emailSender = new EmailSender(5, 3);
        CampaignManager campaignManager = new CampaignManager(emailSender);

        campaignManager.runCampaign(campaign, users);

        // Allow some time for async sending before shutdown
        Thread.sleep(2000);
        emailSender.shutdown();
    }
}
```

**Explanation:**

- `User` stores user details and engagement status.
- `Campaign` holds campaign details and active window.
- `EmailSender` sends emails asynchronously using a thread pool, simulating retries and backoffs.
- `CampaignManager` filters active users and invokes email sends with personalized content.
- The design supports concurrency, retry logic, and simple personalization.

**Edge cases & failure handling:**

- Handles transient failures with retries and backoff.
- Skips inactive users to avoid spamming uninterested users.
- Checks campaign active window before sending.
- Does not block the main thread while sending emails.
- Real implementation would include logging, metrics, and integration with real email services.

## Key follow-up questions

1. **Question:** How would you handle user unsubscribe requests in this system?  
   **Answer:** We'd maintain an unsubscribe list (e.g., a flag per user). Before sending, the CampaignManager queries this list to exclude unsubscribed users. Unsubscribe requests update this list in real-time. Additionally, unsubscribe links in emails direct users to an endpoint updating their preferences.

2. **Question:** How do you ensure the email sending rate does not trigger spam filters?  
   **Answer:** Implement rate limiting in the EmailSender or via external services' API limits. Use batch scheduling and stagger sends over time. Monitor bounce rates and ISP feedback loops to adjust sending volume dynamically.

3. **Question:** How would personalization scale if the number of users grows very large?  
   **Answer:** We can move personalization to a separate lightweight service or use templating engines that support token replacement at scale. Caching user data and employing distributed task queues can parallelize email generation and sending.

4. **Question:** What are potential failure modes and how do you mitigate them?  
   **Answer:** Failures include email server downtime, network issues, and invalid addresses. Mitigate with retries, circuit breakers, fallback mechanisms, and validation of addresses before sending. Logging and alerts ensure rapid detection and remediation.

5. **Question:** How would you track email opens and clicks?  
   **Answer:** Embed tracking pixels or unique links in emails that trigger backend endpoints when opened or clicked. This provides analytics data on engagement. Data should be processed asynchronously to avoid impacting user experience.

## Takeaways

- Design modular components around user targeting, campaign management, and email delivery.
- Use asynchronous email sending and retry mechanisms for reliability.
- Consider compliance and user preferences early.
- Personalization improves engagement but must scale efficiently.
- Handling failures gracefully and tracking outcomes enable iterative improvement.
- Integration with third-party email providers reduces operational complexity but requires abstraction in design.
