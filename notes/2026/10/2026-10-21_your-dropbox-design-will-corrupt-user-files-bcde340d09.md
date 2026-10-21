# Your Dropbox Design Will Corrupt User Files.

> Automatically generated interview-preparation note.

## Original problem

Two users edit the same file simultaneously from different devices. One device comes online late. Which version wins?

## Interview-ready answer

## Problem understanding
The problem involves designing a system to handle concurrent edits on the same file by multiple users or devices. Key challenges include:

- Concurrent modifications to the same file from different endpoints.
- Handling out-of-order updates (e.g., a device coming online late after offline edits).
- Ensuring no user changes are overwritten or lost unknowingly.
- Maintaining consistency and providing conflict resolution.
- Constraints may include network latency, offline usage, and eventual consistency.

The design goal is to prevent file corruption due to conflicting edits and provide a predictable model for resolving or merging differences.

## Interview answer
When multiple users or devices concurrently modify the same file, the system must handle conflicts carefully to prevent data loss.

### Core design considerations:
1. **Versioning and Metadata**  
   Every edit should produce a new version with metadata like timestamps, device IDs, and causal history.

2. **Conflict detection**  
   Detect if two versions diverge from a common ancestor, indicating a conflict.

3. **Conflict resolution strategies:**

   - **Last-Writer-Wins (LWW)**: Simple but loses changes if updates arrive out-of-order.
   - **Operational Transformation (OT)** or **CRDTs (Conflict-free Replicated Data Types)**: Automatically merge changes without user intervention.
   - **User-driven merge/conflict resolution UI**: Notify users to resolve conflicts manually.

4. **Offline/late updates handling**
   - If a device comes online late, and its edit conflicts, the system flags a conflict rather than blindly overwriting newer versions.
   - Store a DAG (Directed Acyclic Graph) of versions to track branching histories.

### Trade-offs
- Simple LWW is easy to implement but risks data loss.
- OT and CRDTs are complex but enable seamless real-time collaboration.
- User-driven merges increase complexity for users but preserve data integrity.
- Storing version histories increases storage needs.

### Proposed approach
- Use version vectors or causal metadata to track edits.
- When a late update arrives, compare history:
  - If it’s a direct descendant, accept it.
  - If it conflicts (diverged fork), store both versions and notify users of conflicts.
- Provide a UI or API for manual merge or accept one version.
  
This balances automated conflict detection and user-driven resolution to avoid silent file corruption.

## Java implementation

```java
import java.util.*;

public class FileVersion {
    private final String fileId;
    private final String versionId;
    private final String content;
    private final Map<String, Integer> versionVector; // deviceId -> version count

    public FileVersion(String fileId, String versionId, String content, Map<String, Integer> versionVector) {
        this.fileId = fileId;
        this.versionId = versionId;
        this.content = content;
        this.versionVector = new HashMap<>(versionVector);
    }

    public String getFileId() { return fileId; }
    public String getVersionId() { return versionId; }
    public String getContent() { return content; }
    public Map<String, Integer> getVersionVector() { return versionVector; }

    /**
     * Determine if this version causally descends from another.
     */
    public boolean isDescendantOf(FileVersion other) {
        Map<String, Integer> otherVV = other.getVersionVector();
        for (Map.Entry<String, Integer> entry : otherVV.entrySet()) {
            String device = entry.getKey();
            int otherCount = entry.getValue();
            int thisCount = versionVector.getOrDefault(device, 0);
            if (thisCount < otherCount) {
                return false;
            }
        }
        return true;
    }

    /**
     * Detect if two versions conflict.
     * Conflict if neither is descendant of other.
     */
    public boolean conflictsWith(FileVersion other) {
        return !(this.isDescendantOf(other) || other.isDescendantOf(this));
    }
}

public class FileVersionStore {
    private final Map<String, List<FileVersion>> fileVersions = new HashMap<>();

    /**
     * Add a new version, handle conflicts
     */
    public synchronized void addVersion(FileVersion newVersion) {
        List<FileVersion> versions = fileVersions.getOrDefault(newVersion.getFileId(), new ArrayList<>());

        // Check conflicts with existing versions
        for (FileVersion v : versions) {
            if (newVersion.conflictsWith(v)) {
                // Conflict detected
                System.out.println("Conflict detected between versions " + newVersion.getVersionId() + " and " + v.getVersionId());
                // Simple policy: keep both versions until user merges
                versions.add(newVersion);
                fileVersions.put(newVersion.getFileId(), versions);
                return;
            }
        }

        // No conflict, prune older versions that are ancestors of newVersion
        versions.removeIf(v -> newVersion.isDescendantOf(v));
        versions.add(newVersion);
        fileVersions.put(newVersion.getFileId(), versions);
    }

    public synchronized List<FileVersion> getVersions(String fileId) {
        return new ArrayList<>(fileVersions.getOrDefault(fileId, new ArrayList<>()));
    }
}
```

**Explanation:**

- We use **version vectors** to track causal history across devices.
- The method `isDescendantOf` determines if one version is a descendant of another by comparing version vectors.
- Conflicts are detected when two versions don't have a causal relationship.
- The `FileVersionStore` stores multiple parallel versions (branches) when conflicts arise.
- This enables tracking offline edits and late coming devices' versions without overwriting.
- Users or higher-level logic can later merge conflicting versions.

### Concurrency and Failure modes
- The store uses `synchronized` methods for thread safety in this simple example.
- In a distributed real system, conflict detection should happen at sync time, with durable version history.
- Network partitions may delay conflict detection.
- Storage of multiple versions increases complexity for syncing and UI.

## Key follow-up questions

1. **Q: How would you handle automatic merging of conflicts?**  
   A: Implement OT or CRDT algorithms that transform or combine concurrent edits seamlessly. For text files, operational transforms can merge insert/delete operations. For binary files, domain-specific merge or chunk-level strategies may be needed.

2. **Q: What if devices have unsynced clocks affecting version timestamps?**  
   A: Avoid relying on physical timestamps for conflict resolution alone. Use logical clocks (like version vectors, Lamport clocks) that reflect causal order rather than absolute times.

3. **Q: How do you scale this design for millions of files and users?**  
   A: Use a distributed database or cloud storage backing. Maintain version metadata efficiently with vector clocks or compact DAG structures. Sync incremental changes rather than full versions. Employ sharding and caching.

4. **Q: Can this approach guarantee strong consistency?**  
   A: No; it provides eventual consistency with conflict detection. Strong consistency demands locking or single-source updates, limiting availability and offline editing.

5. **Q: How do you deal with malicious or corrupted edits?**  
   A: Incorporate authentication and authorization, validate file integrity (hashes), and include audit trails. Conflicted or suspicious edits can be flagged for review.

## Takeaways
- Concurrent edits require causal metadata like version vectors to detect conflicts.
- Simple last-writer-wins solutions are prone to silent data loss when devices update offline or late.
- Tracking multiple versions and notifying users of conflicts prevents file corruption.
- Advanced techniques like OT/CRDT enable seamless merging but increase system complexity.
- Offline edits and late arrivals require careful handling of version histories rather than overwriting blindly.
- Designing robust file sync systems balances consistency, availability, complexity, and user experience.
