# 📦 "Design a Thread-Safe Dropbox File Storage System"

> Automatically generated interview-preparation note.

## Original problem

The Dropbox L5 question that destroyed a senior engineer who said "I'll just store the whole file as one blob".

## Interview-ready answer

## Problem understanding

Design a thread-safe backend system to store files for a Dropbox-like service. The system should allow multiple clients to upload, download, and possibly update large files concurrently, ensuring data consistency and integrity, while avoiding naive approaches like storing the whole file as a single blob without concurrency or partial update considerations.

Key challenges:
- Support concurrent access and updates to large files.
- Efficient storage and retrieval (possibly chunking large files).
- Thread safety: prevent data corruption or inconsistent reads.
- Handling partial failures, retries, and resumable uploads/downloads.
- Managing metadata and versioning for files.

## Interview answer

A naive approach of storing the entire file as a single blob in memory or storage and replacing the whole file on update is inefficient and unsafe in a concurrent environment. It leads to race conditions, long blocking times, and poor scalability.

**Better approach:**

1. **File chunking:**  
   Split files into fixed-size chunks (e.g., 4MB). Each chunk can be independently stored and updated, reducing memory footprint and enabling parallel uploads/downloads.

2. **Metadata management:**  
   Maintain file metadata (file ID, size, list of chunk references, version/timestamps) in a separate, thread-safe data structure or database.

3. **Thread safety:**  
   Synchronize updates at the chunk or file-level using concurrent data structures and locking:
   - Use `java.util.concurrent.locks.ReadWriteLock` to allow multiple readers but exclusive writers.
   - Lock at per-file granularity for metadata.
   - Lock per chunk for chunk updates (optional).

4. **Concurrent operations:**  
   - Support resumable uploads/downloads by managing chunks independently.
   - Handle partial chunk updates by replacing chunks atomically or via versioning.

5. **Storage abstraction:**  
   Abstract the chunk storage so it can be local disk, distributed object storage, or database blobs.

6. **Failure handling and consistency:**  
   Use atomic operations where supported and maintain transaction-like semantics for updating metadata and chunk pointers.

## Java implementation

Below is a simplified, thread-safe LLD Java sketch for storing files as chunked blobs with concurrent reads and writes.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

class Chunk {
    private final int chunkIndex;
    private final byte[] data;

    public Chunk(int chunkIndex, byte[] data) {
        this.chunkIndex = chunkIndex;
        this.data = data;
    }

    public int getChunkIndex() {
        return chunkIndex;
    }

    public byte[] getData() {
        return data;
    }
}

class FileMetadata {
    private final String fileId;
    private final int chunkCount;
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final List<Chunk> chunks;

    public FileMetadata(String fileId, int chunkCount) {
        this.fileId = fileId;
        this.chunkCount = chunkCount;
        this.chunks = new ArrayList<>(Collections.nCopies(chunkCount, null));
    }

    public String getFileId() {
        return fileId;
    }

    // Thread-safe retrieval of chunks
    public List<Chunk> getChunks() {
        rwLock.readLock().lock();
        try {
            return new ArrayList<>(chunks);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    // Update a specific chunk atomically
    public void updateChunk(int index, byte[] data) {
        if (index < 0 || index >= chunkCount) {
            throw new IndexOutOfBoundsException("Chunk index invalid");
        }
        rwLock.writeLock().lock();
        try {
            chunks.set(index, new Chunk(index, data));
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}

class FileStorageService {
    // Map fileId to metadata, thread-safe concurrent map for file registration
    private final ConcurrentMap<String, FileMetadata> files = new ConcurrentHashMap<>();

    // Simulated chunk size
    private static final int CHUNK_SIZE = 4 * 1024 * 1024; // 4MB

    // Create new file record with given size
    public void createFile(String fileId, long fileSize) {
        int chunkCount = (int) Math.ceil((double) fileSize / CHUNK_SIZE);
        FileMetadata meta = new FileMetadata(fileId, chunkCount);
        if (files.putIfAbsent(fileId, meta) != null) {
            throw new IllegalArgumentException("File already exists");
        }
    }

    // Upload chunk data for file
    public void uploadChunk(String fileId, int chunkIndex, byte[] chunkData) {
        FileMetadata meta = files.get(fileId);
        if (meta == null) {
            throw new IllegalArgumentException("File not found");
        }
        if (chunkData.length > CHUNK_SIZE) {
            throw new IllegalArgumentException("Chunk data too large");
        }
        meta.updateChunk(chunkIndex, chunkData);
    }

    // Download file data (simple concatenation of chunks)
    public byte[] downloadFile(String fileId) {
        FileMetadata meta = files.get(fileId);
        if (meta == null) {
            throw new IllegalArgumentException("File not found");
        }
        List<Chunk> chunks = meta.getChunks();
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        for (Chunk chunk : chunks) {
            if (chunk != null) {
                out.write(chunk.getData(), 0, chunk.getData().length);
            }
        }
        return out.toByteArray();
    }
}
```

**Notes:**
- `ReadWriteLock` ensures multiple concurrent readers but exclusive writes per file metadata.
- Each chunk update takes write lock to update safely.
- This is a simplified in-memory sketch. In production:
  - Persist metadata and chunks in durable storage.
  - Optimize locking granularity.
  - Handle partial chunk uploads, retries, and more metadata (version, timestamps).
  - Consider chunk hash or checksum validation.
  - Scale the system to multiple nodes with distributed locking and storage.

## Key follow-up questions

- How to handle partial file updates or appends?
- How to support resumable uploads and concurrent client syncs?
- How to ensure durability and handle storage failures?
- How to scale metadata and chunk storage (sharding, distributed DB)?
- How to handle versioning and conflict resolution for concurrent writes?
- Trade-offs between locking granularity and complexity.
- How to support garbage collection and cleanup of orphaned chunks?
- Can we implement deduplication or compression?

## Takeaways

- Naively storing entire files in one blob is not scalable or thread-safe.
- Chunking files helps support parallelism, partial updates, and resumable transfers.
- Use appropriate concurrency primitives (`ReadWriteLock`, `ConcurrentHashMap`) to protect metadata and chunks.
- Separate data and metadata and design atomic update sequences.
- Consider failure modes—partial writes, retries, and crash consistency.
- Think about storage persistence and scalability when moving beyond in-memory.
- A sound design should balance thread safety, performance, and maintainability.
