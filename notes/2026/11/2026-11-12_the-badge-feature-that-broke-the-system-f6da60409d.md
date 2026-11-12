# 🏅 The Badge Feature That Broke the System

> Automatically generated interview-preparation note.

## Original problem

Let’s give a badge after 100 upvotes. That’s a great idea! But what if someone finds a way to upvote their own answer 100 times? Let’s find out how we can prevent that from happening.

## Interview-ready answer

## Problem understanding
Design a badge awarding system that grants a badge to users whose answer receives at least 100 upvotes. The key constraints and challenges include:

- Preventing fraudulent behavior, specifically users upvoting their own content multiple times to undeservedly earn badges.
- Ensuring data integrity and correctness of badge awarding, even in a concurrent, high-traffic environment.
- Possibly dealing with scale, as many users and answers will be voting simultaneously.
- Designing appropriate data structures and business logic to detect and prevent self-upvoting abuse.

## Interview answer
The core functionality involves awarding a badge once an answer crosses a threshold of 100 upvotes. However, the main concern is preventing the user who authored the answer from gaming the system by upvoting their own answer multiple times.

Key aspects in the design:

1. **Data Model:**

   - **User**: identified by a unique userId.
   - **Answer**: identified by an answerId, with a reference to the authorUserId.
   - **Vote**: tracks who voted for which answer. Each (userId, answerId) pair should be unique.

2. **Constraints to prevent abuse:**

   - Ensure only one vote per user per answer.
   - Disallow the author of the answer from voting on their own answer.

3. **Badge awarding:**

   - Once the count of qualified votes (excluding invalid/self votes) reaches 100, award the badge.
   - Badge awarding should be idempotent, and concurrent votes can happen, so proper synchronization or atomic counters are needed.

4. **Concurrency control:**

   - Use atomic counters or a transaction-based approach to ensure correctness under concurrent voting.
   - Possibly use a caching or event-driven approach to minimize DB load when checking badge thresholds.

5. **Trade-offs:**

   - **Strict checks** (disallowing self-votes) simplify fraud prevention but may need to handle edge cases where votes are removed or users change answers.
   - Storing all votes for auditing is useful for future enhancements, but can be costly at scale.
   - Real-time badge awarding vs batch processing: real-time gives instant feedback but costs more in complexity; batch is simpler but delayed.

## Java implementation

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class BadgeService {
    // Map answerId -> upvote count
    private final ConcurrentHashMap<String, AtomicInteger> upvoteCounts = new ConcurrentHashMap<>();

    // Tracks (userId, answerId) to prevent duplicate votes or self voting
    private final ConcurrentHashMap<String, String> votes = new ConcurrentHashMap<>();
    // Map answerId -> authorUserId
    private final ConcurrentHashMap<String, String> answerAuthors = new ConcurrentHashMap<>();

    // Threshold for badge awarding
    private static final int UPVOTE_THRESHOLD = 100;

    // Register a new answer
    public void addAnswer(String answerId, String authorUserId) {
        answerAuthors.put(answerId, authorUserId);
        upvoteCounts.put(answerId, new AtomicInteger(0));
    }

    // Upvote an answer by a user
    public boolean upvote(String userId, String answerId) {
        String authorUserId = answerAuthors.get(answerId);
        if (authorUserId == null) {
            throw new IllegalArgumentException("Answer does not exist");
        }

        // Prevent user from voting their own answer
        if (userId.equals(authorUserId)) {
            System.out.println("Self-voting not allowed");
            return false;
        }

        String voteKey = userId + ":" + answerId;

        // If user already voted for this answer, reject
        if (votes.putIfAbsent(voteKey, "VOTED") != null) {
            System.out.println("Duplicate vote detected");
            return false;
        }

        // Increment count atomically
        int newCount = upvoteCounts.get(answerId).incrementAndGet();

        // Award badge if threshold reached
        if (newCount == UPVOTE_THRESHOLD) {
            awardBadgeToAuthor(authorUserId, answerId);
        }

        return true;
    }

    private void awardBadgeToAuthor(String userId, String answerId) {
        // Badge awarding logic (e.g. update user profile, send notification)
        System.out.println("Badge awarded to user " + userId + " for answer " + answerId);
    }
}
```

**Explanation:**

- We map each answer to its author and maintain an atomic upvote count.
- For each vote, we verify the voter is not the answer author and has not voted before.
- Votes are tracked in a map with a composite key `userId:answerId` for O(1) duplicate checks.
- Upvote count is updated atomically using `AtomicInteger`.
- Badge awarding occurs once the count hits the threshold.
- This implementation is thread-safe for concurrent upvote operations.
- Edge cases like deleted answers or vote retractions are not handled here but can be added.

## Key follow-up questions

1. **Q:** How would you handle vote retractions (removing votes)?
   **A:** Allow users to remove their votes by deleting the corresponding key from `votes` map and decrementing the atomic count. Additional synchronization or transactional control is needed to avoid race conditions. Badge revocation logic would also be necessary if votes fall below the threshold.

2. **Q:** What if a user creates multiple accounts to fraudulently upvote?
   **A:** This is a more complex anti-fraud concern that requires identity verification, rate limiting, or fraud detection algorithms. One may implement heuristics based on IPs, user behavior, or require stronger authentication.

3. **Q:** How to scale this solution for millions of votes and answers?
   **A:** Move to persistent storage (DB) for votes, use batch processing, caching layers, and possibly event-driven badge awarding. Using distributed locks or atomic operations supported natively by the data store is crucial.

4. **Q:** How to handle consistency between badge awarding and vote counts?
   **A:** Use transactional updates or distributed consensus if in a distributed environment to ensure badge awarding is exactly-once and consistent with votes data.

5. **Q:** What if the threshold changes dynamically or multiple badges exist?
   **A:** Encapsulate badge criteria in configurable rules, possibly stored externally, and design badge awarding logic to be extensible for multiple badge types and thresholds.

## Takeaways

- Preventing self-voting is fundamental and simple: reject votes from the answer author.
- Ensuring one vote per user per answer is necessary to maintain fairness.
- Concurrent vote counting must be thread-safe to correctly trigger badge awarding.
- Complex fraud detection (multiple fake accounts) requires more advanced mechanisms.
- Carefully consider badge awarding consistency in distributed or high-scale systems.
- Real-time feedback vs batch processing trade-offs must be evaluated based on system requirements.
