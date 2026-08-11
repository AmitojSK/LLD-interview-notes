# 📰 Design a Thread-Safe News Feed

> Automatically generated interview-preparation note.

## Original problem

The Meta E5 question that destroyed a senior engineer who said "I'll fan out in a for-loop"

## Interview-ready answer

## Problem understanding

Design a **thread-safe News Feed system** where updates (e.g., new posts) need to be efficiently distributed ("fanned out") to subscribers or followers. The challenge is to handle concurrent updates and reads safely, ensuring correctness without significant performance degradation.

The phrase "I'll fan out in a for-loop" hints at a naive approach of synchronously updating every subscriber's feed inside a simple loop, which can cause performance bottlenecks, scalability issues, or concurrency problems.

## Interview answer

### Key problem aspects

1. **Concurrency:** Multiple users may post updates simultaneously.
2. **Performance and scalability:** Large-scale fan-out (e.g., millions of followers).
3. **Consistency guarantees:** Each user’s feed must be consistent and reflect all posts they should see.
4. **Thread safety:** Feed reads and writes must be safe under concurrent access.
5. **Efficiency:** Avoid blocking large sections of code; asynchronous or batch updates recommended.

### Design insights

- **Fan-out on write vs. fan-out on read:**  
  - Fan-out-on-write: When a user posts, push updates to all followers’ feeds immediately.  
  - Fan-out-on-read: Keep a global post store, fetch relevant posts on feed read dynamically.  
  
  Fan-out-on-write improves read speed but can cause performance bottlenecks when a user has many followers; naive synchronous loops don’t scale.

- **Thread-safe data structure choices:** Use concurrent collections (e.g., `ConcurrentLinkedQueue`, `ConcurrentHashMap`) to avoid locks where possible.

- **Batch Updates and Asynchronous Processing:** To prevent blocking threads for each follower, use a queue or message broker internally and background worker threads to update feeds asynchronously.

- **Sharding or partitioning:** Distribute followers' feed state across multiple partitions/threads to avoid contention.

- **Ordering Guarantees:** Use timestamps or incrementing sequence numbers to ensure feed ordering consistency.

### Proposed design outline

1. **Data structures:**  
   - Maintain a `ConcurrentHashMap<UserId, ConcurrentLinkedDeque<Post>>` to store each user's feed posts in order.  
   - Maintain a `ConcurrentHashMap<UserId, Set<UserId>>` for follower relationships.

2. **Posting a new update:**  
   - Retrieve the followers list atomically.  
   - Instead of blocking synchronization in a loop, submit feed update tasks to a thread pool executor or a queue for asynchronous processing.  
   - Each worker thread delivers the new post to each follower's feed concurrently but safely.

3. **Reading the feed:**  
   - Just read from the per-user concurrent deque, returning the latest N posts.

4. **Thread safety:**  
   - `ConcurrentHashMap` for follower tracking supports concurrent follower modifications.  
   - `ConcurrentLinkedDeque` allows concurrent reads/writes without external locking.

### Why naive for-loop is problematic

- Directly looping over all followers and updating their feed synchronously holds locks or blocks too many threads.
- It may cause latency spikes and reduce throughput.
- Does not scale for users with millions of followers.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;

public class NewsFeedService {

    // Map of userId to their feed posts (newest at front)
    private final ConcurrentHashMap<String, ConcurrentLinkedDeque<Post>> userFeeds = new ConcurrentHashMap<>();

    // Map of userId to their followers
    private final ConcurrentHashMap<String, Set<String>> followersMap = new ConcurrentHashMap<>();

    // Executor for asynchronous fan-out tasks
    private final ExecutorService executor = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors()
    );

    // Post class representing a news feed post
    public static class Post {
        public final String postId;
        public final String userId;
        public final String content;
        public final long timestamp;

        public Post(String postId, String userId, String content) {
            this.postId = postId;
            this.userId = userId;
            this.content = content;
            this.timestamp = System.currentTimeMillis();
        }
    }

    /**
     * Add a new follower relationship (follower follows followee)
     */
    public void addFollower(String followerId, String followeeId) {
        followersMap.computeIfAbsent(followeeId, k -> ConcurrentHashMap.newKeySet())
                    .add(followerId);
    }

    /**
     * Post a new update by the user and fan out to followers asynchronously
     */
    public void postUpdate(Post post) {
        // Add post to the author's own feed (optional, if author sees their posts)
        userFeeds.computeIfAbsent(post.userId, k -> new ConcurrentLinkedDeque<>()).addFirst(post);

        // Fan out to followers asynchronously
        Set<String> followers = followersMap.getOrDefault(post.userId, Collections.emptySet());

        for (String followerId : followers) {
            executor.submit(() -> {
                // Add post to each follower's feed
                userFeeds.computeIfAbsent(followerId, k -> new ConcurrentLinkedDeque<>()).addFirst(post);
            });
        }
    }

    /**
     * Read the latest N posts from a user's feed
     */
    public List<Post> getUserFeed(String userId, int limit) {
        ConcurrentLinkedDeque<Post> feed = userFeeds.get(userId);
        if (feed == null) return Collections.emptyList();

        List<Post> result = new ArrayList<>(limit);
        Iterator<Post> iterator = feed.iterator();

        while (iterator.hasNext() && result.size() < limit) {
            result.add(iterator.next());
        }
        return result;
    }

    /**
     * Shutdown executor to release resources; use in cleanup
     */
    public void shutdown() {
        executor.shutdown();
    }
}
```

## Key follow-up questions

- How to handle very large follower counts (e.g., celebrity users)?
  - Use hybrid fan-out: fan-out-on-read for huge follower sets.
  - Partition feeds across servers and workers.

- How to guarantee feed ordering and eventual consistency?
  - Use timestamps or sequence numbers.
  - Handle delays in asynchronous fan-out via idempotency and visibility windows.

- How to handle feed capacity and eviction?
  - Keep limited feed size per user with pruning logic.
  - Use bounded queues or TTL for old posts.

- How to handle follower addition and removal concurrently with fan-out?
  - Use atomic concurrent data structures and eventual consistency.

- What failure modes and retries should be considered?
  - Asynchronous tasks could fail; implement retry queues.
  - Persistent queues/message broker could increase reliability.

- How to design the system for distributed environments?
  - Use distributed caches, message queues, and sharded data stores.

## Takeaways

- Naively looping over followers synchronously is a simple but not scalable solution.
- Use concurrent data structures in Java (`ConcurrentHashMap`, `ConcurrentLinkedDeque`) for thread safety without blocking.
- Leverage asynchronous execution via `ExecutorService` to offload fan-out tasks and improve throughput.
- Understand trade-offs between fan-out on write vs. fan-out on read, and pick according to scale requirements.
- Consider failure modes, concurrency issues on follower changes, feed eviction, and ordering guarantees.
- Always reason about concurrency correctness, performance implications, and how the system will scale in real-world scenarios.
