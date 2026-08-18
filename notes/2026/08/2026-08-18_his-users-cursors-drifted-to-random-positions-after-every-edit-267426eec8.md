# 👻 His Users' Cursors Drifted to Random Positions After Every Edit

> Automatically generated interview-preparation note.

## Original problem

I had OT working. Operations transformed correctly. The interviewer said 'Great — now show me what happens to User B's cursor when User A inserts 10 characters before it.' I said 'I store cursor positions in a HashMap. When a user moves their cursor, I update their entry. Each user can see where others are.'

## Interview-ready answer

## Problem understanding

The problem involves collaborative text editing with Operational Transformation (OT), a technique that allows concurrent edits by multiple users to be merged consistently. The specific challenge is how to correctly update other users’ cursor positions when one user makes an edit, such as inserting or deleting text before their cursor.

In OT systems, when User A inserts text before User B's cursor, User B's cursor position must be incremented accordingly so that it stays logically correct in the updated document. Simply storing each user's cursor position in a HashMap and updating on explicit user cursor moves is insufficient. Without cursor position transformation, cursors appear to drift randomly and inaccurately after edits from other users.

## Interview answer

To handle cursor positions in a collaborative editor with OT, you must:

1. **Store each user’s cursor position relative to the document state at the last known version.**

2. **When receiving an incoming operation (e.g., an insertion or deletion) from another user, update all other users’ cursor positions accordingly by transforming their cursors against that operation.**

   - Example: If User A inserts 10 characters at position 5, and User B’s cursor was at position 15, User B’s cursor should be updated to position 25 (15 + 10) because the document now has 10 more characters before their cursor.

3. **Represent cursors as positions that get transformed using the same OT logic used for transforming operations.**

4. **Apply the transformation on the cursor positions consistently whenever concurrent edits are integrated.**

This means the cursor map isn't updated just when users move their cursors, but also when any edits happen that shift text in the document. This is essential to maintain cursor correctness and avoid random drift.

## Java implementation

Below is a simplified Java example demonstrating how to maintain cursors in an OT-based editor. It assumes:

- Operations are insertions and deletions with a position and length.
- Cursor positions are represented as integers.
- Cursor positions are updated when an operation is applied.

```java
import java.util.HashMap;
import java.util.Map;

public class CursorManager {
    private final Map<String, Integer> cursorPositions = new HashMap<>();

    // Register or update a user's cursor position explicitly (e.g., when user moves cursor)
    public void setCursor(String userId, int position) {
        cursorPositions.put(userId, position);
    }

    // Get current cursor position for a user
    public int getCursor(String userId) {
        return cursorPositions.getOrDefault(userId, 0);
    }

    /**
     * Call this when an operation from another user is applied to the document.
     * It updates all other users' cursors to reflect the change.
     *
     * @param opPosition The start position of the operation
     * @param opLength   The length of the inserted or deleted text (>0 for insert, <0 for delete)
     * @param originUser The user who performed the operation
     */
    public void transformCursors(int opPosition, int opLength, String originUser) {
        for (Map.Entry<String, Integer> entry : cursorPositions.entrySet()) {
            String user = entry.getKey();
            if (user.equals(originUser)) continue; // No transform needed for origin user's cursor here

            int cursorPos = entry.getValue();

            // If operation is an insertion
            if (opLength > 0) {
                // If cursor is after or at insertion point, shift it forward
                if (cursorPos >= opPosition) {
                    cursorPositions.put(user, cursorPos + opLength);
                }
            } else if (opLength < 0) {
                // For deletion: shift cursor left if after deleted range
                int deleteStart = opPosition;
                int deleteEnd = opPosition - opLength; // since opLength is negative
                
                if (cursorPos > deleteEnd) {
                    // Cursor after deleted section shifts left by deleted length
                    cursorPositions.put(user, cursorPos + opLength); // opLength negative
                } else if (cursorPos > deleteStart) {
                    // Cursor was inside deleted section, set to start of deletion
                    cursorPositions.put(user, deleteStart);
                }
                // If cursor before deletion start, no change
            }
        }
    }
}
```

### Usage example

```java
CursorManager cm = new CursorManager();

cm.setCursor("UserA", 10);
cm.setCursor("UserB", 20);

// User A inserts 5 chars at position 5
cm.transformCursors(5, 5, "UserA");

System.out.println(cm.getCursor("UserA")); // 10, unchanged
System.out.println(cm.getCursor("UserB")); // 25, shifted forward by 5
```

## Key follow-up questions

- How do you handle concurrent conflicting cursor transformations, e.g., if multiple edits happen simultaneously?
- How does this scale with large numbers of users and rapid concurrent edits?
- How do you handle selections (ranges) versus single cursor positions?
- How does this integrate with the OT transformation functions used for operations?
- What happens if a user’s cursor is inside a text range that gets deleted?
- How to handle cursor position updates if the user is currently typing or the cursor is being actively moved?

## Takeaways

- Cursor positions in collaborative editors must be transformed using the same or a related logic as the document edits, not merely stored and updated on user interaction.
- Operational Transformation works both for operations and for auxiliary metadata like cursors and selections.
- Failure to transform cursors leads to inconsistent and confusing user experience, with cursors drifting incorrectly.
- Implementing cursor updates correctly requires careful boundary condition handling for insertions, deletions, and overlapping edits.
- Designing cursor management cleanly improves code maintainability and correctness in real-time collaborative systems.
