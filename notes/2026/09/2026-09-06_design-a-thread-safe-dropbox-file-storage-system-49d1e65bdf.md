# 📦 "Design a Thread-Safe Dropbox File Storage System"

> Automatically generated interview-preparation note.

## Original problem

The Dropbox L5 question that destroyed a senior engineer who said "I'll just store the whole file as one blob".

## Interview-ready answer

## Problem understanding

Design a thread-safe file storage system similar to Dropbox that supports concurrent accesses and modifications. The problem emphasizes handling file data not simply as one large blob (which is naive and inefficient), but rather with a data structure and system supporting:

- Concurrent read/write/update/append operations on files by multiple clients/threads.
- Efficient storage to handle large files without loading entire files into memory.
- Data consistency and thread safety.
- Scalability to support multiple files and users.
- Potential versioning or conflict resolution (optional but relevant).

Design goals:

- Fine-grained concurrency control over file data.
- Efficient access and updates (e.g., block/chunk-based storage).
- Thread-safe APIs for reading and writing.
- Avoid coarse-grained locking that blocks all access on the whole file.
- Support appends, reads, overwrites at arbitrary positions.

## Interview answer

**Core Design:**

1. **File Representation**  
   Represent each file as a sequence of smaller fixed-size blocks or chunks (e.g., 4KB each), stored independently.

2. **Data Structures**  
   Internally, use a concurrent map or list to hold file blocks by index. This allows partial file I/O without loading the entire file into memory.

3. **Concurrency Control**  
   - Use fine-grained locking: one lock per block or use lock-free concurrency (e.g., `ConcurrentHashMap` for blocks).  
   - Reads and writes can operate on different blocks simultaneously without blocking the whole file.  
   - For operations spanning multiple blocks (e.g., overwrite), lock the affected blocks in order to prevent deadlock.

4. **API Considerations**  
   - Provide thread-safe read(offset, length) and write(offset, data) methods.  
   - Append can be handled as write at file end (requires atomic file length update).  
   - File length management must be thread-safe.

5. **Memory & Storage**  
   - Persist blocks independently to disk or external storage.  
   - Cache hot blocks in memory.  
   - Support lazy loading of blocks.

6. **Error Handling & Consistency**  
   - Use transactional write or CAS-like operation on blocks for atomic updates.  
   - Handle partial updates and failures to avoid data corruption.

**Trade-offs:**

- Fixed-size blocks add complexity but enable concurrency and efficient memory use.  
- More locks increase overhead but improve parallelism. Balance is needed.  
- Using a global file lock simplifies correctness but kills concurrency.  
- Managing file metadata (length, version) atomically is important for consistency.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

public class ThreadSafeFileStorage {

    private static final int BLOCK_SIZE = 4 * 1024; // 4KB blocks

    /**
     * Represents a file stored in blocks.
     */
    public static class File {
        // Blocks indexed by block number
        private final ConcurrentHashMap<Integer, byte[]> blocks = new ConcurrentHashMap<>();
        // Lock per block to provide fine-grained concurrency
        private final ConcurrentHashMap<Integer, ReentrantReadWriteLock> blockLocks = new ConcurrentHashMap<>();
        // To atomically maintain file length
        private final AtomicLong fileLength = new AtomicLong(0);

        // Returns the lock object for a block, creating if absent
        private ReentrantReadWriteLock getLockForBlock(int blockIndex) {
            return blockLocks.computeIfAbsent(blockIndex, k -> new ReentrantReadWriteLock());
        }

        /**
         * Thread-safe read operation from file.
         * @param offset offset in the file to start reading
         * @param length length of data to read
         * @return byte array with data read
         */
        public byte[] read(long offset, int length) {
            if (offset < 0 || length < 0) throw new IllegalArgumentException("Invalid offset/length");
            if (offset >= fileLength.get()) return new byte[0]; // EOF or beyond

            long maxReadable = fileLength.get() - offset;
            int toRead = (int) Math.min(length, maxReadable);
            byte[] result = new byte[toRead];

            int startBlock = (int) (offset / BLOCK_SIZE);
            int endBlock = (int) ((offset + toRead - 1) / BLOCK_SIZE);

            int resultOffset = 0;
            for (int blockIndex = startBlock; blockIndex <= endBlock; blockIndex++) {
                ReentrantReadWriteLock lock = getLockForBlock(blockIndex);
                lock.readLock().lock();
                try {
                    byte[] block = blocks.get(blockIndex);
                    if (block == null) {
                        // Block not written yet, treat as zeroes
                        int blockStart = blockIndex * BLOCK_SIZE;
                        int blockOffset = (int) Math.max(offset - blockStart, 0);
                        int blockEnd = Math.min(BLOCK_SIZE, (int) ((offset + toRead) - blockStart));
                        int len = blockEnd - blockOffset;
                        Arrays.fill(result, resultOffset, resultOffset + len, (byte) 0);
                        resultOffset += len;
                    } else {
                        int blockStart = blockIndex * BLOCK_SIZE;
                        int blockOffset = (int) Math.max(offset - blockStart, 0);
                        int blockEnd = Math.min(BLOCK_SIZE, (int) ((offset + toRead) - blockStart));
                        int len = blockEnd - blockOffset;
                        System.arraycopy(block, blockOffset, result, resultOffset, len);
                        resultOffset += len;
                    }
                } finally {
                    lock.readLock().unlock();
                }
            }
            return result;
        }

        /**
         * Thread-safe write operation at given offset.
         * May extend the file length.
         * @param offset offset in file to write to
         * @param data data to write
         */
        public void write(long offset, byte[] data) {
            if (offset < 0) throw new IllegalArgumentException("Negative offset");
            if (data == null || data.length == 0) return;

            int startBlock = (int) (offset / BLOCK_SIZE);
            int endBlock = (int) ((offset + data.length - 1) / BLOCK_SIZE);

            // Acquire write locks on all affected blocks in order to avoid deadlocks
            List<ReentrantReadWriteLock> locksToAcquire = new ArrayList<>();
            for (int i = startBlock; i <= endBlock; i++) {
                locksToAcquire.add(getLockForBlock(i));
            }
            locksToAcquire.forEach(lock -> lock.writeLock().lock());

            try {
                int dataOffset = 0;
                for (int blockIndex = startBlock; blockIndex <= endBlock; blockIndex++) {
                    int blockStart = blockIndex * BLOCK_SIZE;
                    int blockEnd = blockStart + BLOCK_SIZE;

                    int writeStartInBlock = (int) Math.max(offset, blockStart) - blockStart;
                    int writeEndInBlock = (int) Math.min(offset + data.length, blockEnd) - blockStart;
                    int lengthToWrite = writeEndInBlock - writeStartInBlock;

                    byte[] block = blocks.get(blockIndex);
                    if (block == null) {
                        block = new byte[BLOCK_SIZE];
                        blocks.put(blockIndex, block);
                    }

                    System.arraycopy(data, dataOffset, block, writeStartInBlock, lengthToWrite);
                    dataOffset += lengthToWrite;
                }

                // Update file length atomically if write extends file
                long endPos = offset + data.length;
                fileLength.updateAndGet(len -> Math.max(len, endPos));
            } finally {
                locksToAcquire.forEach(lock -> lock.writeLock().unlock());
            }
        }

        /**
         * Thread-safe append operation.
         * @param data data to append
         */
        public void append(byte[] data) {
            while (true) {
                long length = fileLength.get();
                long newLength = length + data.length;
                // Try to reserve the space atomically
                if (fileLength.compareAndSet(length, newLength)) {
                    write(length, data);
                    break;
                }
                // else retry
            }
        }

        /**
         * Return the current length of the file.
         */
        public long length() {
            return fileLength.get();
        }
    }

    /**
     * Manage multiple files identified by unique names/ids.
     */
    public static class FileStorage {
        private final ConcurrentHashMap<String, File> files = new ConcurrentHashMap<>();

        public File createFile(String fileName) {
            return files.computeIfAbsent(fileName, k -> new File());
        }

        public File getFile(String fileName) {
            return files.get(fileName);
        }

        public void deleteFile(String fileName) {
            files.remove(fileName);
        }
    }
}
```

**Explanation:**

- We split file data into fixed-size blocks stored in `ConcurrentHashMap`.  
- Each block has an associated `ReentrantReadWriteLock` for concurrency control.  
- Reads acquire read locks on relevant blocks, supporting parallel reads.  
- Writes acquire write locks on relevant blocks to update safely. Locks acquired in ascending block order prevent deadlocks.  
- File length tracked by `AtomicLong` to atomically maintain consistency.  
- Append uses CAS loop to reserve a range and write data.

**Edge cases handled:**

- Reads beyond EOF return zero bytes without error.  
- Writes extending file length update it atomically.  
- Uninitialized blocks are treated as zeros during reads.  
- Concurrency at block granularity maximizes parallelism while ensuring data correctness.

## Key follow-up questions

1. **Q: Why do we split files into fixed-size blocks?**  
   **A:** Splitting into blocks allows fine-grained concurrency control, partial loading, and efficient memory/storage usage. It avoids locking the entire file for small operations.

2. **Q: How does the locking strategy prevent deadlocks when multiple blocks are locked?**  
   **A:** Locks are acquired in increasing block index order consistently. This ordered locking avoids circular wait conditions, preventing deadlocks.

3. **Q: What happens if multiple threads append data concurrently?**  
   **A:** Appends use an atomic CAS on file length to reserve a write offset, ensuring each append gets a unique contiguous space. Then, writes execute safely at reserved offsets.

4. **Q: How would you extend this design to support versioning or snapshots?**  
   **A:** We could store multiple versions of blocks, use copy-on-write, and maintain version metadata. Reads specify versions, and writes create new versions atomically.

5. **Q: What are potential performance bottlenecks in this design, and how to mitigate them?**  
   **A:** High contention on same blocks may cause lock contention. Mitigation includes block size tuning, read-write locks allowing concurrent reads, and caching. Also consider lock striping or lock-free structures.

## Takeaways

- Naively storing full files as blobs kills concurrency and efficiency; chunked storage enables parallel, scalable file operations.  
- Fine-grained locking based on blocks balances concurrency and correctness.  
- Atomic metadata management (e.g., atomic file length) is critical for consistency.  
- Ordered lock acquisition prevents deadlocks in multi-block operations.  
- Thread-safe append requires careful atomic position reservation.  
- Block abstraction simplifies complex file operations into manageable units and supports scalability.
