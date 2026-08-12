# Deadlock on the File System LLD Roundtable

> Automatically generated interview-preparation note.

## Original problem

The Google L5 question that destroyed a senior engineer who said "I'll lock the whole tree with one mutex".

## Interview-ready answer

## Problem understanding

The problem revolves around designing a file system or tree-like structure with concurrency in mind, specifically focusing on avoiding deadlocks during locking operations. A naive approach where one tries to lock the entire tree using a single mutex leads to performance bottlenecks and scalability issues. Conversely, locking individual nodes or parts of the tree requires careful handling to avoid deadlocks caused by circular waiting when multiple threads try to lock overlapping resources.

## Interview answer

### Key points:
- Locking the whole tree with one global mutex serializes all operations, killing concurrency.
- Fine-grained locking (e.g., per node or per subtree) allows concurrent operations but introduces deadlock risks.
- To avoid deadlocks, locks must be acquired in a consistent, hierarchical order.
- One common approach is to acquire locks in top-down order (e.g., from root to leaf), never ascending locks backwards.
- If multiple locks are needed, establish a global locking order to prevent cyclic dependencies.
- Techniques such as lock coupling, hand-over-hand locking, or using try-lock with backoff/retry can help.
- Using optimistic concurrency (e.g., versioning with validation) can reduce locking needs.
- Sometimes read-write locks (ReentrantReadWriteLock) can allow multiple readers without blocking.
  
### Deadlock scenario to avoid:
- Thread A locks node N1 then waits for N2.
- Thread B locks node N2 then waits for N1.
- Both are waiting on each other's locks — creating a deadlock cycle.

### Final design:
- Lock nodes along the path in a single top-down pass.
- If acquiring multiple sibling nodes, define a strict order (e.g., lexicographical) in acquiring locks.
- Avoid locking parents after children are held.
- Consider optimistic reads with validation and exclusive locking only on writes.

## Java implementation

Below is a simplified Java snippet illustrating lock ordering for a file-system tree node. This example uses ReentrantLock on each node and demonstrates acquiring locks for a path from root to leaf safely.

```java
import java.util.*;
import java.util.concurrent.locks.ReentrantLock;

class FileSystemNode {
    private final String name;
    private final Map<String, FileSystemNode> children = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public FileSystemNode(String name) {
        this.name = name;
    }

    // Add child node
    public synchronized void addChild(FileSystemNode child) {
        children.put(child.name, child);
    }

    public FileSystemNode getChild(String name) {
        return children.get(name);
    }

    // Lock the path from this node to the leaf for performing operations ensuring deadlock avoidance
    public static List<ReentrantLock> lockPath(FileSystemNode root, List<String> path) throws InterruptedException {
        List<ReentrantLock> locks = new ArrayList<>();
        FileSystemNode current = root;

        // Acquire locks top-down
        for (String nodeName : path) {
            current.lock.lock();
            locks.add(current.lock);
            current = current.getChild(nodeName);
            if (current == null) {
                // Release acquired locks on failure
                unlockAll(locks);
                throw new IllegalArgumentException("Invalid path");
            }
        }
        // Lock leaf node as well
        current.lock.lock();
        locks.add(current.lock);

        return locks;
    }

    public static void unlockAll(List<ReentrantLock> locks) {
        for (int i = locks.size() - 1; i >= 0; i--) {
            locks.get(i).unlock();
        }
    }

    // Example usage - read or write operation locked on a path
    public static void performOperation(FileSystemNode root, List<String> path, Runnable operation) throws InterruptedException {
        List<ReentrantLock> locks = lockPath(root, path);
        try {
            operation.run();
        } finally {
            unlockAll(locks);
        }
    }
}
```

**Explanation:**
- Locks are acquired from root to leaf in order.
- If at any point a child node does not exist, already acquired locks are released to prevent inconsistent state.
- Unlocking is done in reverse order after the operation completes.
- This scheme avoids circular waiting and thus deadlock.

## Key follow-up questions

- How would you handle concurrent modifications such as node insertions or deletions?
- How can you optimize for read-heavy workloads? Would read-write locks help?
- What if multiple siblings must be locked? How do you enforce a consistent lock order among them?
- How would you implement lock timeouts or deadlock detection if locks cannot be acquired immediately?
- Can you describe optimistic concurrency control alternatives?
- How would the design change if this system needs to work in distributed settings?

## Takeaways

- Avoid a single, coarse-grained global lock in concurrent tree structures as it kills concurrency.
- Use fine-grained locks and acquire them in strict global order (top-down in tree).
- Implement lock acquisition and release logic carefully to avoid deadlocks.
- Consider read-write locks or optimistic concurrency to improve read performance.
- Think through failure cases: invalid paths, lock acquisition failure, exception safety.
- Deadlock avoidance strategies in hierarchical locking are critical for scalable and safe concurrent file system design.
