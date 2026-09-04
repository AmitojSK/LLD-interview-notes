# Deadlock on the File System LLD Roundtable

> Automatically generated interview-preparation note.

## Original problem

The Google L5 question that destroyed a senior engineer who said "I'll lock the whole tree with one mutex".

## Interview-ready answer

## Problem understanding

Design a file system or a tree-structured data store with hierarchical directories and files, supporting concurrent operations safely and efficiently. The challenge is to design a locking strategy that allows multiple operations (reads/writes) without deadlocks or excessive contention. Naively locking the entire file system tree with a single global mutex is easy but leads to poor concurrency and scalability. A finer-grained locking scheme is needed—such as per-node locks—while avoiding deadlocks caused by cycles in lock acquisition order.

Constraints and design goals:
- Support concurrent operations on files and directories, including reads and writes.
- Avoid deadlocks during concurrent lock acquisitions on multiple nodes.
- Maximize concurrency by allowing unrelated operations to proceed without blocking.
- Keep lock acquisition and release logic manageable and efficient.
- Ensure correctness and consistent view of the file system state.
- Handle potential corner cases like hierarchical lock upgrades, deletes, and moves atomically.

## Interview answer

### Core design

1. **Fine-grained locking**:
   Use locks at the node (file/directory) level instead of a single global lock. This enables multiple operations on different subtrees concurrently.

2. **Lock ordering to prevent deadlock**:
   Deadlock occurs when two threads try to lock nodes in conflicting orders. To avoid deadlock:
   - Impose a global lock acquisition order.
   - The natural order is the path from root downwards.
   - Always acquire locks top-down in the file system hierarchy.
   
3. **Lock modes**:
   - Use **read/write locks** (e.g., ReentrantReadWriteLock) to distinguish read-only vs. write operations.
   - Readers can share locks while writers are exclusive.
   
4. **Operations:**
   - To read or modify a node, first lock the path from root to the target node in **read mode**.
   - To modify a node or its children (add/delete), acquire write locks top-down.
   - For moves or renames (operations affecting multiple nodes), lock all affected nodes in path order.

5. **Lock acquisition protocol**:
   - Always acquire parent locks before child locks.
   - Avoid releasing an acquired lock until all needed locks are held (lock coupling).
   - If unable to acquire a lock, release currently held locks and retry (backoff strategy).

6. **Potential optimizations**:
   - Use lock coupling / hand-over-hand locking to reduce held locks concurrently.
   - Cache parent locks or lock upgrades with attention to concurrency.
   - Deadlock avoidance requires careful analysis of lock acquisition paths.

### Trade-offs and reasoning

- Global lock is simple but serializes all operations.
- Fine-grained locks improve concurrency but introduce complexity.
- Strict lock ordering prevents deadlocks but requires careful lock acquisition logic.
- Read/write locks improve throughput under mostly read workloads.
- Complex operations like moves require locking multiple nodes atomically, which is tricky.
- Backoff and retry add robustness but complicate logic and reduce performance in heavy contention.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.locks.*;

public class FileSystem {
    private final DirectoryNode root;

    public FileSystem() {
        this.root = new DirectoryNode("", null);
    }

    // Abstract node class
    abstract static class Node {
        final String name;
        final DirectoryNode parent;
        final ReadWriteLock lock;

        Node(String name, DirectoryNode parent) {
            this.name = name;
            this.parent = parent;
            this.lock = new ReentrantReadWriteLock(true);
        }

        String getPath() {
            if (parent == null || parent.name.isEmpty()) return "/" + name;
            return parent.getPath() + "/" + name;
        }
    }

    static class FileNode extends Node {
        private String content;

        FileNode(String name, DirectoryNode parent) {
            super(name, parent);
        }

        String readContent() {
            lock.readLock().lock();
            try {
                return content;
            } finally {
                lock.readLock().unlock();
            }
        }

        void writeContent(String content) {
            lock.writeLock().lock();
            try {
                this.content = content;
            } finally {
                lock.writeLock().unlock();
            }
        }
    }

    static class DirectoryNode extends Node {
        private final Map<String, Node> children = new HashMap<>();

        DirectoryNode(String name, DirectoryNode parent) {
            super(name, parent);
        }

        // Add a child under write lock
        void addChild(Node child) {
            lock.writeLock().lock();
            try {
                children.put(child.name, child);
            } finally {
                lock.writeLock().unlock();
            }
        }

        Node getChild(String name) {
            lock.readLock().lock();
            try {
                return children.get(name);
            } finally {
                lock.readLock().unlock();
            }
        }

        void removeChild(String name) {
            lock.writeLock().lock();
            try {
                children.remove(name);
            } finally {
                lock.writeLock().unlock();
            }
        }
    }

    /** 
     * Helper to lock nodes from root to target path in read or write mode.
     * Throws if path does not exist.
     */
    private List<Node> lockPath(String path, boolean write) {
        String[] parts = path.split("/");
        DirectoryNode current = root;
        List<Node> lockedNodes = new ArrayList<>();
        // Lock root first
        (write ? current.lock.writeLock() : current.lock.readLock()).lock();
        lockedNodes.add(current);
        for (int i = 1; i < parts.length; i++) {
            String part = parts[i];
            Node child;
            if (current == null) {
                // Release all acquired locks before returning
                unlockNodes(lockedNodes);
                throw new IllegalArgumentException("Invalid path");
            }
            child = current.getChild(part);
            if (child == null) {
                unlockNodes(lockedNodes);
                throw new IllegalArgumentException("Invalid path: " + path);
            }
            // Lock next node before releasing current to avoid race condition
            (write ? child.lock.writeLock() : child.lock.readLock()).lock();
            lockedNodes.add(child);
            if (child instanceof DirectoryNode) {
                current = (DirectoryNode) child;
            } else if (i < parts.length - 1) {
                // Trying to traverse into a file - invalid path
                unlockNodes(lockedNodes);
                throw new IllegalArgumentException("Invalid path: not a directory");
            }
        }
        return lockedNodes;
    }

    private void unlockNodes(List<Node> nodes) {
        // Unlock in reverse order
        Collections.reverse(nodes);
        for (Node node : nodes) {
            ReentrantReadWriteLock rwLock = (ReentrantReadWriteLock) node.lock;
            if (rwLock.isWriteLockedByCurrentThread()) {
                rwLock.writeLock().unlock();
            } else {
                rwLock.readLock().unlock();
            }
        }
    }

    public String readFile(String path) {
        List<Node> locked = null;
        try {
            locked = lockPath(path, false); // read lock
            Node node = locked.get(locked.size() - 1);
            if (!(node instanceof FileNode)) throw new IllegalArgumentException("Not a file");
            return ((FileNode) node).readContent();
        } finally {
            if (locked != null) unlockNodes(locked);
        }
    }

    public void writeFile(String path, String content) {
        List<Node> locked = null;
        try {
            locked = lockPath(path, true); // write lock
            Node node = locked.get(locked.size() - 1);
            if (!(node instanceof FileNode)) throw new IllegalArgumentException("Not a file");
            ((FileNode) node).writeContent(content);
        } finally {
            if (locked != null) unlockNodes(locked);
        }
    }

    public void createFile(String path) {
        int lastSlash = path.lastIndexOf('/');
        if (lastSlash < 0) throw new IllegalArgumentException("Invalid path");
        String dirPath = path.substring(0, lastSlash);
        String fileName = path.substring(lastSlash + 1);
        List<Node> locked = null;
        try {
            locked = lockPath(dirPath.isEmpty() ? "/" : dirPath, true); // write lock on parent dir
            Node parent = locked.get(locked.size() - 1);
            if (!(parent instanceof DirectoryNode)) throw new IllegalArgumentException("Parent not directory");
            DirectoryNode dir = (DirectoryNode) parent;
            if (dir.getChild(fileName) != null) throw new IllegalArgumentException("File exists");
            dir.addChild(new FileNode(fileName, dir));
        } finally {
            if (locked != null) unlockNodes(locked);
        }
    }

    // Similar methods for delete, move etc can follow the same pattern.
}
```

### Explanation

- Each `Node` has its own `ReadWriteLock`.
- Locks are acquired top-down on the path from root.
- Lock acquisition follows consistent order to prevent deadlocks.
- Unlocking happens in reverse order.
- Methods throw exceptions on invalid operations or path errors.
- This design allows concurrent operations on different parts of tree.
- Write locks are exclusive, read locks are sharable.
- The implementation is simplified and can be extended with retries/backoff for contention.

## Key follow-up questions

1. **Question: How do you prevent deadlocks when multiple concurrent operations lock overlapping nodes?**  
   Answer: By enforcing a strict global lock acquisition order (top-down traversal order) and never locking child nodes before parents. This prevents cycles in the wait-for graph.

2. **Question: How would you handle rename or move operations that need to lock multiple unrelated nodes?**  
   Answer: Acquire locks on all affected nodes in a globally consistent order—typically lexicographically sorted paths—to avoid circular wait and deadlock.

3. **Question: What is the impact of lock granularity on performance?**  
   Answer: Finer granularity increases concurrency but complicates locking logic. Coarser granularity is simpler but reduces parallelism. Optimal granularity balances concurrency and complexity.

4. **Question: How would you implement lock upgrades (read to write) to avoid deadlocks?**  
   Answer: Lock upgrading can cause deadlocks if not designed carefully. One strategy is to release the read lock before acquiring the write lock, possibly with retries, or design the API to acquire write locks from the start if modification is anticipated.

5. **Question: How does this design behave under high contention?**  
   Answer: High contention on the same nodes serializes access, reducing throughput. Retrying after failed lock attempts and backoff can reduce livelock. Monitoring hotspots can help optimize.

6. **Question: Could you use optimistic concurrency control instead of locks?**  
   Answer: Yes, for some workloads and read-heavy scenarios, optimistic concurrency with version checks and retries can improve throughput but complicates failure handling.

## Takeaways

- Avoiding deadlocks in hierarchical data structures requires strict lock ordering.
- Fine-grained locking with per-node locks and read/write locks enables better concurrency.
- Lock acquisition and release order have to be carefully designed to prevent cycles.
- Write operations require exclusive locks, while reads can be concurrent.
- Complex operations like move/rename require careful multi-node locking protocols.
- Robust error handling and retry mechanisms improve reliability.
- Trade-offs between complexity and performance must be weighed carefully.
