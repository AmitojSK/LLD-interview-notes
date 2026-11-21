# Deadlock on the File System LLD Roundtable

> Automatically generated interview-preparation note.

## Original problem

The Google L5 question that destroyed a senior engineer who said "I'll lock the whole tree with one mutex".

## Interview-ready answer

## Problem understanding

The problem involves designing a file system layer (or similar hierarchical structure) that supports concurrent access and modifications. The key challenge is to manage locks for a tree of directories and files without causing deadlocks or performance bottlenecks. A naive solution—locking the entire tree with a single global mutex—ensures correctness but severely hurts concurrency and scalability, often unacceptable in large systems.

Design goals:
- Correctness under concurrent modifications and traversals.
- Avoid deadlocks when multiple threads acquire locks on different nodes or subtrees.
- Maximize concurrency allowing parallel operations on independent parts of the file system.
- Reasonable locking granularity balancing complexity and performance.

Constraints:
- The file system is hierarchical, a tree structure where directories can contain files and other directories.
- Operations require locking a path or subtree, e.g., create, move, read, delete.
- Deadlocks arise when multiple threads lock different nodes in conflicting orders.

## Interview answer

### Core design

1. **Avoid single global lock**: A global mutex serializes everything, hurting concurrency and likely unreasonable for large trees.

2. **Hierarchical locking with strict ordering**: Locks should be acquired from the root down (or in a consistent order) to prevent cycles in wait-for graphs (deadlocks).
   - For example, always lock nodes from ancestor to descendant.
   - Release locks as soon as possible, ideally after the critical section per node.

3. **Fine-grained locking per node**:
   - Each node (file/directory) holds its own lock.
   - Operations traverse the tree, acquiring locks in a top-down manner.
   - This allows disjoint parts of the tree to be accessed concurrently.

4. **Use lock coupling / hand-over-hand locking**:
   - Acquire lock on the current node.
   - Then acquire lock on the child node before releasing the parent lock.
   - This preserves consistency when traversing the tree.

5. **Lock types for read/write concurrency**:
   - Use read-write locks (ReentrantReadWriteLock) to allow concurrent readers.
   - Writers require exclusive locks.

6. **Deadlock prevention**:
   - Always acquire locks in a strict global order (e.g., based on path hierarchy).
   - Avoid locking nodes arbitrarily or after locking other nodes lower in the hierarchy.
   - Avoid lock escalation patterns that break this ordering.

### Trade-offs

- **Lock overhead vs concurrency**: Fine-grained locks increase complexity but improve concurrency.
- **Complexity**: Hand-over-hand locking is tricky to get right and requires careful programming.
- **Performance**: More locking operations increase overhead — careful optimization needed.

### Example use cases

- File create/delete require locking parent directory exclusively to prevent conflicting modifications.
- Reading file content needs read lock on the file node.
- Moving files/directories require locking both source and destination paths carefully to avoid deadlocks.

## Java implementation

Below is a simplified example showing a file system tree node with per-node ReentrantReadWriteLock and a hand-over-hand traversal for locking.

```java
import java.util.*;
import java.util.concurrent.locks.*;

public class FileSystemNode {
    private final String name;
    private final boolean isFile;
    private final Map<String, FileSystemNode> children;
    
    // Use a ReadWriteLock to support concurrent readers.
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    private FileSystemNode parent;

    public FileSystemNode(String name, boolean isFile) {
        this.name = name;
        this.isFile = isFile;
        this.children = isFile ? null : new HashMap<>();
    }

    // Acquire read lock for current node
    public void readLock() {
        lock.readLock().lock();
    }

    public void readUnlock() {
        lock.readLock().unlock();
    }

    // Acquire write lock for current node
    public void writeLock() {
        lock.writeLock().lock();
    }

    public void writeUnlock() {
        lock.writeLock().unlock();
    }

    public boolean isFile() {
        return isFile;
    }

    public FileSystemNode getChild(String name) {
        if (isFile) {
            return null;
        }
        return children.get(name);
    }

    // Add child (requires write lock held)
    public void addChild(FileSystemNode child) {
        if (isFile) throw new UnsupportedOperationException("File cannot have children");
        children.put(child.name, child);
        child.parent = this;
    }

    public String getName() {
        return name;
    }

    public FileSystemNode getParent() {
        return parent;
    }

    // Traverse path locking hand-over-hand: Acquire parent lock first, then child, etc.
    // Returns the node at the end of the path, or null if not found.
    public static FileSystemNode traverseAndLock(FileSystemNode root, List<String> path, boolean write) {
        FileSystemNode current = root;
        // Lock root first
        if (write) current.writeLock();
        else current.readLock();

        for (int i = 0; i < path.size(); i++) {
            String part = path.get(i);

            // Lock child before releasing current to hold hand-over-hand locking
            FileSystemNode nextNode;
            // Always lock current while finding child to avoid race
            if (write) current.writeLock();
            else current.readLock();

            try {
                nextNode = current.getChild(part);
                if (nextNode == null) {
                    // Unlock current and return null
                    if (write) current.writeUnlock();
                    else current.readUnlock();
                    return null;
                }
            } finally {
                // Unlock current will be done *after* locking the child, so we delay it below
            }

            // Lock child
            if (write) nextNode.writeLock();
            else nextNode.readLock();

            // Unlock parent after locking child
            if (write) current.writeUnlock();
            else current.readUnlock();

            current = nextNode;
        }

        // Return the leaf node with the appropriate lock held
        return current;
    }

    // Unlock node (write or read)
    public void unlock(boolean write) {
        if (write) writeUnlock();
        else readUnlock();
    }
}
```

**Notes on this implementation**:
- The `traverseAndLock` method acquires locks from root down to target in hand-over-hand fashion.
- Locks are held on the final node to allow safe operation.
- Caller is responsible for unlocking the node returned.
- For simplicity, this implementation doesn't cover unlocking the whole chain or multiple nodes at once.
- Deadlocks are prevented by consistent top-down locking order.
- Write locks are exclusive; read locks permit concurrent readers.

## Key follow-up questions

1. **Q: How do you handle move/rename operations that involve locking two different paths?**

   **A:** Always lock the two paths in a globally consistent order (e.g., lexicographically by path string or based on hierarchy index) to avoid deadlocks. For example, lock the path with the "smaller" path first, then the other. Also, carefully release locks after the operation.

2. **Q: What if a node is deleted or renamed while another thread holds a lock on it?**

   **A:** The locks prevent such unsafe actions; for example, to delete a node, you must hold the parent's write lock. Before deleting, confirm the node is in the parent's children map. Reference integrity and lock ordering avoid races. If needed, a version or generation number can be added to detect stale references.

3. **Q: Would optimistic locking or lock-free algorithms help here?**

   **A:** They can improve concurrency but add complexity. Optimistic locking (version checks) could be combined with locking for short modifications. Lock-free trees exist but are complex and less common in file system metadata due to complexity and rarity of conflicts.

4. **Q: How do you handle failure modes like exceptions or thread interruptions while holding locks?**

   **A:** Always use try-finally blocks to release locks reliably. If an exception occurs, release held locks to avoid deadlocks. Also, design operations to be idempotent or consistent if partially completed.

5. **Q: How would you extend this design to scale to distributed file systems?**

   **A:** Centralized locking doesn't scale. Use distributed lock managers or consensus protocols (e.g., Paxos, Raft) to coordinate locks. Caching and lease-based locks can improve performance at the cost of complexity and potential split-brain scenarios.

## Takeaways

- Naive coarse locks (single global lock) eliminate concurrency and hurt performance.
- Deadlocks arise from inconsistent lock acquisition order — always lock top-down in hierarchical structures to prevent them.
- Hand-over-hand (lock coupling) locking helps maintain consistency and prevents races during traversal.
- Using per-node read-write locks balances concurrency for reads versus writes.
- Careful ordering, error handling, and unlocking protocols are essential for correctness.
- Designing locking strategy in hierarchical data structures is a classic system design problem requiring attention to granularity, ordering, and concurrency trade-offs.
