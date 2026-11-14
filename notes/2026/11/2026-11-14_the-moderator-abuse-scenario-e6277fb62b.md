# ⚖️ The Moderator Abuse Scenario

> Automatically generated interview-preparation note.

## Original problem

Moderator deletes 200 questions.

## Interview-ready answer

## Problem understanding
The problem involves designing a system or feature where a moderator has the ability to delete a large number of questions (e.g., 200). The key concerns are ensuring that this bulk deletion process is efficient, safe, auditable, and respects system constraints such as concurrency, data consistency, and potential misuse or abuse by moderators. The goal is to design a scalable, maintainable, and secure approach to moderator actions affecting multiple entities at once.

## Interview answer
When designing a bulk moderator deletion system, consider the following:

1. **Core Requirements:**
   - Support deletion of multiple questions in a single operation.
   - Maintain data integrity and consistency, especially concerning associated data (comments, answers, votes).
   - Audit moderator actions for accountability and traceability.
   - Handle concurrency to avoid race conditions when multiple moderators operate simultaneously.
   - Implement appropriate access control ensuring only authorized moderators can perform deletions.

2. **Design Approach:**
   - **API:** Provide a bulk delete endpoint accepting a list of question IDs to delete.
   - **Transaction/Batching:** Use database transactions or batch operations to atomically delete or mark questions as deleted (soft deletes preferred for auditability).
   - **Soft Delete:** Prefer marking questions as deleted rather than physical removal to allow recovery and audit.
   - **Audit Logs:** Record moderator ID, timestamp, action type, and affected question IDs.
   - **Rate Limiting:** To prevent abuse, put limits on bulk deletions per moderator or require additional review for large operations.
   - **Eventual Consistency:** If dependencies (answers, comments) are stored in different microservices, implement asynchronous event-driven deletion.
   - **Concurrency:** Use optimistic or pessimistic locking to prevent inconsistent deletion states.

3. **Trade-offs:**
   - **Soft vs Hard Delete:** Soft delete increases storage but enables audit and recovery; hard delete frees space but loses traceability.
   - **Synchronous vs Asynchronous deletion:** Synchronous ensures immediate consistency but may impact performance; asynchronous improves scalability but delays full deletion.
   - **Batch Size:** Large batch sizes improve efficiency but risk long transactions and potential deadlocks.

4. **Security Considerations:**
   - Proper authentication and authorization checks.
   - Detailed audit trail.
   - Confirmation mechanisms for bulk destructive actions.

## Java implementation
```java
import java.time.Instant;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

// Enum for audit actions
enum AuditAction {
    DELETE_QUESTION
}

// Represents a moderator action audit log
class ModeratorAuditLog {
    private final long moderatorId;
    private final AuditAction action;
    private final List<Long> questionIds;
    private final Instant timestamp;

    public ModeratorAuditLog(long moderatorId, AuditAction action, List<Long> questionIds) {
        this.moderatorId = moderatorId;
        this.action = action;
        this.questionIds = List.copyOf(questionIds);
        this.timestamp = Instant.now();
    }

    @Override
    public String toString() {
        return "ModeratorAuditLog{" +
                "moderatorId=" + moderatorId +
                ", action=" + action +
                ", questionIds=" + questionIds +
                ", timestamp=" + timestamp +
                '}';
    }
}

// Simplified representation of a Question entity
class Question {
    private final long id;
    private volatile boolean deleted;

    public Question(long id) {
        this.id = id;
    }

    public synchronized boolean delete() {
        if (deleted) {
            return false;
        }
        deleted = true;
        return true;
    }

    public boolean isDeleted() {
        return deleted;
    }

    public long getId() {
        return id;
    }
}

// Service to manage questions and moderator actions
class ModeratorService {
    // In-memory questions storage (substitute for DB)
    private final ConcurrentHashMap<Long, Question> questions = new ConcurrentHashMap<>();

    // Audit logs storage (append only)
    private final List<ModeratorAuditLog> auditLogs = new java.util.concurrent.CopyOnWriteArrayList<>();

    // Add question (for setup)
    public void addQuestion(Question q) {
        questions.put(q.getId(), q);
    }

    // Bulk delete questions with audit and safety checks
    public synchronized boolean bulkDeleteQuestions(long moderatorId, List<Long> questionIds) {
        // Authorization and rate limiting checks would go here

        boolean allDeleted = true;
        for (Long qId : questionIds) {
            Question q = questions.get(qId);
            if (q == null) {
                allDeleted = false; // Could log missing question IDs
                continue;
            }
            boolean deleted = q.delete();
            if (!deleted) {
                allDeleted = false; // Could log already deleted
            }
        }
        auditLogs.add(new ModeratorAuditLog(moderatorId, AuditAction.DELETE_QUESTION, questionIds));

        return allDeleted;
    }

    public List<ModeratorAuditLog> getAuditLogs() {
        return auditLogs;
    }
}
```

**Explanation:**
- `Question.delete()` is synchronized to avoid race conditions when changing deletion state.
- The bulk delete method synchronizes on the service to avoid parallel conflicting deletions (could be improved with finer concurrency controls).
- Soft delete implemented using a boolean flag.
- Audit logs record the moderator’s action and question IDs.
- The in-memory maps simulate persistence for simplicity.

**Failure Modes & Edge Cases:**
- Questions missing in the list are ignored but would be logged.
- Already deleted questions are skipped.
- This simple implementation does not handle cascading deletes (comments/answers).
- Performance may degrade for very large batches; batching and async processing can improve scalability.

## Key follow-up questions
1. **Q: How would you handle cascading deletes for related entities like answers and comments?**  
   A: Introduce deletion propagation either synchronously in the same transaction or asynchronously via event-driven messaging. For scalability and availability, asynchronous deletion is preferred, where a deletion event triggers consumers to clean up related data. Soft deletes on related entities should be consistent with question deletion.

2. **Q: What security measures would you implement to prevent moderator abuse?**  
   A: Implement strict role-based access control, multi-factor authentication, detailed audit logging with tamper-evidence, rate limiting on bulk deletion operations, and possibly require multi-level approval for large-scale deletions or flag suspicious activity for human review.

3. **Q: How would you design the system to support undo or restore deleted questions?**  
   A: Store deleted questions with a `deletedAt` timestamp and a `deletedBy` field. Implement a "restore" operation that unsets the deleted flag and logs the restoration action. Soft deletion naturally supports undo by preserving the data.

4. **Q: How do you ensure consistency when multiple moderators try to delete overlapping question sets concurrently?**  
   A: Use appropriate concurrency controls such as optimistic locking with version fields or database-level locks within transactions. Additionally, the application layer can synchronize or serialize bulk delete requests per moderator or globally to avoid conflicts.

5. **Q: How would the system scale if moderator deletions increase significantly?**  
   A: Employ asynchronous batch processing with worker queues handling deletion tasks, use distributed locks if necessary, separate audit logging into a scalable log system, and partition data storage to handle large-scale concurrent operations.

## Takeaways
- Bulk deletion by moderators requires balancing performance, safety, and auditability.
- Soft deletes are preferred to retain history and support recovery.
- Audit logs are essential for accountability and detecting abuse.
- Concurrency and consistency must be carefully managed in bulk operations.
- Security and abuse prevention involve access control, rate limiting, and monitoring.
- Scalability often requires asynchronous processing and event-driven architectures.
- Edge cases such as missing entities, partial failures, and cascading dependencies should be planned upfront.
