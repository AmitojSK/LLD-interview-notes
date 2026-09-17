# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding

The problem involves designing a structured, scalable email notification system to send promotional offers or updates to a targeted set of users. Key constraints include managing large user bases, ensuring timely delivery, handling personalized content, and supporting rate limiting to prevent spam filters. Design goals focus on system reliability, fault tolerance, extensibility to different campaign types, and maintaining user engagement with minimal downtime or errors.

## Interview answer

To design a promotional email system, we need to address:

1. **User Targeting and Segmentation:**
   - Ability to filter users based on various criteria (e.g., activity, preferences).
   - Support dynamic lists or segments to personalize content.

2. **Email Content Management:**
   - Templates with placeholders for user personalization.
   - Support for HTML/text content and subject lines.

3. **Batching and Rate Limiting:**
   - Handle sending bulk emails in batches to prevent server overload or spam flags.
   - Implement retry for transient failures with exponential backoff.

4. **Delivery and Tracking:**
   - Interface with email delivery service providers (SMTP servers, third-party APIs).
   - Track delivery and open rates for feedback and analytics.

5. **Failure Handling and Idempotency:**
   - Guarantee that duplicate emails are not sent.
   - Ensure partial failures don’t cause inconsistent state.

6. **Extensibility and Monitoring:**
   - Modular design to add other notification channels (SMS, push).
   - Logs and metrics to monitor campaign success or issues.

**Trade-offs:**

- Building a full SMTP service in-house vs. integrating with third parties.
- Real-time sends vs. scheduled batch sends.
- Personalization depth impacting system complexity.

## Java implementation

Below is a simplified design focusing on sending personalized promotional emails ensuring batching, retries, and personalization.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class PromoEmailService {
    private final EmailSender emailSender;
    private final int batchSize;
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

    // To keep retry counts for failed emails
    private final Map<String, Integer> retryCount = new ConcurrentHashMap<>();
    private final int maxRetries = 3;

    public PromoEmailService(EmailSender emailSender, int batchSize) {
        this.emailSender = emailSender;
        this.batchSize = batchSize;
    }

    public void sendPromo(List<User> users, EmailTemplate template) {
        List<List<User>> batches = createBatches(users, batchSize);
        AtomicInteger batchNumber = new AtomicInteger(0);

        for (List<User> batch : batches) {
            int delay = batchNumber.getAndIncrement() * 10; // delay batches by 10 sec intervals
            scheduler.schedule(() -> sendBatch(batch, template), delay, TimeUnit.SECONDS);
        }
    }

    private List<List<User>> createBatches(List<User> users, int size) {
        List<List<User>> batches = new ArrayList<>();
        for (int i = 0; i < users.size(); i += size) {
            batches.add(users.subList(i, Math.min(i + size, users.size())));
        }
        return batches;
    }

    private void sendBatch(List<User> batch, EmailTemplate template) {
        for (User user : batch) {
            sendEmailWithRetry(user, template);
        }
    }

    private void sendEmailWithRetry(User user, EmailTemplate template) {
        String personalizedContent = template.render(user);
        String emailId = user.getEmail();
        try {
            emailSender.send(emailId, template.getSubject(), personalizedContent);
            retryCount.remove(emailId);
            System.out.println("Email sent to: " + emailId);
        } catch (TransientEmailException e) {
            int count = retryCount.getOrDefault(emailId, 0);
            if (count < maxRetries) {
                retryCount.put(emailId, count + 1);
                // Retry with exponential backoff
                int delay = (int) Math.pow(2, count);
                scheduler.schedule(() -> sendEmailWithRetry(user, template), delay, TimeUnit.SECONDS);
            } else {
                System.err.println("Failed to send email to: " + emailId + " after max retries.");
                retryCount.remove(emailId);
                // Log or alert for manual review
            }
        } catch (PermanentEmailException e) {
            System.err.println("Permanent failure for email: " + emailId + ". No retry.");
            // Log permanently failed emails separately
        }
    }

    public void shutdown() {
        scheduler.shutdown();
    }
}

class User {
    private final String email;
    private final Map<String, String> attributes;

    public User(String email, Map<String, String> attributes) {
        this.email = email;
        this.attributes = attributes;
    }

    public String getEmail() {
        return email;
    }

    public Map<String, String> getAttributes() {
        return attributes;
    }
}

class EmailTemplate {
    private final String subjectTemplate;
    private final String bodyTemplate;

    public EmailTemplate(String subjectTemplate, String bodyTemplate) {
        this.subjectTemplate = subjectTemplate;
        this.bodyTemplate = bodyTemplate;
    }

    public String getSubject() {
        return subjectTemplate;
    }

    // Simple variable replacement, e.g., {{name}} replaced with user attribute
    public String render(User user) {
        String content = bodyTemplate;
        for (Map.Entry<String, String> entry : user.getAttributes().entrySet()) {
            content = content.replace("{{" + entry.getKey() + "}}", entry.getValue());
        }
        return content;
    }
}

interface EmailSender {
    void send(String email, String subject, String content) throws TransientEmailException, PermanentEmailException;
}

class TransientEmailException extends Exception {}
class PermanentEmailException extends Exception {}
```

**Explanation:**

- Users are split into batches to avoid overwhelming the email service.
- Each batch is sent with a delay to rate limit sending.
- Retry logic is implemented using scheduled tasks with exponential backoff.
- User details are used for personalizing email content.
- Failures are distinguished between transient (retriable) and permanent (not retriable).
- `EmailSender` interface abstracts the actual email delivery service implementation.
- Scheduler threads are used to handle asynchronous sending and retries.

## Key follow-up questions

1. **Q: How will you handle duplicate emails if the system crashes during sending?**  
   A: We can maintain an idempotency key or state in persistent storage indicating emails sent per user per campaign. Before sending, check the stored state to avoid duplicates, and after successfully sending, update the state.

2. **Q: What are the considerations for choosing batch size and rate limits?**  
   A: Batch size and rate depend on email service provider thresholds, risk of being flagged as spam, and system capacity. Smaller batches reduce overload but increase latency. Trade-off between throughput and deliverability.

3. **Q: How can you extend this system to support multiple channels like SMS or push notifications?**  
   A: Design channel-specific sender interfaces (e.g., `SmsSender`, `PushNotificationSender`) with a common notification interface. The scheduling and retry logic can be abstracted and reused.

4. **Q: How do you track email delivery and opens?**  
   A: Integrate feedback hooks or webhooks from email services. Embed tracking pixels for opens and unique links for clicks. Store these events in analytics to improve targeting and campaign effectiveness.

5. **Q: What failure modes must be handled beyond email sending errors?**  
   A: Failures can include network partitions, data corruption, partial batch failures, or user data inconsistencies. Implementing monitoring, alerts, and fallback mechanisms is essential.

## Takeaways

- Designing an email notification system requires balancing deliverability, personalization, scalability, and fault tolerance.
- Batch processing with scheduling and retries is crucial to manage load and transient failures.
- Abstracting the sending logic allows for flexible integration with various email service providers.
- Persistence and state tracking prevent duplicate sends and support reliable resumption after failure.
- Extensibility for multi-channel notifications and real-time analytics adds business value.
- Common pitfalls include ignoring rate limits, lacking retry mechanisms, and poor error classification leading to missed emails or spamming users.
