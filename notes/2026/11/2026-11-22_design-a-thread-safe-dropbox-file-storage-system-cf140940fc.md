# 📦 "Design a Thread-Safe Dropbox File Storage System"

> Automatically generated interview-preparation note.

## Original problem

The Dropbox L5 question that destroyed a senior engineer who said "I'll just store the whole file as one blob".

## Interview-ready answer

## Problem understanding

Design a thread-safe file storage system similar to Dropbox that supports storing files efficiently and safely with concurrent access in mind. The core constraints and requirements include:

- **Thread Safety:** Multiple threads may upload or retrieve files concurrently. The system must handle concurrent reads/writes without corrupting state.
- **Efficient Storage:** Storing a whole file as a single blob is simple but may not scale or support partial updates, deduplication, or efficient retrieval.
- **File Metadata Management:** Track file metadata, versions, and possibly partial file chunks or updates.
- **Consistency & Atomicity:** Uploads or updates should be atomic to avoid corrupted or partial files being visible to readers.
- **Scalability & Extensibility:** The design should allow scaling with large file sizes and many users.

## Interview answer

### Core design

A robust Dropbox-like backend system should not treat each file as a single monolithic blob. Instead, it should:

1. **Chunk files into smaller blocks:** Splitting files into fixed-size chunks enables:
    - Efficient storage and retrieval, even partial file reads.
    - Deduplication by storing identical chunks once.
    - Parallel uploads/downloads.
  
2. **Maintain metadata and chunks index:**
    - A **FileMetadata** class holds file version, file size, and a list (or ordered map) of chunk references.
    - Chunk references point to the stored data chunks.
  
3. **Concurrency control:**
    - Use **synchronized blocks** or higher-level concurrency utilities (like `ReadWriteLock`) to protect metadata updates.
    - Ensure atomic updates to metadata and chunk storage to avoid readers seeing inconsistent versions.
  
4. **Storage abstraction:**
    - Separate storage for metadata (e.g., in-memory concurrent map or persistent DB).
    - Chunk storage layer that stores/retrieves chunk data blobs by chunk ID or hash.

5. **Write and read operations:**
    - Write (upload) operation chunks the file data, uploads chunks, then atomically updates metadata referencing all chunks.
    - Read operation reads metadata first, then fetches chunks in sequence.

### Trade-offs

- Splitting files into chunks adds complexity but improves scalability and deduplication.
- Locking can reduce throughput if coarse-grained; fine-grained locks per file or chunk references are better.
- Storing metadata and chunks can be split physically (e.g., DB vs object store), which improves scalability.

### Reasoning

Storing whole files as single blobs causes:

- Large memory or I/O spikes for big files.
- No support for partial updates/downloads.
- Difficulties with deduplication or concurrent modifications.
  
Chunk-based design balances complexity with real-world Dropbox-like feature needs.

## Java implementation

Below is a simplified thread-safe implementation focusing on in-memory metadata and chunk storage simulation.

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.*;

public class DropboxFileStorage {
    private static final int CHUNK_SIZE = 1024 * 1024; // 1MB

    // Stores chunkId -> chunkData
    private final ConcurrentHashMap<String, byte[]> chunkStorage = new ConcurrentHashMap<>();

    // Stores fileId -> FileMetadata
    private final ConcurrentHashMap<String, FileMetadata> metadataStorage = new ConcurrentHashMap<>();

    // Lock to protect metadata updates per fileId
    private final ConcurrentHashMap<String, ReadWriteLock> fileLocks = new ConcurrentHashMap<>();

    // Represents metadata of a file including versioning and chunk references
    private static class FileMetadata {
        private final String fileId;
        private final int version;
        private final List<String> chunkIds; // ordered chunk references
        private final long size;

        FileMetadata(String fileId, int version, List<String> chunkIds, long size) {
            this.fileId = fileId;
            this.version = version;
            this.chunkIds = Collections.unmodifiableList(new ArrayList<>(chunkIds));
            this.size = size;
        }
    }

    // Utility to get or create lock per fileId
    private ReadWriteLock getLockForFile(String fileId) {
        return fileLocks.computeIfAbsent(fileId, k -> new ReentrantReadWriteLock());
    }

    /**
     * Upload or overwrite a file. This method is thread-safe.
     *
     * @param fileId unique identifier for the file
     * @param data   full file data
     */
    public void uploadFile(String fileId, byte[] data) {
        Objects.requireNonNull(fileId);
        Objects.requireNonNull(data);

        ReadWriteLock lock = getLockForFile(fileId);
        lock.writeLock().lock();
        try {
            List<String> chunkIds = new ArrayList<>();
            int offset = 0;
            while (offset < data.length) {
                int length = Math.min(CHUNK_SIZE, data.length - offset);
                byte[] chunkData = Arrays.copyOfRange(data, offset, offset + length);
                String chunkId = storeChunk(chunkData);
                chunkIds.add(chunkId);
                offset += length;
            }
            int newVersion = 1;
            FileMetadata oldMeta = metadataStorage.get(fileId);
            if (oldMeta != null) {
                newVersion = oldMeta.version + 1;
            }
            FileMetadata newMeta = new FileMetadata(fileId, newVersion, chunkIds, data.length);
            metadataStorage.put(fileId, newMeta);
        } finally {
            lock.writeLock().unlock();
        }
    }

    /**
     * Download a file. Returns null if file doesn't exist.
     *
     * @param fileId file identifier
     * @return file data or null if not found
     */
    public byte[] downloadFile(String fileId) {
        Objects.requireNonNull(fileId);

        ReadWriteLock lock = getLockForFile(fileId);
        lock.readLock().lock();
        try {
            FileMetadata meta = metadataStorage.get(fileId);
            if (meta == null) {
                return null;
            }
            byte[] fileData = new byte[(int) meta.size];
            int offset = 0;
            for (String chunkId : meta.chunkIds) {
                byte[] chunk = chunkStorage.get(chunkId);
                if (chunk == null) {
                    // Chunk missing - corrupt state
                    throw new IllegalStateException("Missing chunk data for id: " + chunkId);
                }
                System.arraycopy(chunk, 0, fileData, offset, chunk.length);
                offset += chunk.length;
            }
            return fileData;
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * Store chunk data and return its id. Deduplicate via chunk hash.
     */
    private String storeChunk(byte[] data) {
        String chunkId = Arrays.toString(java.security.MessageDigest.getInstance("SHA-256").digest(data));
        chunkStorage.putIfAbsent(chunkId, data);
        return chunkId;
    }
}
```

### Explanation:

- **Chunking:** Files are split into 1MB chunks.
- **Chunk storage:** Stored in a concurrent map keyed by chunk hash (SHA-256) to deduplicate identical chunks.
- **File metadata:** Each file has its version and an immutable list of chunk ids.
- **Locking:** Per-file `ReadWriteLock` protects concurrent metadata updates and reads:
  - Multiple downloads (readLock) can proceed concurrently.
  - Upload acquires writeLock to atomically update metadata.
- **Failure modes:** If chunk data is missing (corruption), download throws exception.
- **Edge cases:** Files smaller than chunk size stored as one chunk, empty data can be handled similarly.

Note: The cryptographic hash calculation throws `NoSuchAlgorithmException` which must be handled or rethrown in production code; omitted here for clarity.

## Key follow-up questions

1. **Q:** How does chunking help with concurrent uploads and downloads?  
   **A:** Chunking allows parallel processing of discrete file parts. Different clients or threads can upload/download chunks simultaneously, enabling partial updates and resuming interrupted uploads. It also limits memory usage per operation.

2. **Q:** What problems might arise if storing the entire file as one blob?  
   **A:** Large files may use excessive memory during upload/download, support for partial updates or streaming is limited, deduplication is not possible, and concurrency at finer granularity is infeasible.

3. **Q:** How does your design ensure atomic file updates?  
   **A:** We first upload all chunks to storage, then acquire a write lock and replace file metadata atomically. Readers see either the entire old or entire new version, never a partial state.

4. **Q:** What if multiple uploads happen concurrently for the same file?  
   **A:** The write lock ensures only one upload modifies metadata at a time, serializing concurrent writes to avoid corruption or conflicting versions.

5. **Q:** How can you extend this design to support file version history?  
   **A:** Instead of overwriting metadata, we can keep a versioned map of metadata objects. Each upload creates a new version entry. Reads can specify versions, enabling rollback or snapshot views.

6. **Q:** How would you persist this system beyond in-memory storage?  
   **A:** Metadata can be stored in a durable key-value store or RDBMS, chunk data in distributed blob/object storage (e.g., Amazon S3). Proper transactional guarantees or versioning would be needed for consistency.

## Takeaways

- Treating files as a single blob is too simplistic for real-scale services like Dropbox.
- Chunking files improves storage efficiency, concurrency, and supports advanced features like deduplication.
- Carefully designed concurrency control (per-file locks) allows safe concurrent reads and writes.
- Versioning and metadata management are critical for consistency and atomicity.
- Edge cases, partial failures, and storage durability are key considerations for production readiness.
- Thread-safe design requires balancing lock granularity and throughput to avoid bottlenecks.
