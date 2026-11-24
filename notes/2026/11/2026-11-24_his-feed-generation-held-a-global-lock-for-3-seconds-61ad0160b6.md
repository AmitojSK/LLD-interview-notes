# 📰 His Feed Generation Held a Global Lock for 3 Seconds

> Automatically generated interview-preparation note.

## Original problem

My design was clean — User, Profile, Connection, Post, Feed. The interviewer said 'Walk me through getFeed().

## Interview-ready answer

## Problem understanding

We need to design a scalable `getFeed()` function for a social media platform. The system entities include User, Profile, Connections (followers/followees), Posts, and Feed. The primary goal is to generate a user’s feed efficiently, avoiding long delays or global locks. Key challenges include handling large-scale data, supporting real-time or near-real-time updates, and ensuring the feed generation is performant and concurrent without blocking other operations.

Constraints and design goals:

- The feed should show posts from a user’s connections, ideally sorted by recency or relevance.
- The system must be scalable to millions of users and posts.
- Avoid global locking or any operation that blocks the system excessively.
- Support fast read (feed retrieval) performance since it’s a high-demand feature.
- Allow for eventual consistency or near real-time updates, balancing complexity and freshness.
- The system should gracefully handle failures or partial data.

## Interview answer

### High-level design

1. **Entities**:
   - `User`: represents the user.
   - `Profile`: contains user metadata.
   - `Connection`: represents follower/followee relationships.
   - `Post`: user-generated content with timestamps.
   - `Feed`: a personalized timeline of posts for a user.

2. **Feed generation approaches**:
   - **Pull model (on-demand aggregation)**:
     Retrieve the latest posts from each connection’s timeline at request time, merge and sort them to create the feed.
     *Pros:* Always fresh feeds; no stale data.
     *Cons:* High latency, expensive queries, not scalable with many connections.
   - **Push model (precomputed feeds)**:
     When a user posts, push that post to the feeds of all followers.
     *Pros:* Fast feed reads, simple calls to `getFeed()`.
     *Cons:* Write amplification, complexity in handling large follower counts.
   - **Hybrid model**:
     Precompute feeds for users with small connections (push), pull on-demand for those with many connections.

3. **Avoiding global locks**:
   - Locks should be coarse and limited to per-user or per-post granularity.
   - Use asynchronous processing for feed updates (e.g., via message queues).
   - Read operations should be lock-free or use optimistic concurrency.
   - Avoid global synchronization — it causes delays (like the 3-second global lock).

4. **Concurrency and performance optimizations**:
   - Cache hot user feeds in distributed cache (Redis/Memcached).
   - Use asynchronous worker jobs to update feeds.
   - Use pagination to avoid loading entire feeds at once.
   - Implement rate limiting or batching for feed updates.

5. **Failure handling**:
   - Allow partial feeds; if some posts are missing due to failures, show the rest.
   - Use retries and dead-letter queues for failed updates.
   - Design for eventual consistency rather than strict ACID.

### Walkthrough of `getFeed()`:

- Query the cache first.
- If cache miss:
  - Depending on the model, either pull latest posts from connection timelines or read the precomputed feed from storage.
- Merge and sort posts by timestamp.
- Return a paginated response.
- All operations to be done without holding global locks.

---

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

class Post {
    final String postId;
    final String userId;
    final String content;
    final long timestamp;

    public Post(String postId, String userId, String content, long timestamp) {
        this.postId = postId;
        this.userId = userId;
        this.content = content;
        this.timestamp = timestamp;
    }
}

class User {
    final String userId;

    public User(String userId) {
        this.userId = userId;
    }

    // Returns a set of userIds this user follows (connections)
    Set<String> getConnections() {
        // For demo, return fixed or query from database
        return FeedService.getUserConnections(userId);
    }
}

class FeedService {
    // Simulating caches and store
    private static Map<String, List<Post>> userPosts = new ConcurrentHashMap<>();
    private static Map<String, List<Post>> precomputedFeeds = new ConcurrentHashMap<>();
    private static Map<String, Set<String>> connections = new ConcurrentHashMap<>(); // userId -> set of userIds followed

    /**
     * Get user connections (follows)
     */
    static Set<String> getUserConnections(String userId) {
        return connections.getOrDefault(userId, Collections.emptySet());
    }

    /**
     * For illustration: precompute feed asynchronously when user posts
     */
    static void addPost(Post post) {
        // Store post for author
        userPosts.computeIfAbsent(post.userId, k -> new CopyOnWriteArrayList<>()).add(post);

        // Push post to followers' precomputed feeds asynchronously
        Set<String> followers = getFollowers(post.userId);
        for (String follower : followers) {
            // Append post to follower's feed
            precomputedFeeds.computeIfAbsent(follower, k -> new CopyOnWriteArrayList<>()).add(post);
        }
    }

    // Inverse of connections map: who follows a user?
    static Set<String> getFollowers(String userId) {
        Set<String> followers = new HashSet<>();
        for (Map.Entry<String, Set<String>> entry : connections.entrySet()) {
            if (entry.getValue().contains(userId)) {
                followers.add(entry.getKey());
            }
        }
        return followers;
    }

    /**
     * The core method: get feed for requested user.
     * Uses precomputed feed if available, otherwise falls back to pull model.
     */
    public static List<Post> getFeed(String userId, int limit) {
        List<Post> feed = precomputedFeeds.get(userId);
        if (feed != null && !feed.isEmpty()) {
            return feed.stream()
                    .sorted(Comparator.comparingLong((Post p) -> p.timestamp).reversed())
                    .limit(limit)
                    .collect(Collectors.toList());
        }

        // Fallback: pull model: get recent posts from connections
        Set<String> connections = getUserConnections(userId);
        PriorityQueue<Post> maxHeap = new PriorityQueue<>((a, b) -> Long.compare(b.timestamp, a.timestamp));
        for (String friendId : connections) {
            List<Post> friendPosts = userPosts.getOrDefault(friendId, Collections.emptyList());
            maxHeap.addAll(friendPosts);
        }
        List<Post> result = new ArrayList<>();
        while (!maxHeap.isEmpty() && result.size() < limit) {
            result.add(maxHeap.poll());
        }
        return result;
    }
}
```

### Explanation

- The `FeedService` uses two models: 
  - `precomputedFeeds` map stores pushed posts from users' connections. This simulates the push model.
  - If no precomputed feed exists, it falls back to the pull model by aggregating posts from connections on demand.
- `addPost` represents a user posting new content; it pushes posts to followers’ feed asynchronously (in real systems via queues).
- We use thread-safe data structures `ConcurrentHashMap` and `CopyOnWriteArrayList` for concurrent reads/writes.
- Sorting and pagination are done in-memory here for clarity.
- No global lock is held; all operations on per-user data structures.
- The design can be extended to incorporate caching layers, queues, databases, and more.

---

## Key follow-up questions

1. **Question:** How do you handle a user with millions of followers when pushing a post to their feeds?

   **Answer:** For users with very large follower lists ("celebrities"), pushing posts to all followers synchronously is expensive. We can:
   - Use a hybrid approach, where feeds of such users' followers are generated lazily (pull model).
   - Push to a limited set of followers and provide a separate "celebrity feed."
   - Use fanout-on-read strategy or optimize via sharding.
   - Employ a message queue and batch/bulk operations to avoid blocking.

2. **Question:** What happens if feed update processing fails?

   **Answer:** The system should ensure eventual consistency:
   - Use retries and dead-letter queues for failed feed updates.
   - Allow users to read partial feeds without blocking on update completeness.
   - Monitor failures and alert for systemic issues.
   - In case of permanent failure, fall back on pull model as a backup.

3. **Question:** How do you handle concurrent posts or concurrent read/write to user feeds?

   **Answer:** Use thread-safe data structures and optimistic concurrency control:
   - No global locks; per-user data structures like concurrent maps and copy-on-write lists.
   - Atomic operations or synchronization at fine granularity (e.g., on per-user feeds).
   - Feed generation is usually read-heavy and can be handled with eventual consistency.
   - Writes (adds to feeds) can be asynchronous to minimize locking.

4. **Question:** How to support pagination in the feed?

   **Answer:** Pagination can be implemented by:
   - Passing a cursor or timestamp to `getFeed()` to retrieve posts after a certain point.
   - Using fixed-size pages with offset and limit.
   - Storing feed entries in sorted order allowing efficient range queries.
   - Cache pagination state or use database features supporting pagination.

5. **Question:** Can feed personalization beyond simple recency sorting be supported?

   **Answer:** Yes, extend feed ranking with:
   - User engagement signals (likes, comments).
   - Machine learning-based ranking models.
   - Content filtering based on user preferences.
   - Decouple feed generation and ranking from simple post aggregation; introduce scoring phases.

---

## Takeaways

- Designing `getFeed()` requires balancing latency, freshness, scalability, and complexity.
- Push-based precomputed feeds enable fast reads but increase write complexity.
- Pull-based real-time aggregation offers fresh data but higher latency.
- Avoid global locks completely; prefer asynchronous or fine-grained concurrency.
- Use concurrent data structures and caching layers for performance.
- Hybrid approaches and fallback mechanisms improve resiliency and scalability.
- Always consider failure modes and eventual consistency in large distributed systems.
- Pagination, personalization, and scalability are common extensions for real-world feed systems.
