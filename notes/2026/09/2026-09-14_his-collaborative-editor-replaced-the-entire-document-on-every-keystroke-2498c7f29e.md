# 📝 His Collaborative Editor Replaced the Entire Document on Every Keystroke

> Automatically generated interview-preparation note.

## Original problem

My design looked solid — Document, User, CollaborativeEditor, Observer Pattern for updates. The interviewer said 'Walk me through what happens when User A types at position 5 while User B types at position 20.' I said 'editDocument() takes the new text string, replaces the content field, and notifies all observers.'

## Interview-ready answer

## Problem understanding

The problem involves designing a collaborative text editor where multiple users can concurrently edit the same document. The main challenges include:

- Handling simultaneous edits from different users without overwriting valid input.
- Avoiding replacing the entire document on every keystroke, which is inefficient and error-prone.
- Ensuring that updates propagate correctly in real-time to all connected users.
- Maintaining consistency and minimizing latency in collaborative editing.
- Properly managing ordering and merging of changes, preferably at the character or range level.

Constraints and goals:
- Support concurrent, potentially conflicting edits.
- Efficiently update document state without full content replacement.
- Use design patterns suitable for event notification (e.g., Observer).
- Provide a clear update mechanism that handles positional edits rather than full text replacement.

## Interview answer

A solid collaborative editor design should manage edits at a granular level (e.g., insertions, deletions at specific positions) rather than replacing the whole document content with every keystroke. 

Key design points include:

1. **Data Model**:
   - A `Document` class containing the text state.
   - Instead of a single content string replaced fully, keep the document as a mutable structure, maybe a `StringBuilder` or another data structure supporting efficient insertions/removals.

2. **Operations**:
   - Model changes as discrete operations (insert, delete, replace) with position and text/length.
   - When User A and User B edit simultaneously, their edits need to be transformed/merged to avoid conflicts — this is a classical problem in collaborative editing.
   - Use Operational Transformation (OT) or Conflict-free Replicated Data Types (CRDTs) for merging concurrent edits correctly.

3. **Update Propagation**:
   - Use the Observer pattern to notify clients of incremental changes (e.g., "insert 'hello' at position 5") rather than sending the entire document.
   - Clients update their local views by applying these operations.

4. **Concurrency and Consistency**:
   - Accept edits in the order they arrive at the server.
   - Transform operations relative to others to maintain document consistency.
   - Careful timestamping or operation versioning is required.

5. **Example Workflow**:
   - User A inserts "X" at pos 5; operation: insert(pos=5, text="X")
   - User B inserts "Y" at pos 20; operation: insert(pos=20, text="Y")
   - Server receives and transforms these operations as needed and broadcasts.
   - Clients apply received operations incrementally.

Trade-offs:
- Operational Transformation and CRDTs add complexity but are necessary for consistent collaboration.
- Full text replacement is simpler but scales poorly and loses concurrency correctness.
- Incremental updates require a more complex client update mechanism.

## Java implementation

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

// Define the change operation interface and a couple of concrete operations
interface DocumentOperation {
    void apply(StringBuilder document);
    int getPosition();
}

class InsertOperation implements DocumentOperation {
    private final int position;
    private final String text;

    public InsertOperation(int position, String text) {
        this.position = position;
        this.text = text;
    }

    public int getPosition() {
        return position;
    }

    public String getText() {
        return text;
    }

    @Override
    public void apply(StringBuilder document) {
        document.insert(position, text);
    }

    @Override
    public String toString() {
        return "Insert '" + text + "' at " + position;
    }
}

// Observer interface for clients interested in changes
interface DocumentObserver {
    void onOperation(DocumentOperation op);
}

class Document {
    private final StringBuilder content = new StringBuilder();
    // Thread-safe list since notifications may occur from multiple threads
    private final List<DocumentObserver> observers = new CopyOnWriteArrayList<>();

    public synchronized void applyOperation(DocumentOperation op) {
        op.apply(content);
        notifyObservers(op);
    }

    public void addObserver(DocumentObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(DocumentObserver observer) {
        observers.remove(observer);
    }

    private void notifyObservers(DocumentOperation op) {
        // Notify asynchronously if needed to prevent blocking
        for (DocumentObserver observer : observers) {
            observer.onOperation(op);
        }
    }

    public synchronized String getContent() {
        return content.toString();
    }
}

// Example editor interface that receives operations from users
class CollaborativeEditor {
    private final Document document;

    public CollaborativeEditor(Document document) {
        this.document = document;
    }

    /**
     * Called when user performs an edit (e.g., insert).
     * This method encapsulates receiving a granular operation.
     */
    public void onUserEdit(DocumentOperation op) {
        // In a real system, transform operations from concurrent users here
        document.applyOperation(op);
    }
}

// A simple client that prints ops
class Client implements DocumentObserver {
    private final String name;

    public Client(String name) {
        this.name = name;
    }

    @Override
    public void onOperation(DocumentOperation op) {
        System.out.println("Client " + name + " received operation: " + op);
    }
}
```

### Notes on concurrency and failure modes

- Using `synchronized` on `applyOperation` ensures consistent updates to `content`.
- In real systems, applying OT or CRDT algorithms would require buffering, conflict resolution, and version tracking.
- Network delays and message reorderings must be accounted for.
- CopyOnWriteArrayList allows safe concurrent observer addition/removal.
- Full state synchronization might be required on client (re-)connections.

## Key follow-up questions

1. Question: How does your design handle simultaneous edits at nearby positions?
   Answer: The design assumes use of Operational Transformation (OT) or CRDTs to transform and merge concurrent operations at the server before applying them, ensuring that edits are integrated consistently even if positions overlap or are close.

2. Question: Why not replace the whole document on every edit?
   Answer: Replacing the whole document on every keystroke is inefficient, causes unnecessary network and processing overhead, and leads to conflicts when multiple users edit simultaneously because the latest overwrite discards concurrent changes.

3. Question: How would you ensure eventual consistency with concurrent edits?
   Answer: By applying OT or CRDT techniques which transform incoming operations relative to other concurrent ones, we ensure all replicas converge to the same state regardless of order of arrival.

4. Question: What data structure would you use to optimize text editing?
   Answer: A gap buffer, piece table, or a balanced tree (like a rope) can be used to optimize insertions and deletions at arbitrary positions in large documents with good performance.

5. Question: How do you handle network failures or client reconnection?
   Answer: Clients may request a full document sync upon reconnection. Additionally, operations can be versioned so clients can catch up by replaying missing operations from the server.

## Takeaways

- Collaborative editors must handle incremental positional edits rather than wholesale document replacement.
- Fine-grained operation modeling (insert/delete/replace at positions) is key.
- Applying well-known algorithms (OT, CRDT) is necessary for consistency under concurrent edits.
- Observer pattern is useful to notify interested clients of incremental changes.
- Handling concurrency, network delays, and failures complicates real-time collaboration.
- Efficient, mutable data structures and partial updates improve performance and user experience.
