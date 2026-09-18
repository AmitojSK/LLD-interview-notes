# 500 Offer | Last few Hours to Avail 500rs

> Automatically generated interview-preparation note.

## Original problem

You've been an amazing part of the LLDcoding community — reading our blogs, practicing problems, and pushing your LLD skills forward.

## Interview-ready answer

## Problem understanding
The problem is to design a system for sending promotional email campaigns targeted at a community of users engaged in low-level design (LLD) learning activities. The campaign includes offers (such as a ₹500 discount) and personalized engagement messages to encourage user participation. The design must efficiently handle large volumes of users, support flexible message templates, offer personalization, and handle scheduling (e.g., last few hours to avail offers). Constraints include ensuring timely delivery, minimizing spam, and respecting user preferences.

## Interview answer
To design a promotional email campaign system for a community platform, we focus on the following:

1. **User Segmentation and Targeting:**
   - Users can be segmented based on activity (e.g., active readers, problem solvers).
   - Campaign metadata includes start/end time (e.g., last hours to avail offer).

2. **Template Management:**
   - Email templates must support dynamic content placeholders (e.g., user name, offer amount).
   - Versions to support A/B testing.

3. **Scheduling and Delivery:**
   - A scheduler triggers email dispatch for the campaign at defined times.
   - A message queue or bulk email services ensure scalable delivery.

4. **Personalization:**
   - Merge fields applied per recipient during email construction.
   - Can include personalized offers, usage stats, etc.

5. **Failure Handling & Retries:**
   - Track delivery status, handle bounces, and retry logic.
   - Maintain logs for audit and compliance.

6. **Opt-out and Compliance:**
   - Respect unsubscribe preferences.
   - Comply with email service regulations.

**Trade-offs & Design Decisions:**
- Using third-party email service providers (ESPs) vs in-house SMTP servers.
- Storing user preferences and campaign data relationally vs NoSQL depending on query needs.
- Design for eventual consistency (for campaign tracking) or strong consistency (for timely opt-out enforcement).

## Java implementation

This simplified Java example models key components: EmailTemplate, Campaign, User, and EmailService.

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;

class User {
    private String email;
    private String name;
    private boolean optedOut;

    public User(String email, String name, boolean optedOut) {
        this.email = email;
        this.name = name;
        this.optedOut = optedOut;
    }

    public String getEmail() { return email; }
    public String getName() { return name; }
    public boolean hasOptedOut() { return optedOut; }
}

class EmailTemplate {
    private String subjectTemplate;
    private String bodyTemplate;

    public EmailTemplate(String subjectTemplate, String bodyTemplate) {
        this.subjectTemplate = subjectTemplate;
        this.bodyTemplate = bodyTemplate;
    }

    public String renderSubject(Map<String, String> params) {
        return replacePlaceholders(subjectTemplate, params);
    }

    public String renderBody(Map<String, String> params) {
        return replacePlaceholders(bodyTemplate, params);
    }

    private String replacePlaceholders(String template, Map<String, String> params) {
        String result = template;
        for (Map.Entry<String, String> entry : params.entrySet()) {
            result = result.replace("{{" + entry.getKey() + "}}", entry.getValue());
        }
        return result;
    }
}

class Campaign {
    private String id;
    private EmailTemplate template;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private Map<String, String> fixedParams;

    public Campaign(String id, EmailTemplate template, LocalDateTime startTime, LocalDateTime endTime, Map<String, String> fixedParams) {
        this.id = id;
        this.template = template;
        this.startTime = startTime;
        this.endTime = endTime;
        this.fixedParams = fixedParams;
    }

    public boolean isActive() {
        LocalDateTime now = LocalDateTime.now();
        return now.isAfter(startTime) && now.isBefore(endTime);
    }

    public EmailTemplate getTemplate() { return template; }
    public Map<String, String> getFixedParams() { return fixedParams; }
}

class EmailService {
    // Simulate sending email
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Sending email to " + to);
        System.out.println("Subject: " + subject);
        System.out.println("Body: " + body);
        // Integrate with SMTP or third-party provider here
    }
}

class CampaignManager {
    private EmailService emailService = new EmailService();
    private ExecutorService executor = Executors.newFixedThreadPool(10);

    public void sendCampaign(Campaign campaign, List<User> users) {
        if (!campaign.isActive()) {
            System.out.println("Campaign is not active.");
            return;
        }
        for (User user : users) {
            if (user.hasOptedOut()) continue;
            executor.submit(() -> sendEmailToUser(user, campaign));
        }
    }

    private void sendEmailToUser(User user, Campaign campaign) {
        Map<String, String> params = new HashMap<>(campaign.getFixedParams());
        params.put("userName", user.getName());
        String subject = campaign.getTemplate().renderSubject(params);
        String body = campaign.getTemplate().renderBody(params);
        try {
            emailService.sendEmail(user.getEmail(), subject, body);
        } catch (Exception e) {
            // logging and retry logic should go here
            System.err.println("Failed to send email to " + user.getEmail());
        }
    }

    public void shutdown() {
        executor.shutdown();
    }
}

public class PromotionalEmailSystem {
    public static void main(String[] args) {
        EmailTemplate template = new EmailTemplate(
                "Special Offer: ₹{{offerAmount}} Off Just for You!",
                "Hi {{userName}},\n\nYou've been an amazing part of the LLDcoding community. Avail your ₹{{offerAmount}} discount before it expires!\n\nCheers!");

        Map<String, String> fixedParams = new HashMap<>();
        fixedParams.put("offerAmount", "500");

        Campaign campaign = new Campaign(
                "CMP2024",
                template,
                LocalDateTime.now().minusHours(1),
                LocalDateTime.now().plusHours(3),
                fixedParams);

        List<User> users = List.of(
                new User("user1@example.com", "Alice", false),
                new User("user2@example.com", "Bob", true),  // opted-out
                new User("user3@example.com", "Charlie", false));

        CampaignManager manager = new CampaignManager();
        manager.sendCampaign(campaign, users);
        manager.shutdown();
    }
}
```

**Explanation:**
- `EmailTemplate` enables placeholder substitution.
- `Campaign` has timing and fixed parameters (the ₹500 offer).
- `User` tracks email, name, and opt-out.
- `CampaignManager` concurrently sends emails only to users who haven't opted out.
- Basic error handling logs failures.
- Concurrency via thread pool ensures scalable sending.

### Edge cases & failure modes:
- Users who opt out during the campaign window.
- Email bounces and feedback loops.
- Template missing dynamic keys.
- Campaign start/end time misconfigurations.
- Network or SMTP provider failures.

## Key follow-up questions
1. **Q:** How would you handle high volume (millions of users) in this design?  
   **A:** Introduce distributed task queues (Kafka, RabbitMQ), batch processing, rate limiting, and leverage ESPs with bulk APIs. Use a data pipeline for segmentation and caching. Employ horizontal scaling for the sender service.

2. **Q:** How do you ensure user privacy and compliance with spam regulations?  
   **A:** Store opt-out data securely; enforce opt-out strictly before sending. Include unsubscribe links. Log consents. Comply with GDPR, CAN-SPAM, and similar laws. Review message content for compliance.

3. **Q:** What metrics would you capture for campaign effectiveness?  
   **A:** Delivery success/failure rates, open rates, click-through rates, conversion rates, bounce rates, unsubscribe rates, and campaign ROI.

4. **Q:** How can you personalize offers dynamically?  
   **A:** Store user attributes and purchase history; use machine learning models or rule-based logic to assign personalized offer values, merged during email generation.

5. **Q:** How would you test the email system?  
   **A:** Unit tests for template rendering and service logic; integration tests for email dispatch (with mocks); end-to-end tests in staging with sample users; manual QA for email formatting; load testing under heavy send load.

## Takeaways
- Effective email campaigns require well-structured template management and personalization.
- Handling opt-outs and compliance is critical to avoid legal and user trust issues.
- Scheduling and concurrency allow scaling to large user bases.
- Failure handling (retry, logging) improves reliability.
- Integration with third-party ESPs often simplifies infrastructure but introduces dependency trade-offs.
- Monitoring and metrics capture empower data-driven improvements.
