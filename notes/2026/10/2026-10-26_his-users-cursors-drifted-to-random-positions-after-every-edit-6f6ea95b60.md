# 👻 His Users' Cursors Drifted to Random Positions After Every Edit

> Automatically generated interview-preparation note.

## Original problem

I had OT working. Operations transformed correctly. The interviewer said 'Great — now show me what happens to User B's cursor when User A inserts 10 characters before it.' I said 'I store cursor positions in a HashMap. When a user moves their cursor, I update their entry. Each user can see where others are.'

## Interview-ready answer

## Problem understanding

In collaborative text editing systems, multiple users concurrently edit a shared document. The challenge is to maintain consistent document state across users and update their cursor positions correctly after each edit.

**Key constraints and goals:**
- When one user performs an edit (insert/delete), the system applies operational transformations (OT) to keep document content consistent.
- User cursors need to update dynamically to reflect the change in text length and position.
- An insert or delete before a user's cursor should shift their cursor accordingly (e.g., if 10 chars inserted before, cursor should move 10 positions ahead).
- Cursor updates must be consistent and avoid drifting randomly, so all users see correct cursor positions after concurrent edits.
- Must efficiently maintain and update cursor positions, potentially for many users.

## Interview answer

The core issue is that cursor positions are stored as absolute indices without tying them to the text operations that modify the document. When User A inserts characters before User B’s cursor, User B’s cursor must shift by the length of the insert, but a naive `HashMap<User, Integer>` storing cursor offsets does not automatically adjust for this.

**Design considerations:**

1. **Cursor as a logical position:**
   - Represent cursors as logical positions that can be updated via operational transformations.
   - When an operation arrives (insert/delete), transform cursor positions accordingly.
   
2. **Per operation update:**
   - Whenever an insert/delete is applied, iterate over all cursors and update positions:
     - If cursor is at or after operation start, move it by the length of the change.
   - For example, if 10 characters inserted at position `p`, then any cursor >= `p` shifts by +10.

3. **More scalable approach with data structures:**
   - For many users, iterating all cursors on each edit is expensive.
   - Use interval trees or balanced binary trees keyed by cursor positions to quickly find cursors that need updates.

4. **Use of OT to transform cursors:**
   - In OT, cursors and operations are both transformed to maintain consistency.
   - Define a transform function on cursors given an operation.
   - This integrates naturally with document operations.

5. **Edge cases:**
   - Cursors inside inserted text (e.g., at operation start) often move to after the new text.
   - Deletion overlapping cursor position may reset cursor position or move it to deletion start.
   - Concurrent operations may cause complex cursor transformations requiring careful OT design.

**Summary:**
Store cursors as offsets, but apply operational transforms to update them. A `HashMap<User, Integer>` alone is insufficient unless you also explicitly update all cursors after every edit.

## Java implementation

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CollaborativeCursorManager {

    private final Map<String, Integer> userCursors = new ConcurrentHashMap<>();

    public void setCursor(String userId, int position) {
        userCursors.put(userId, position);
    }

    /**
     * Call this method after applying a text operation to adjust cursor positions.
     * @param opPosition the position where the operation applies
     * @param lengthChange positive for insert length, negative for delete length
     */
    public void updateCursors(int opPosition, int lengthChange) {
        for (Map.Entry<String, Integer> entry : userCursors.entrySet()) {
            int pos = entry.getValue();
            if (pos > opPosition) {
                // Cursor after operation position shifts by lengthChange
                entry.setValue(pos + lengthChange);
            } else if (pos == opPosition && lengthChange > 0) {
                // Cursor at insert position moves to end of inserted text
                entry.setValue(pos + lengthChange);
            }
            // cursors before opPosition stay unchanged
        }
    }

    public int getCursor(String userId) {
        return userCursors.getOrDefault(userId, 0);
    }
}
```

**Explanation:**

- `userCursors` maintains each user’s cursor as an integer offset.
- After an operation on text is applied, call `updateCursors()` with the start position and length change.
- The method updates all cursors that are at or after the operation position.
- We use a thread-safe `ConcurrentHashMap` to allow concurrent reads/writes.
- Edge cases like concurrent updates and overlapping deletes would require more complex OT logic.

**Potential enhancements:**

- Support cursor affinity policies (e.g., whether cursor sticks before or after inserted text).
- Use more advanced data structures to optimize for large user counts.
- Integrate full OT transform functions that transform both operations and cursors.

## Key follow-up questions

1. **Q: How do you handle deletes overlapping cursor positions?**  
   A: If a delete removes text before a cursor, shift the cursor backward by the length of the deleted text. If a cursor sits inside deleted text, move it to the deletion start to avoid invalid positions.

2. **Q: Can cursors be represented as objects rather than integers? What benefits would this bring?**  
   A: Yes, representing cursors as objects allows storing affinity (stick before/after), selection ranges, and enables more complex transforms than simple integer offsets.

3. **Q: What if multiple edits happen concurrently? How do you keep cursor positions consistent?**  
   A: Use OT or CRDT algorithms to transform concurrent operations and update cursor positions in the same transformed space, applying transformations to cursors alongside document edits.

4. **Q: Is iterating over all cursors to update positions scalable?**  
   A: For small user counts, it is fine. For large-scale systems, use indexed data structures like segment trees or balanced trees keyed by cursor positions to efficiently find affected cursors.

5. **Q: How do you prevent cursors from drifting due to repeated transformations?**  
   A: Apply consistent and invertible operational transformations. Track operation ordering and apply cursor updates atomically and transactionally to avoid cumulative error.

## Takeaways

- Simply storing cursor positions as absolute offsets is insufficient for collaborative editing.
- Cursor positions must be updated consistently via operational transformations applied to all user cursors after every edit.
- Implementing cursor transformation requires careful handling of edge cases such as inserts/deletes at/around cursor positions.
- Scaling cursor updates requires efficient data structures or algorithms.
- Understanding and integrating OT or CRDT principles is critical for building robust collaborative editors.
- Interviewers expect demonstration of understanding of these update dynamics, not just raw storage of cursor positions.
