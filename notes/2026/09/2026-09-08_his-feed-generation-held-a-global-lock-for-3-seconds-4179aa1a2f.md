# 📰 His Feed Generation Held a Global Lock for 3 Seconds

> Automatically generated interview-preparation note.

## Original problem

My design was clean — User, Profile, Connection, Post, Feed. The interviewer said 'Walk me through getFeed().

## Interview-ready answer

## Problem understanding
The core problem is to design a scalable, efficient feed generation system in a social media context. The entities involved include User, Profile, Connection (friends/followers), Post, and Feed. The key operation to optimize is `getFeed()`, which returns a personalized feed of posts for a user, reflecting recent and relevant content from their connections.

Constraints and design goals:
- **Latency:** `getFeed()` must be low latency, ideally under milliseconds, enabling real-time or near-real-time feed display.
- **Consistency:** The feed should reflect recent updates from connections with acceptable consistency.
- **Scalability:** The system must scale to large user bases and many posts.
- **Concurrency:** Avoid global locks or bottlenecks that block multiple users' feed retrieval at once.
- **Throughput:** Must handle high request volumes efficiently.

The email mentions a design where feed generation held a **global lock for 3 seconds**, indicating a blocking operation that hurts scalability and responsiveness.

## Interview answer

### Core design

1. **Entities:**
   - **User**: Represents a user account.
   - **Profile**: Contains metadata about the user.
   - **Connection**: Represents follower/followee or friends relationships.
   - **Post**: Contains user-generated content with timestamps.
   - **Feed**: The ordered list of posts relevant to a user.

2. **Feed generation approaches:**
   - **Pull model:** On `getFeed()`, fetch posts from all connections in real time, merge and sort them by timestamp.
     - **Pros:** Always fresh.
     - **Cons:** Slow and expensive for users with many connections.
   - **Push model:** On post creation, push post IDs or content to followers' feeds.
     - **Pros:** Fast read time.
     - **Cons:** Expensive writes, large fan-out, storage overhead.
   - **Hybrid:** Use a cache or precomputed feeds refreshed asynchronously.

3. **Avoiding blocking global locks:**
   - Use fine-grained locks (e.g., per-user feed lock).
   - Use lock-free or concurrent data structures.
   - Use asynchronous feed generation with caching and eventual consistency.

4. **Feed generation step walk-through (`getFeed()`):**
   - For the user, obtain a list of connection IDs.
   - Fetch recent posts from those connections from a post store.
   - Merge posts efficiently, sorted by recency.
   - Apply filtering or ranking rules (e.g., only posts from last 7 days, relevance score).
   - Return the feed list.

5. **Possible optimization:**
   - Precompute feeds periodically or on write.
   - Use Redis sorted sets or similar data structure to store user feeds.
   - Use background workers to build feeds asynchronously.
   - Avoid locking entire feed generation or user store state.

### Why a global lock is bad
- It serializes all feed requests, killing throughput.
- Causes increased latency for all users.
- Doesn't scale; blocks even unrelated users.
- Alternatives include fine-grained locks, lock-free caches, and wait-free concurrent reads.

## Java implementation

Below is a simplified idiomatic Java design illustrating key components and avoiding global locks, using concurrent data structures and asynchronous feed generation.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

// Represents a social media Post
class Post {
    final String postId;
    final String userId; // author
    final String content;
    final long timestamp; // epoch millis

    public Post(String postId, String userId, String content, long timestamp) {
        this.postId = postId;
        this.userId = userId;
        this.content = content;
        this.timestamp = timestamp;
    }

    public long getTimestamp() {
        return timestamp;
    }
}

// Represents Users and their connections
class UserConnections {
    private final ConcurrentMap<String, Set<String>> userToConnections = new ConcurrentHashMap<>();

    public void addConnection(String userId, String connectedUserId) {
        userToConnections.computeIfAbsent(userId, k -> ConcurrentHashMap.newKeySet()).add(connectedUserId);
    }

    public Set<String> getConnections(String userId) {
        return userToConnections.getOrDefault(userId, Collections.emptySet());
    }
}

// Post store simulates persistent storage for posts
class PostStore {
    private final ConcurrentMap<String, List<Post>> postsByUser = new ConcurrentHashMap<>();

    public void addPost(Post post) {
        postsByUser.computeIfAbsent(post.userId, k -> new CopyOnWriteArrayList<>()).add(post);
    }

    public List<Post> getRecentPosts(String userId, int limit) {
        return postsByUser.getOrDefault(userId, Collections.emptyList())
                .stream()
                .sorted(Comparator.comparingLong(Post::getTimestamp).reversed())
                .limit(limit)
                .collect(Collectors.toList());
    }
}

// Feed service to generate and cache feed asynchronously
class FeedService {
    private final UserConnections userConnections;
    private final PostStore postStore;

    // Cache of user feeds to avoid repeated computation
    private final ConcurrentMap<String, List<Post>> feedCache = new ConcurrentHashMap<>();

    // Feed update executor
    private final ScheduledExecutorService feedUpdateExecutor = Executors.newScheduledThreadPool(4);

    public FeedService(UserConnections userConnections, PostStore postStore) {
        this.userConnections = userConnections;
        this.postStore = postStore;

        // Periodically refresh feeds (e.g., every 5 seconds)
        feedUpdateExecutor.scheduleAtFixedRate(this::refreshAllFeeds, 0, 5, TimeUnit.SECONDS);
    }

    // Public API to get feed for user - returns cached feed instantly
    public List<Post> getFeed(String userId) {
        return feedCache.getOrDefault(userId, Collections.emptyList());
    }

    // Background refresh - called periodically or triggered on post creation
    private void refreshAllFeeds() {
        for (String userId : userConnections.userToConnections.keySet()) {
            refreshFeed(userId);
        }
    }

    private void refreshFeed(String userId) {
        Set<String> connections = userConnections.getConnections(userId);
        List<Post> feedPosts = new ArrayList<>();

        // Fetch recent posts from each connection - limits to reduce work
        for (String connId : connections) {
            feedPosts.addAll(postStore.getRecentPosts(connId, 10));
        }

        // Sort all posts by timestamp descending and limit feed size
        List<Post> sortedFeed = feedPosts.stream()
                .sorted(Comparator.comparingLong(Post::getTimestamp).reversed())
                .limit(50)
                .collect(Collectors.toList());

        // Update cache atomically - no lock needed thanks to ConcurrentHashMap
        feedCache.put(userId, sortedFeed);
    }

    public void shutdown() {
        feedUpdateExecutor.shutdown();
    }
}
```

### Key points:
- No global locks used; concurrency handled with `ConcurrentHashMap` and lock-free collections.
- Feed generation happens asynchronously on a scheduled executor to avoid blocking `getFeed()`.
- `getFeed()` returns a cached feed instantly, avoiding expensive real-time merging.
- `refreshFeed()` merges posts from connections, sorts, and caches.
- Edge cases like no connections or empty posts handled by returning empty lists.
- The design can be extended with more sophisticated ranking, filtering, and fan-out on write.

## Key follow-up questions

1. **Question:** How would you handle scalability when a user has millions of connections (followers/friends)?
   **Answer:** For large fan-out users, push-based feed generation can be very expensive. We typically use a hybrid approach: cache feeds for most users but generate feeds on request for users with millions of connections (like celebrities) by fetching recent posts from key connections only or using a prioritized subset. Sharding and partitioning user data across multiple servers also helps horizontal scale.

2. **Question:** How do you ensure consistency between feed cache and new posts?
   **Answer:** The feed is eventually consistent — there is a small delay until background refresh runs. For near real-time, event-driven feed refresh or incremental updates via message queues (like Kafka) can push new posts to follower feed caches immediately.

3. **Question:** Can `getFeed()` cause race conditions when updating feeds?
   **Answer:** Since `getFeed()` only reads cached feeds from thread-safe maps, no race conditions occur on read. Updates are done atomically by replacing the entire list. ConcurrentHashMap provides atomic `put()` so no locks are needed.

4. **Question:** How would you handle deleted or edited posts in the feed?
   **Answer:** Invalidation strategies must be in place. On post deletion or edit, events trigger feed cache refresh for affected users. Could mark posts as deleted in the cache or remove them during next refresh.

5. **Question:** What data structures are suitable for efficiently merging sorted posts from connections?
   **Answer:** Use a k-way merge with min-heaps or priority queues to efficiently merge multiple sorted lists avoiding full sort on concatenated posts.

## Takeaways

- Avoid global locks in scalable systems; they cause bottlenecks and increase latency.
- Asynchronous feed generation and caching provide a good trade-off between freshness and performance.
- Use concurrent collections in Java (ConcurrentHashMap, CopyOnWriteArrayList) for thread safety without heavy locking.
- Real-time feed updates must balance compute cost with user experience — hybrid push/pull models help.
- Design must handle failures gracefully: caches falling behind, partial data, or network issues.
- Efficient merging of sorted post streams is a key algorithmic challenge in feed generation systems.
