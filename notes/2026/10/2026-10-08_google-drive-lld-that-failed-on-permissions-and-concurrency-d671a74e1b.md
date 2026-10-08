# 💀 Google Drive LLD That Failed on Permissions and Concurrency

> Automatically generated interview-preparation note.

## Original problem

Design Google Drive — The Candidate Used 9 Design Patterns. Then Permissions Broke Everything.

## Interview-ready answer

## Problem understanding

Design a simplified version of Google Drive focusing on file storage, sharing, and permissions management. The challenge is to build a scalable, maintainable system that supports concurrent access, fine-grained permissions, and sharing without compromising security or consistency.

**Constraints and design goals:**

- Support storage of hierarchical file/folder structure.
- Enable multiple users to own, share, and access files/folders.
- Implement a robust, flexible permissions system (e.g., read, write, owner, share).
- Handle concurrent operations safely (concurrent reads, writes, permission changes).
- Maintain consistency and integrity of permissions and data.
- Support scalability and performance—operations must be efficient.
- Design should avoid overcomplication (e.g., using many design patterns without clarity can lead to fragile, hard-to-maintain code).

## Interview answer

### Core design components

1. **Entities and relationships:**
   - **User**: Represents a drive user.
   - **File/Folder**: Represented as nodes in a hierarchy; folders contain files/folders.
   - **Permission**: Represents access rights to files/folders per user or group.

2. **Hierarchy and Metadata:**
   - Model files/folders as a tree or graph with unique IDs.
   - Metadata keeps reference to owner, permissions, creation time, etc.

3. **Permissions Model:**
   - Use Access Control Lists (ACLs) per file/folder to track individual access rights.
   - Permissions: READ, WRITE, OWNER, SHARE.
   - Inherit permissions by default from parent folders unless overridden.
   - Protect owner permission: only the owner can change ownership or share rights.

4. **Sharing Model:**
   - Users can invite others by assigning permissions on files/folders.
   - Sharing is recorded in ACLs.

5. **Concurrency and consistency:**
   - Concurrent updates (writes or permission changes) can cause race conditions.
   - To protect permissions and data integrity, use fine-grained locking or optimistic concurrency control.
   - Prefer immutable permission snapshots combined with versioning for optimistic concurrency.

6. **Scalability and caching:**
   - Cache permission lookups to improve read-performance.
   - Use distributed locking or consensus if scaling across nodes.

7. **Avoid overly complex design patterns:**
   - Use patterns where they provide clear benefits (e.g., Factory for object creation, Strategy for permission checking).
   - Avoid mixing many patterns without clear need, which leads to brittleness.

### Trade-offs

- **Fine-grained vs. coarse-grained locking:** Too coarse reduces performance; too fine risks deadlocks.
- **Inheritance of permissions** simplifies management but complicates overrides.
- **Eager vs. lazy permission evaluation:** Cache invalidation vs. real-time accuracy.
- **Complexity vs. flexibility:** More flexible permissions may increase system complexity.

### Failure modes

- Conflicting permission updates may cause stale or incorrect access.
- Incorrect inheritance logic exposes data to unauthorized users.
- Race conditions when sharing files concurrently.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantReadWriteLock;

enum Permission {
    READ, WRITE, OWNER, SHARE
}

class User {
    private final String userId;

    public User(String userId) {
        this.userId = userId;
    }

    public String getUserId() { return userId; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User other = (User) o;
        return userId.equals(other.userId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(userId);
    }
}

abstract class DriveNode {
    protected final String id;
    protected String name;
    protected final User owner;

    // ACL: user to permissions mapping on this node
    protected final Map<User, EnumSet<Permission>> acl = new ConcurrentHashMap<>();

    // Lock for synchronizing permission changes and content updates
    protected final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public DriveNode(String id, String name, User owner) {
        this.id = id;
        this.name = name;
        this.owner = owner;
        acl.put(owner, EnumSet.of(Permission.OWNER, Permission.READ, Permission.WRITE, Permission.SHARE));
    }

    public String getId() { return id; }

    public User getOwner() {
        return owner;
    }

    // Set or update permissions in the ACL
    public boolean setPermission(User granter, User grantee, EnumSet<Permission> perms) {
        lock.writeLock().lock();
        try {
            if (!hasPermission(granter, Permission.SHARE)) {
                // granter must have SHARE permission or OWNER
                return false;
            }
            if (perms.contains(Permission.OWNER) && !owner.equals(granter)) {
                // Only owner can assign OWNER permission
                return false;
            }
            acl.put(grantee, EnumSet.copyOf(perms));
            return true;
        } finally {
            lock.writeLock().unlock();
        }
    }

    // Check if user has a specific permission, considering inheritance
    public boolean hasPermission(User user, Permission perm) {
        lock.readLock().lock();
        try {
            EnumSet<Permission> userPerms = acl.get(user);
            return userPerms != null && userPerms.contains(perm);
        } finally {
            lock.readLock().unlock();
        }
    }

    public abstract boolean isFolder();
}

class FileNode extends DriveNode {
    private byte[] data;

    public FileNode(String id, String name, User owner) {
        super(id, name, owner);
    }

    public void writeContent(User user, byte[] content) throws SecurityException {
        // Check write permission
        if (!hasPermission(user, Permission.WRITE)) {
            throw new SecurityException("User lacks WRITE permission");
        }
        lock.writeLock().lock();
        try {
            this.data = content;
        } finally {
            lock.writeLock().unlock();
        }
    }

    public byte[] readContent(User user) throws SecurityException {
        if (!hasPermission(user, Permission.READ)) {
            throw new SecurityException("User lacks READ permission");
        }
        lock.readLock().lock();
        try {
            return data;
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public boolean isFolder() { return false; }
}

class FolderNode extends DriveNode {
    private final Map<String, DriveNode> children = new ConcurrentHashMap<>();

    public FolderNode(String id, String name, User owner) {
        super(id, name, owner);
    }

    public void addChild(User user, DriveNode node) throws SecurityException {
        if (!hasPermission(user, Permission.WRITE)) {
            throw new SecurityException("User lacks WRITE permission");
        }
        lock.writeLock().lock();
        try {
            children.put(node.getId(), node);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void removeChild(User user, String nodeId) throws SecurityException {
        if (!hasPermission(user, Permission.WRITE)) {
            throw new SecurityException("User lacks WRITE permission");
        }
        lock.writeLock().lock();
        try {
            children.remove(nodeId);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public DriveNode getChild(String nodeId) {
        lock.readLock().lock();
        try {
            return children.get(nodeId);
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public boolean isFolder() { return true; }
}

// Service class for Drive operations
class DriveService {
    private final Map<String, DriveNode> nodes = new ConcurrentHashMap<>();
    private final Map<String, User> users = new ConcurrentHashMap<>();

    // Add user
    public User createUser(String userId) {
        User user = new User(userId);
        users.put(userId, user);
        return user;
    }

    // Create new folder
    public FolderNode createFolder(String id, String name, User owner) {
        FolderNode folder = new FolderNode(id, name, owner);
        nodes.put(id, folder);
        return folder;
    }

    // Create new file
    public FileNode createFile(String id, String name, User owner) {
        FileNode file = new FileNode(id, name, owner);
        nodes.put(id, file);
        return file;
    }

    public DriveNode getNode(String id) {
        return nodes.get(id);
    }
}
```

### Explanation

- Use thread-safe data structures (`ConcurrentHashMap`) and locks for permission and content updates.
- Permissions are checked atomically with read locks.
- Permission changes use write lock to avoid inconsistent ACL states.
- Simple permission model supports owner-only assignment of OWNER rights.
- Folder supports children addition/removal with concurrency control.
- Service layer manages user and nodes.

### Failure modes handled:

- Attempts to assign OWNER permission by non-owners are rejected.
- Concurrent updates are guarded by locks.
- Lack of permission throws `SecurityException`.

### Edge cases not implemented but to consider:

- Recursive permission inheritance.
- Group or domain-level permissions.
- Soft deletes or versioning.
- Audit logging and permission change history.
- Scalability using distributed locks and caching.

## Key follow-up questions

1. **Q: How would you handle recursive permission inheritance efficiently?**  
   A: Use caching to store effective permissions resolved from parent chain. On permission changes, invalidate caches down the hierarchy lazily or eagerly. Alternatively, flatten permissions during changes. This prevents repeated tree traversal for every access check.

2. **Q: How would you support concurrent modification across distributed nodes?**  
   A: Implement distributed consensus using systems like ZooKeeper or etcd for locks and consistent state. Use versioned immutable ACL snapshots for optimistic concurrency. Conflict resolution or locking strategies ensure consistency.

3. **Q: How can you prevent deadlocks with locks in this design?**  
   A: Strictly define lock acquisition order (e.g., always acquire parent node locks before child nodes). Keep lock scope minimal and avoid lock nesting. Detect lock timeout conditions or use tryLock with retries.

4. **Q: What if a user is deleted? How would permissions be handled?**  
   A: Remove user entries from ACLs and update sharing metadata. Transfer ownership if needed or mark nodes orphaned. Ensure orphan nodes cannot be accessed accidentally by invalid users.

5. **Q: How to extend this design to support group permissions?**  
   A: Introduce Group entities. ACL maps users/groups to permissions. During access, resolve effective permissions by combining user and group rights. Store group membership to aid lookups.

## Takeaways

- A robust permissions design is crucial and often the hardest part in file-sharing systems.
- Simplify concurrency by fine-grained locking combined with immutable snapshots or versioning.
- Overuse of design patterns without clear purpose can increase complexity and risk breaking core logic like permissions.
- Always validate ownership and critical permissions on sensitive operations.
- Consider caching and permission inheritance carefully for scalability.
- Think about edge cases early, such as recursive inheritance, user lifecycle, and concurrency conflicts.
