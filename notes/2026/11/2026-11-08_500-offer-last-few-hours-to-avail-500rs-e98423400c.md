# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
Design a low-level system to send targeted promotional email offers to a community of users based on their engagement history. Key constraints include ensuring scalability to millions of users, avoiding spammy behavior with throttling, personalization of content, and handling partial failures gracefully. The design goals are to deliver personalized, timely e-mails efficiently, track engagement metrics, and support future A/B testing of offers.

## Interview answer
To build an email promotional system:

1. **User Engagement Tracking**: Maintain user profiles with engagement history (blog reads, problem attempts, etc.) to tailor offers.
2. **Offer Personalization**: Use segmentation algorithms to categorize users (active, dormant) and assign appropriate offer content.
3. **Email Campaign Management**: Have a campaign scheduler to start/stop offers, excluding users based on past email frequency to avoid spam.
4. **Email Delivery Pipeline**:
   - Prepare personalized email content templates with placeholders.
   - Use a distributed queue (e.g., Kafka) to enqueue email send requests per user.
   - Multiple worker services consume from the queue, format emails, and send via an email service provider API.
5. **Throttling and Rate Limits**: Enforce per-user and system-wide send limits to comply with provider limits and reduce bounce-back.
6. **Failure and Retry Handling**: On transient failures (e.g., network or SMTP errors), retry with exponential backoff. Persist failures and alert if too many occur.
7. **Metrics and Tracking**: Include tracking pixels or unique links to capture open rates and click-throughs, feeding back into the engagement system.
8. **Extensibility**: Design the system to accommodate other communications (SMS, push) and support A/B testing by randomly assigning variants.

This approach balances scalability, personalization, and delivery reliability.

## Java implementation
Below is a simplified Java implementation focusing on the core email sending pipeline with personalization, concurrency, and failure handling.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class EmailPromotionSystem {
  
    // User profile storing engagement info and email metadata
    static class UserProfile {
        final String userId;
        final String email;
        int emailsSent;
        long lastEmailTimestamp;
        Map<String, Object> engagementData;

        UserProfile(String userId, String email) {
            this.userId = userId;
            this.email = email;
            this.emailsSent = 0;
            this.lastEmailTimestamp = 0L;
            this.engagementData = new HashMap<>();
        }
    }
  
    // Email template with placeholders
    static class EmailTemplate {
        final String subjectTemplate;
        final String bodyTemplate;

        EmailTemplate(String subjectTemplate, String bodyTemplate) {
            this.subjectTemplate = subjectTemplate;
            this.bodyTemplate = bodyTemplate;
        }

        String personalize(Map<String, String> params) {
            String subject = subjectTemplate;
            String body = bodyTemplate;
            for (Map.Entry<String, String> entry : params.entrySet()) {
                subject = subject.replace("{{" + entry.getKey() + "}}", entry.getValue());
                body = body.replace("{{" + entry.getKey() + "}}", entry.getValue());
            }
            return subject + "\n\n" + body;
        }
    }
  
    // Email service provider mock interface
    interface EmailServiceProvider {
        boolean sendEmail(String to, String subject, String body) throws Exception;
    }

    // Basic email service simulation
    static class DummyEmailProvider implements EmailServiceProvider {
        Random random = new Random();
        public boolean sendEmail(String to, String subject, String body) throws Exception {
            // Simulate transient failure 10% of the time
            if (random.nextInt(10) == 0) throw new Exception("Transient SMTP failure");
            System.out.println("Sent email to " + to + " with subject: " + subject);
            return true;
        }
    }

    // Email sending job
    static class EmailJob {
        final UserProfile user;
        final EmailTemplate template;
        final Map<String, String> personalizationParams;

        EmailJob(UserProfile user, EmailTemplate template, Map<String, String> personalizationParams) {
            this.user = user;
            this.template = template;
            this.personalizationParams = personalizationParams;
        }
    }
  
    // Email sending worker with retries
    static class EmailSenderWorker implements Runnable {
        private final BlockingQueue<EmailJob> queue;
        private final EmailServiceProvider provider;
        private final int maxRetries;

        public EmailSenderWorker(BlockingQueue<EmailJob> queue, EmailServiceProvider provider, int maxRetries) {
            this.queue = queue;
            this.provider = provider;
            this.maxRetries = maxRetries;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    EmailJob job = queue.take();
                    int attempt = 0;
                    boolean success = false;
                    Exception lastException = null;

                    while (attempt <= maxRetries && !success) {
                        try {
                            String content = job.template.personalize(job.personalizationParams);
                            String[] parts = content.split("\n\n", 2);
                            String subject = parts[0];
                            String body = parts.length > 1 ? parts[1] : "";
                            provider.sendEmail(job.user.email, subject, body);
                            success = true;

                            // Update user profile post send
                            job.user.emailsSent++;
                            job.user.lastEmailTimestamp = System.currentTimeMillis();
                        } catch (Exception e) {
                            lastException = e;
                            attempt++;
                            if (attempt <= maxRetries) {
                                Thread.sleep((long) Math.pow(2, attempt) * 1000); // exponential backoff
                            }
                        }
                    }
                    if (!success) {
                        System.err.println("Failed to send email to " + job.user.email + ": " + lastException.getMessage());
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }

    // Campaign manager - enqueues emails based on targeting
    static class CampaignManager {
        private final Map<String, UserProfile> userProfiles;
        private final BlockingQueue<EmailJob> emailQueue;
        private final EmailTemplate promotionTemplate;

        public CampaignManager(Map<String, UserProfile> users, BlockingQueue<EmailJob> queue, EmailTemplate template) {
            this.userProfiles = users;
            this.emailQueue = queue;
            this.promotionTemplate = template;
        }

        public void launchCampaign() {
            long now = System.currentTimeMillis();
            for (UserProfile user: userProfiles.values()) {
                // Basic throttle logic: max 3 emails total, 1 email per 24h
                if (user.emailsSent >= 3) continue;
                if (now - user.lastEmailTimestamp < 24*3600*1000) continue;

                Map<String,String> params = new HashMap<>();
                params.put("userName", user.userId);
                params.put("offerAmount", "500rs");
                emailQueue.offer(new EmailJob(user, promotionTemplate, params));
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Map<String, UserProfile> users = new HashMap<>();
        users.put("alice", new UserProfile("alice", "alice@example.com"));
        users.put("bob", new UserProfile("bob", "bob@example.com"));
        
        EmailTemplate promoTemplate = new EmailTemplate(
            "Exclusive {{offerAmount}} Offer Just for You!",
            "Hi {{userName}},\n\nYou've been amazing part of the community. Don't miss out on your exclusive {{offerAmount}} offer — only a few hours left!"
        );
        
        BlockingQueue<EmailJob> emailQueue = new LinkedBlockingQueue<>();
        EmailServiceProvider provider = new DummyEmailProvider();

        // Start workers
        int workerCount = 2;
        ExecutorService executor = Executors.newFixedThreadPool(workerCount);
        for (int i = 0; i < workerCount; i++) {
            executor.submit(new EmailSenderWorker(emailQueue, provider, 3));
        }

        CampaignManager manager = new CampaignManager(users, emailQueue, promoTemplate);
        manager.launchCampaign();

        // Sleep to allow sends to complete
        Thread.sleep(5000);
        executor.shutdownNow();
    }
}
```

### Explanation:
- `UserProfile` stores engagement data and limits.
- `EmailTemplate` allows string substitutions for personalization.
- `EmailSenderWorker` handles sending asynchronously with retries and exponential backoff.
- `CampaignManager` applies throttling and queues jobs.
- Concurrency managed by `ExecutorService` and a thread-safe queue.
- Simulated email sending with transient failure injected.
- Edge cases: throttling prevents spam, retries on failures.

## Key follow-up questions
1. **Q:** How would you handle the case where users may want to opt-out of promotional emails?  
   **A:** Maintain an opt-out flag on user profiles. CampaignManager checks this flag before queueing emails. Provide a mechanism for users to update preferences via UI and sync with the backend database.

2. **Q:** How do you ensure the scalability of email sending at millions of users?  
   **A:** Use distributed queues (Kafka, SQS), horizontal scaling of worker nodes, batching where possible, and delegate actual send infrastructure to specialized providers (SES, SendGrid). Also shard user profiles and campaign logic across services.

3. **Q:** How would you track email opens and clicks?  
   **A:** Embed tracking pixels (small unique image URLs) in the email body referencing our server. Also, use unique campaign links with query parameters to capture clicks. The system collects telemetry asynchronously and updates user engagement stats.

4. **Q:** What failure modes have you considered, and how do those impact data consistency?  
   **A:** Network or SMTP provider failures cause retries with backoff. Permanent failures or invalid emails cause failures logging and exclusion from future retries. Email send state is updated post-success, so duplicate sends are unlikely. If dual writes are used (database + queue), implement idempotency keys to avoid duplicates.

5. **Q:** How would you support A/B testing of offers in this system?  
   **A:** Extend `CampaignManager` to randomly assign variants (A, B) to users and create separate templates per variant. Track variant delivered and subsequent engagement for experiment evaluation.

## Takeaways
- Personalization improves email relevance but adds complexity.
- Throttling and opt-out management are critical for compliance and user experience.
- Using async queues and worker pools enables scalable email sending.
- Failure handling with retries and logging prevents data loss.
- Tracking user responses feeds into adaptive campaign improvements.
- The design must balance between delivery rate, personalization, fault tolerance, and scalability.
