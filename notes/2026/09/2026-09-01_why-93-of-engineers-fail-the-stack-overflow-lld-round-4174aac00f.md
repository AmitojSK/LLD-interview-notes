# Why 93% of Engineers Fail the Stack Overflow LLD Round

> Automatically generated interview-preparation note.

## Original problem

Alice clicks upvote. Her request checks the vote map — she hasn't voted. But before the insert, Bob's server thread (also Alice's account from another tab) runs the same check.

## Interview-ready answer

## Problem understanding

The problem describes a concurrency issue in a low-level design (LLD) for an upvote feature, such as on a Q&A platform like Stack Overflow.

- Alice clicks "upvote."
- The system checks if Alice has already voted.
- Since Alice hasn't voted yet, the vote should be recorded.
- However, a race condition arises if Alice acts simultaneously from another thread or browser tab.
- That second request also checks — finds no vote — and both proceed to insert a vote record.
- This causes inconsistent or duplicated votes and shows inadequate concurrency control in the design.

The core problem is managing concurrent requests modifying the state about votes by the same user on the same item, avoiding race conditions and ensuring correctness.

## Interview answer

This concurrency problem is classic in distributed systems or multi-threaded environments where multiple requests attempt to read-modify-write the same data.

Key points to address:

1. **Race condition:** The check-then-act (check if vote exists, then insert) is not atomic, causing duplicate insertions when concurrent requests interleave.

2. **Concurrency control:** To solve the race condition, the system must:

   - Use proper synchronization or locking on the vote check and insert steps, or
   - Employ atomic database operations or constraints to prevent duplicates, or
   - Use optimistic concurrency control techniques or transactions that detect conflicts.

3. **Idempotency:** The system should be idempotent concerning voting — if a vote exists, no new vote is added.

4. **Database constraints:** Enforce a unique constraint on (userId, postId) in the votes table to prevent duplicates at the DB level.

5. **Approaches to fix:**

   - **Pessimistic locking:** Lock the vote entry for this (user, post) pair during the check and insert, e.g., via database row locking or explicit synchronization.
   - **Optimistic locking:** Insert the vote without prior checking and handle duplicate key error if the vote already exists.
   - **Upsert mechanism:** Use database "INSERT ... ON CONFLICT DO NOTHING" or "MERGE" statements to atomically insert if not present.
   - **Atomic read-write operations:** Perform the check and insert as a single atomic operation in the database or via transactions.

6. **Design trade-offs:**

   - Pessimistic locking can serialize operations and hurt scalability.
   - Optimistic locking can be more scalable but requires handling errors explicitly.
   - Database-level constraints are essential as a last line of defense.

To summarize, the system design should ensure atomicity of the vote insertion or rely on database constraints plus proper error handling to avoid duplicate votes due to concurrent requests.

## Java implementation

Below is a simple Java example of handling this scenario on the backend, assuming a relational database and a DAO layer:

```java
public class VoteService {

    private final VoteDao voteDao;

    public VoteService(VoteDao voteDao) {
        this.voteDao = voteDao;
    }

    /**
     * Attempts to upvote a post by a user.
     * Returns true if vote was successfully created, false if vote already exists.
     * Throws exception on other failures.
     */
    public boolean upvotePost(int userId, int postId) {
        try {
            // Attempt atomic insert directly without explicit prior check
            return voteDao.insertVote(userId, postId);
        } catch (DuplicateVoteException e) {
            // This means the vote already exists
            return false;
        }
    }
}
```

The VoteDao interface might look like this:

```java
public interface VoteDao {
    /**
     * Inserts a vote for a user on a post atomically.
     * Returns true if inserted, false if already existing.
     * Throws DuplicateVoteException if unique constraint violation detected.
     */
    boolean insertVote(int userId, int postId) throws DuplicateVoteException;
}
```

If using JDBC with SQL supporting upsert:

```sql
INSERT INTO votes(user_id, post_id, created_at)
VALUES (?, ?, NOW())
ON CONFLICT (user_id, post_id) DO NOTHING;
```

The DAO would execute this; if 0 rows inserted, the vote exists.

Alternatively, if the database does not support upsert, the service may use pessimistic locking with transactions:

```java
public boolean upvotePost(int userId, int postId) {
    // Begin transaction
    Vote existing = voteDao.findVoteByUserAndPostForUpdate(userId, postId);
    if (existing != null) {
        // Already voted
        return false;
    }
    voteDao.insertVote(userId, postId);
    // Commit transaction
    return true;
}
```

Here, `findVoteByUserAndPostForUpdate` acquires a row-level lock to prevent concurrent modifications.

## Key follow-up questions

- How do you enforce data integrity under concurrent writes at the database level?
- What trade-offs exist between optimistic and pessimistic locking for this use case?
- How would your design change in a distributed environment with multiple application servers?
- How do you design APIs and clients to be idempotent?
- What monitoring and alerting would you add to detect race conditions or data inconsistencies?
- How do you handle partial failures in distributed voting (e.g., network partition)?
- Can you think about performance impacts of locking patterns on high-traffic endpoints?

## Takeaways

- Race conditions arise when check-then-act operations are not atomic.
- Concurrency control is vital in any system modifying shared state concurrently.
- Database constraints (unique indexes) are critical as a safety net.
- Optimistic concurrency patterns often yield better scalability but require error detection/handling.
- Pessimistic locking prevents conflicts but can reduce throughput and cause contention.
- Designing idempotent APIs and operations aids robustness in distributed systems.
- Low-level design questions should not only focus on data structures but also concurrency, transactions, and failure handling.
