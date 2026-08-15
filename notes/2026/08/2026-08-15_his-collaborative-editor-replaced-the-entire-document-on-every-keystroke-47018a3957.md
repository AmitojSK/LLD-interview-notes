# 📝 His Collaborative Editor Replaced the Entire Document on Every Keystroke

> Automatically generated interview-preparation note.

## Original problem

My design looked solid — Document, User, CollaborativeEditor, Observer Pattern for updates. The interviewer said 'Walk me through what happens when User A types at position 5 while User B types at position 20.' I said 'editDocument() takes the new text string, replaces the content field, and notifies all observers.'

## Interview-ready answer

## Problem understanding

The candidate designed a collaborative text editor with basic components: Document, User, CollaborativeEditor, and an Observer pattern to notify updates. The interviewer asked for a scenario where two users edit the same document concurrently. The candidate's design simply replaced the entire document content on every keystroke, then notified observers.

This reveals a fundamental issue: naive full-document replacement on every edit does not scale well for collaborative editing. It doesn't handle concurrent edits, merges, or apply fine-grained changes. It causes performance and consistency problems, especially with multiple users typing at different positions simultaneously.

## Interview answer

In a collaborative editor with concurrent edits, the main concerns are:

- Applying incremental edits rather than replacing the entire document on every change.
- Handling concurrent edits from multiple users, potentially at overlapping or different positions.
- Preserving document consistency and avoiding conflicts.
- Efficiently notifying observers with partial changes.

A better design approach includes:

1. Representing the Document as a sequence of characters (e.g., StringBuilder or gap buffer) with methods for insert, delete, and replace at specific positions.

2. Modeling user input as operations — e.g., `InsertOperation(position, text)`, `DeleteOperation(position, length)` — rather than whole document states.

3. Applying individual operations to the document, modifying only affected regions, not the entire content.

4. Handling concurrent edits using algorithms like Operational Transformation (OT) or Conflict-free Replicated Data Types (CRDT). When User A and User B edit simultaneously:

   - Each user’s operation is transformed based on other concurrent operations to maintain consistency.
   - For example, if User A inserts text at position 5, and User B deletes text at position 20, their operations don’t conflict directly. But if positions overlap, correct transformation is needed.

5. Notifying observers with granular changes (insert/delete at position) instead of entire document replacement notifications.

6. Optionally batching edits per user keystroke sequences to reduce update frequency.

This approach ensures a scalable, real-time collaborative editing experience without replacing the entire document content per keystroke.

## Java implementation

Below is an idiomatic simplified example of applying incremental edits with an observer pattern, illustrating the key points. It skips complex OT or CRDT mechanisms for brevity but shows the design skeleton.

```java
import java.util.*;

interface DocumentObserver {
    void onInsert(int position, String text);
    void onDelete(int position, int length);
}

class Document {
    private final StringBuilder content = new StringBuilder();
    private final List<DocumentObserver> observers = new ArrayList<>();

    public String getContent() {
        return content.toString();
    }

    public synchronized void insert(int position, String text) {
        if (position < 0 || position > content.length()) {
            throw new IndexOutOfBoundsException("Invalid insert position");
        }
        content.insert(position, text);
        notifyInsert(position, text);
    }

    public synchronized void delete(int position, int length) {
        if (position < 0 || position + length > content.length()) {
            throw new IndexOutOfBoundsException("Invalid delete range");
        }
        content.delete(position, position + length);
        notifyDelete(position, length);
    }

    public void addObserver(DocumentObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(DocumentObserver observer) {
        observers.remove(observer);
    }

    private void notifyInsert(int position, String text) {
        for (DocumentObserver observer : observers) {
            observer.onInsert(position, text);
        }
    }

    private void notifyDelete(int position, int length) {
        for (DocumentObserver observer : observers) {
            observer.onDelete(position, length);
        }
    }
}

// Represents an edit operation from a user
abstract class EditOperation {
    final int position;
    EditOperation(int position) {
        this.position = position;
    }
    abstract void apply(Document doc);
}

class InsertOperation extends EditOperation {
    private final String text;

    InsertOperation(int position, String text) {
        super(position);
        this.text = text;
    }

    @Override
    void apply(Document doc) {
        doc.insert(position, text);
    }
}

class DeleteOperation extends EditOperation {
    private final int length;

    DeleteOperation(int position, int length) {
        super(position);
        this.length = length;
    }

    @Override
    void apply(Document doc) {
        doc.delete(position, length);
    }
}

class CollaborativeEditor {
    private final Document document;

    CollaborativeEditor(Document document) {
        this.document = document;
    }

    public synchronized void applyOperation(EditOperation op) {
        // In a real system, operations would be transformed before applying
        op.apply(document);
    }

    public String getDocumentContent() {
        return document.getContent();
    }
}
```

This design:

- Applies user edits as incremental operations.
- Avoids replacing the entire document.
- Notifies observers with precise changes.
- Respects thread safety with synchronization.
- Can be extended with OT or CRDT for concurrency.

## Key follow-up questions

- How would you handle concurrent edits occurring at overlapping or adjacent positions?
- What concurrency control algorithms (OT, CRDT) would you consider?
- How will you ensure eventual consistency among multiple replicas?
- How do you minimize latency and bandwidth for updates in a real network environment?
- How to handle undo/redo operations in collaborative editing?
- What data structures optimize insertion/deletion performance on large documents?
- How do you represent complex edits like copy-paste or replacement?
- How to notify observers efficiently to avoid UI flicker or overload?
- How do you handle authorization and conflict resolution when users have different permission levels?

## Takeaways

- Naively replacing the entire document on every edit does not scale for collaboration.
- Collaborative editing systems must apply incremental, position-based operations.
- Concurrency requires specialized algorithms (OT/CRDT) to merge changes correctly.
- Observer notifications should communicate changes precisely to improve efficiency.
- Designing for thread safety and performance is essential.
- Understanding trade-offs in data structures and consistency models drives effective editor design.

This problem is an excellent test of understanding data modeling, concurrency, synchronization, and distributed systems principles in real-time collaborative applications.
