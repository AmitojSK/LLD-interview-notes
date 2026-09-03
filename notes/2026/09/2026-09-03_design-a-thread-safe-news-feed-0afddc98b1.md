# 📰 Design a Thread-Safe News Feed

> Automatically generated interview-preparation note.

## Original problem

The Meta E5 question that destroyed a senior engineer who said "I'll fan out in a for-loop"

## Interview-ready answer

## Problem understanding

Design a thread-safe News Feed system that efficiently delivers updates from publishers to the followers' feeds. The system should support concurrent publishing and reading operations while ensuring correctness, consistency, and optimal performance. The naive approach of "fan out in a for-loop" (i.e., synchronously pushing each new post to every follower in a single thread) is inefficient, blocking, and does not scale well.

Key constraints and goals:
- Concurrent publishing by multiple users and concurrent reading by followers.
- Efficiently handle large numbers of followers per publisher.
- Ensure thread safety and avoid race conditions.
- Minimize latency for new feed generation.
- Balance between real-time updates and system scalability.
- Storage-efficient and scalable for millions of users.

## Interview answer

The core challenge is handling the "fan-out" problem — how to propagate a new post from a publisher to potentially millions of followers in a scalable and thread-safe manner.

### Core Design Approaches

1. **Pull Model (Lazy fan-out)**
   - Instead of pushing updates to every follower eagerly, the system stores posts in a publisher's timeline.
   - Followers fetch updates on demand by reading from the publishers they follow.
   - Pros: Scales well for users with many followers, low write amplification.
   - Cons: Reading a feed can be slower, as it must aggregate multiple timelines dynamically.

2. **Push Model (Eager fan-out)**
   - On publishing a post, the system immediately pushes it into each follower's feed.
   - Pros: Fast read access since each feed is pre-aggregated.
   - Cons: High write cost and resource contention, especially for users with many followers.

3. **Hybrid Approach**
   - Push fan-out for users with a small number of followers (to speed up reads).
   - Pull fan-out for users with very large followers to avoid write bottleneck.
   - Use a message queue or async job system to decouple fan-out work.

### Thread Safety and Concurrency

- Use concurrency-safe data structures (e.g., ConcurrentHashMap) for managing followers and feeds.
- Use immutable posts to avoid concurrent modification.
- Use asynchronous processing (e.g., ExecutorService or a message queue) to handle fan-out in parallel threads safely.
- Avoid the naive "for-loop synchronous push" which blocks and can cause thread contention.

### Key Components

- **UserManager:** Maintain user->followers mapping.
- **FeedStore:** Store feeds per user, must support concurrent appends and reads.
- **PostStore:** Store posts immutably for reuse.
- **FanoutService:** Responsible for publishing posts to followers' feeds asynchronously.
- **FeedService:** Provides the interface for reading feeds, fetching recent posts.

### Scalability & Optimization

- Partition feeds by user ID for horizontal scalability.
- Use caching for hot feeds.
- Use materialized views for some users (push) and on-the-fly aggregation for others (pull).
- Implement backpressure and retries in async fan-out.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;

// Immutable Post class
class Post {
    private final String postId;
    private final String publisherId;
    private final String content;
    private final long timestamp;

    public Post(String postId, String publisherId, String content) {
        this.postId = postId;
        this.publisherId = publisherId;
        this.content = content;
        this.timestamp = System.currentTimeMillis();
    }

    // Getters
    public String getPostId() { return postId; }
    public String getPublisherId() { return publisherId; }
    public String getContent() { return content; }
    public long getTimestamp() { return timestamp; }
}

// Manages user->followers mapping
class UserManager {
    // ConcurrentHashMap for thread-safe follower management
    private final ConcurrentMap<String, Set<String>> followersMap = new ConcurrentHashMap<>();

    // Follow operation
    public void follow(String followerId, String publisherId) {
        followersMap.computeIfAbsent(publisherId, k -> ConcurrentHashMap.newKeySet()).add(followerId);
    }

    // Unfollow operation
    public void unfollow(String followerId, String publisherId) {
        Set<String> followers = followersMap.get(publisherId);
        if (followers != null) {
            followers.remove(followerId);
        }
    }

    // Get followers for a publisher
    public Set<String> getFollowers(String publisherId) {
        return followersMap.getOrDefault(publisherId, Collections.emptySet());
    }
}

// Thread-safe, concurrent feed storage per user
class FeedStore {
    // Use ConcurrentMap with thread-safe deque for feed posts per user
    private final ConcurrentMap<String, ConcurrentLinkedDeque<Post>> feedMap = new ConcurrentHashMap<>();
    private final int MAX_FEED_SIZE = 1000;

    // Add post to user's feed
    public void addPostToFeed(String userId, Post post) {
        feedMap.computeIfAbsent(userId, k -> new ConcurrentLinkedDeque<>());
        ConcurrentLinkedDeque<Post> feed = feedMap.get(userId);

        synchronized(feed) {
            feed.addFirst(post);
            if (feed.size() > MAX_FEED_SIZE) {
                feed.removeLast();
            }
        }
    }

    // Retrieve recent posts (e.g., last N posts)
    public List<Post> getFeed(String userId, int limit) {
        ConcurrentLinkedDeque<Post> feed = feedMap.get(userId);
        if (feed == null) return Collections.emptyList();
        List<Post> result = new ArrayList<>(limit);
        Iterator<Post> iter = feed.iterator();
        int count = 0;
        while (iter.hasNext() && count < limit) {
            result.add(iter.next());
            count++;
        }
        return result;
    }
}

// Stores posts indexed by postId (usually backed by DB in prod)
class PostStore {
    private final ConcurrentMap<String, Post> posts = new ConcurrentHashMap<>();

    public void save(Post post) {
        posts.put(post.getPostId(), post);
    }

    public Post get(String postId) {
        return posts.get(postId);
    }
}

// Fanout service handles async fan-out of posts
class FanoutService {
    private final UserManager userManager;
    private final FeedStore feedStore;
    private final ExecutorService executor;

    public FanoutService(UserManager userManager, FeedStore feedStore, int workerThreads) {
        this.userManager = userManager;
        this.feedStore = feedStore;
        this.executor = Executors.newFixedThreadPool(workerThreads);
    }

    public void publishPost(Post post) {
        // Async fan-out: submit a task for each follower
        Set<String> followers = userManager.getFollowers(post.getPublisherId());

        // For large follower counts, consider batching or async queue instead of for-loop here
        for (String follower : followers) {
            executor.submit(() -> feedStore.addPostToFeed(follower, post));
        }

        // Add post to publisher's own feed
        feedStore.addPostToFeed(post.getPublisherId(), post);
    }

    public void shutdown() {
        executor.shutdown();
    }
}

// FeedService for reading feeds (pull model)
class FeedService {
    private final FeedStore feedStore;

    public FeedService(FeedStore feedStore) {
        this.feedStore = feedStore;
    }

    public List<Post> getUserFeed(String userId, int limit) {
        return feedStore.getFeed(userId, limit);
    }
}
```

### Explanation

- The `UserManager` maintains thread-safe sets of followers per publisher.
- `FeedStore` uses a `ConcurrentLinkedDeque` per user to store recent posts in a thread-safe manner.
- `FanoutService` asynchronously pushes the new post to all followers' feeds using a thread pool, avoiding blocking in the publisher thread.
- Adding the post to the publisher's own feed happens synchronously as an optimization.
- `FeedService` reads the pre-aggregated feed from feed store.
- Synchronization on feed deque ensures consistent update when trimming feed size.
- The design can be extended with batching, queues (e.g., Kafka), and partitioning for scalability.

## Key follow-up questions

1. **Q: What are the trade-offs between push and pull models here?**  
   A: Push model offers low-latency reads and a pre-built feed per user but at the cost of high write amplification and difficulty scaling for publishers with many followers. Pull model scales well on writes and is simpler to implement but may cause slower reads due to runtime aggregation.

2. **Q: How would you handle a publisher with millions of followers?**  
   A: Use asynchronous batching or queue-based systems to fan out posts gradually. For very large follower counts, switch to a mostly pull-based approach or use a hybrid model. Also, partition the fan-out workload over multiple machines.

3. **Q: How do you ensure thread safety for feed updates?**  
   A: Use concurrent data structures (like ConcurrentLinkedDeque) and synchronization where necessary for compound operations. Immutable post objects avoid data races on post data.

4. **Q: How would you handle feed storage to avoid memory blowup?**  
   A: Limit the feed size per user by trimming older posts. Use persistent storage (databases or caches like Redis) with eviction policies. Store only references or post IDs instead of full post content.

5. **Q: How can you improve latency for feed reads?**  
   A: Use caching layers, pre-aggregate feeds for active users, maintain time-ordered feed indexes, and optimize database queries.

## Takeaways

- Naive synchronous fan-out in a for-loop blocks the publisher thread and doesn’t scale.
- Decoupling fan-out logic via async tasks or message queues is crucial for scalability and responsiveness.
- Thread-safe data structures and immutable domain objects prevent concurrency bugs.
- Hybrid push/pull feed models help balance read/write load depending on user follower counts.
- Proper resource management, batching, caching, and partitioning are essential in large-scale feed designs.
- Understanding trade-offs between latency, scalability, and consistency is key in designing large distributed systems like News Feeds.
