# The Dropbox LLD Answer Looked Perfect…

> Automatically generated interview-preparation note.

## Original problem

What happens when two users edit the same file simultaneously from different devices?

## Interview-ready answer

## Problem understanding

The problem is to design a system component that handles concurrent edits to the same file by multiple users or devices, such as in Dropbox or other collaborative cloud storage services. Key constraints include:

- Ensuring data consistency despite multiple concurrent modifications.
- Minimizing data loss and providing a seamless user experience.
- Handling network latencies and device offline modes.
- Supporting eventual synchronization across multiple clients and the server.
- Providing conflict resolution either automatically or via user intervention.

Design goals:
- Enable concurrent edits with minimal user friction.
- Ensure the final file version accurately reflects the merged changes or highlights conflicts.
- Provide clear mechanisms for conflict detection and resolution.
- Maintain scalability and responsiveness for millions of users.

---

## Interview answer

When two users edit the same file simultaneously from different devices, the system must detect these concurrent modifications and reconcile them without losing data.

**Core Design:**

1. **Versioning:** Each file or file chunk maintains a version identifier (e.g., a vector clock or a simple monotonic version number) to track changes.

2. **Edit Detection:** The server or sync engine detects when multiple edits occur on the same file version before synchronization.

3. **Conflict Detection:** If the server receives an update based on an outdated version, it flags a conflict.

4. **Conflict Resolution Strategies:**
   - **Last Write Wins (LWW):** Simplest approach, where the latest edit overwrites previous ones. This is prone to data loss and usually insufficient.
   - **File Merging:** For text files, merge changes using Operational Transform (OT) or Conflict-free Replicated Data Types (CRDTs) to automatically merge concurrent changes.
   - **Conflict Copies:** For binary or non-mergeable files, create separate conflict copies for manual resolution.
   - **User Intervention:** Notify users about conflicts with options to resolve manually.

5. **Offline Support:** Each device maintains a local copy and syncs when online. Conflict detection then occurs at the next sync.

6. **Atomic Sync Operations:** Ensure that updates to a file are atomic to prevent partial writes.

**Trade-offs:**

- OT and CRDT allow seamless merges but increase system complexity.
- LWW is simpler but risks overwriting important changes.
- Conflict copies preserve data but can confuse users and clutter storage.

---

## Java implementation

Here is a simplified Java model capturing the core logic of conflict detection and resolution during file edits. This example focuses on versioning and conflict detection. Merging logic is omitted due to complexity.

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class FileSyncService {

    // Represents a single file version record
    static class FileVersion {
        String fileId;
        int version;
        byte[] content;

        FileVersion(String fileId, int version, byte[] content) {
            this.fileId = fileId;
            this.version = version;
            this.content = content;
        }

        // Deep copy for creating conflict copies if needed
        FileVersion copy() {
            return new FileVersion(fileId, version, Arrays.copyOf(content, content.length));
        }
    }

    // Store the latest version per file
    private final Map<String, FileVersion> fileStore = new ConcurrentHashMap<>();

    // Client sends their edited file version
    public SyncResult syncFile(String fileId, int clientVersion, byte[] newContent) {
        FileVersion current = fileStore.get(fileId);

        if (current == null) {
            // New file, accept the upload, version starts at 1
            FileVersion newVersion = new FileVersion(fileId, 1, newContent);
            fileStore.put(fileId, newVersion);
            return new SyncResult(true, newVersion.version, null);
        }

        if (clientVersion == current.version) {
            // No conflict: accept the new version and increment version
            FileVersion newVersion = new FileVersion(fileId, current.version + 1, newContent);
            fileStore.put(fileId, newVersion);
            return new SyncResult(true, newVersion.version, null);
        }

        // Conflict detected: concurrent edits based on older version
        // Option 1: Signal conflict to client for manual resolution
        // Option 2: Create "conflict copy" as new file version (illustrative)
        String conflictFileId = fileId + "_conflict_" + UUID.randomUUID();
        FileVersion conflictCopy = new FileVersion(conflictFileId, 1, newContent);
        fileStore.put(conflictFileId, conflictCopy);

        return new SyncResult(false, current.version, conflictFileId);
    }

    /**
     * Result of sync attempt.
     * success = true means client update was accepted.
     * conflictFileId indicates ID of conflict copy if conflict occurred.
     */
    public static class SyncResult {
        public final boolean success;
        public final int serverVersion;
        public final String conflictFileId;

        public SyncResult(boolean success, int serverVersion, String conflictFileId) {
            this.success = success;
            this.serverVersion = serverVersion;
            this.conflictFileId = conflictFileId;
        }
    }
}
```

**Explanation:**

- Server stores the latest file version with a version number.
- Client submits sync requests with their version number and content.
- If the client's version matches the server version, the update is accepted and version incremented.
- If client version is older (due to concurrent edits), conflict is detected.
- The system returns conflict info; a conflict copy is created for manual resolution.
- This model represents core versioning and conflict detection logic.
  
**Concurrency & thread-safety:**  
- Uses `ConcurrentHashMap` for file storage to handle concurrent sync requests.
- Atomic updates to the map ensure latest versions are consistent.
- Implementations in production might use distributed locks or transactions for strong consistency and recovery.

**Edge cases and failure modes:**

- Network partitions or offline edits cause delayed synchronization and possible conflicts.
- Large files might need chunk-wise synchronization.
- Complex file merges (e.g., OT or CRDT) require additional protocols.
- Storage of conflict copies might lead to clutter unless cleaned up by user or system.

---

## Key follow-up questions

1. **Q: How would you handle conflict resolution for text files edited concurrently?**  
   A: Use Operational Transforms (OT) or CRDTs to merge concurrent changes automatically, preserving intention and enabling real-time collaboration. This requires sophisticated algorithms and metadata tracking edits at a fine-grained level.

2. **Q: How can the system support offline edits and synchronization once devices reconnect?**  
   A: Each device maintains a local copy and versioning metadata. When reconnecting, the client syncs updates, which the server merges or flags as conflicts based on version comparison. Offline changes may generate conflicts needing user intervention.

3. **Q: What if the file is large? How to optimize sync?**  
   A: Use chunked sync and delta encoding—only changed parts of the file are sent and merged. This reduces bandwidth and latency.

4. **Q: Could conflicts be resolved automatically instead of creating copies?**  
   A: For text files, yes, through merging algorithms. For binary files (e.g., images), automatic merging is typically infeasible, so copy creation with user intervention is safer.

5. **Q: How would you avoid the "last write wins" problem causing silent data loss?**  
   A: Avoid blind overwrites by version checking and conflict detection. Use user notifications, conflict copies, or merging strategies to preserve all edits.

---

## Takeaways

- Simultaneous edits require versioning and conflict detection to maintain consistency.
- Simple approaches like "last write wins" are easy but risky.
- More complex methods like OT and CRDT enable real-time collaboration at increased complexity.
- Conflict copies provide a fallback but increase user burden.
- Support for offline mode necessitates robust synchronization protocols.
- Designing effective concurrent editing systems balances correctness, user experience, performance, and implementation complexity.
