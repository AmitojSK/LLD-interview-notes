# 👻 His Users' Cursors Drifted to Random Positions After Every Edit

> Automatically generated interview-preparation note.

## Original problem

I had OT working. Operations transformed correctly. The interviewer said 'Great — now show me what happens to User B's cursor when User A inserts 10 characters before it.' I said 'I store cursor positions in a HashMap. When a user moves their cursor, I update their entry. Each user can see where others are.'

## Interview-ready answer

## Problem understanding

The problem focuses on managing multiple users' cursor positions in a collaborative text editor using Operational Transformation (OT). The core challenge is to keep all users' cursors correctly synchronized as concurrent edits (inserts, deletes) are applied by different users. Specifically, when User A inserts characters before User B’s cursor, User B’s cursor must be adjusted (shifted) to remain properly positioned relative to the document, reflecting the transformed state.

Constraints and design goals:
- Maintain real-time consistency of cursor positions across multiple users.
- Ensure cursor positions update correctly after concurrent OT operations.
- Support arbitrary insert and delete edits that may affect cursor locations.
- Data structures and updates should be efficient and avoid race conditions or data corruption.
- Keep cursor state consistent with document state after each operation.

---

## Interview answer

To correctly update user cursors after edits, particularly an insert before another user’s cursor, you must apply the same transformation logic to the cursor positions as to the document text.

**Core points:**

1. **Representation:** Store cursor positions as *offset indices* in the document buffer, e.g., an integer representing the character position.

2. **Transformation:** When an edit (insert/delete) occurs, cursors located after the edit position should be adjusted:
   - Insert of length *L* at position *p* means all cursors at or after *p* move forward by *L*.
   - Delete of length *L* at position *p* means cursors after *p + L* shift backward by *L*, cursors inside the deleted range collapse to *p*.

3. **Storing Cursor Positions:** A `Map<UserId, Integer>` is reasonable for holding cursor positions keyed by user ID.

4. **Applying Edits:** Each time an operation is applied to the document, iterate over all user cursors and update positions based on the operation’s positional impact.

**Trade-offs and reasoning:**

- Storing cursor positions as raw offsets is simple but requires careful and consistent transformation on each operation.
- An alternative is to store cursors as references (objects) that can be rebased or mapped through a transformation function.
- Not updating cursors leads to cursor drift and a poor user experience.
- Updating all cursors on every edit might be inefficient with thousands of users; in such cases, more sophisticated structures like interval trees or segment trees can reduce update costs.
- Consider concurrency controls if multiple operations and cursor updates happen in parallel.

---

## Java implementation

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CollaborativeEditor {
    private StringBuilder document = new StringBuilder();
    private final Map<String, Integer> cursorPositions = new ConcurrentHashMap<>();

    /**
     * Inserts text at given position and transforms cursors accordingly.
     * @param position insertion index
     * @param text text to insert
     */
    public synchronized void insert(int position, String text) {
        if (position < 0 || position > document.length()) {
            throw new IllegalArgumentException("Insert position out of bounds");
        }
        int len = text.length();
        document.insert(position, text);
        updateCursorsForInsert(position, len);
    }

    /**
     * Deletes text starting from position of given length, updates cursors.
     * @param position start index of deletion
     * @param length number of characters to delete
     */
    public synchronized void delete(int position, int length) {
        if (position < 0 || position + length > document.length()) {
            throw new IllegalArgumentException("Delete range invalid");
        }
        document.delete(position, position + length);
        updateCursorsForDelete(position, length);
    }

    /**
     * Updates cursors after insertion of length characters at position.
     * Cursors at or after position are moved forward.
     * @param position insert position
     * @param length length of inserted text
     */
    private void updateCursorsForInsert(int position, int length) {
        cursorPositions.replaceAll((user, pos) -> (pos >= position) ? pos + length : pos);
    }

    /**
     * Updates cursors after deletion of length characters at position.
     * Cursors after deleted range are moved backward.
     * Cursors inside deleted range collapse to position.
     * @param position delete position
     * @param length length of deletion
     */
    private void updateCursorsForDelete(int position, int length) {
        cursorPositions.replaceAll((user, pos) -> {
            if (pos >= position + length) {
                return pos - length;
            } else if (pos >= position) {
                return position; // collapse into deleted area start
            } else {
                return pos;
            }
        });
    }

    /**
     * Updates a user's cursor position (e.g. after user moves cursor).
     * @param user user identifier
     * @param position new cursor position
     */
    public void updateCursor(String user, int position) {
        if (position < 0 || position > document.length()) {
            throw new IllegalArgumentException("Cursor position out of bounds");
        }
        cursorPositions.put(user, position);
    }

    /**
     * Returns the current cursor position for a user.
     * @param user user identifier
     * @return cursor index
     */
    public int getCursor(String user) {
        return cursorPositions.getOrDefault(user, 0);
    }

    /**
     * Returns the current document content.
     */
    public String getDocument() {
        return document.toString();
    }
}
```

**Notes on concurrency:**  
- Methods are synchronized to ensure atomicity of document and cursors updates.  
- Using `ConcurrentHashMap` allows concurrent reads; mutations are guarded by synchronization.  
- In a distributed environment, the transformation logic would be part of OT algorithm and happen when OT operations are received.

**Edge cases and failure modes:**  
- Insert/delete out of bounds should throw exceptions.  
- Cursor positions must never be negative or exceed document length.  
- After deletion, cursors in the deleted region collapse safely.  
- Handling simultaneous edits requires OT concurrency control on state updates outside this class.

---

## Key follow-up questions

1. **Question:** How would you handle multiple concurrent edits from different users?  
   **Answer:** Use an OT algorithm to transform operations against each other, ensuring consistency. Cursor updates would follow by transforming cursor positions similarly with the same operational transformations.

2. **Question:** What if a user’s cursor is inside a deleted range, how should it be updated?  
   **Answer:** It’s standard to collapse the cursor to the start of the deleted region to avoid dangling cursors outside the document.

3. **Question:** How would you optimize cursor updates for thousands of users?  
   **Answer:** Use data structures like segment trees or interval trees to batch update cursor positions efficiently without iterating over every cursor on each operation.

4. **Question:** Could storing cursors as offsets cause problems?  
   **Answer:** Yes, if not carefully transformed, cursors can drift from the user’s intended position. Storing cursors as character references or markers with transformation support can improve robustness.

5. **Question:** How do you handle undo/redo with cursor transformations?  
   **Answer:** The undo and redo operations must also transform cursor positions inversely, maintaining consistency across document state history.

6. **Question:** Are there any thread-safety concerns in your implementation?  
   **Answer:** Synchronization is used to prevent race conditions in document and cursor modifications, but in high-performance systems, finer-grained concurrency or immutable data structures might be preferable.

---

## Takeaways

- Cursor synchronization in collaborative editors requires treating cursor positions like "operations" that must be transformed consistently with text edits.
- Simple offset storage for cursors is straightforward but demands careful update logic after every edit to avoid cursor drift.
- Properly transforming cursor positions on every insert/delete leads to a good user experience where cursors always point where the users expect.
- Concurrency and scalability considerations may necessitate more advanced data structures or distributed algorithms.
- Interviewers focus on your ability to extend basic OT logic to related state (like cursors) and discuss failure modes and optimization points.
