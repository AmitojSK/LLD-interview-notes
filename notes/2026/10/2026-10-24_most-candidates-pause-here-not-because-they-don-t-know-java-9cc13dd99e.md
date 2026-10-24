# Most candidates pause here. Not because they don't know Java

> Automatically generated interview-preparation note.

## Original problem

My design was clean — User, Profile, Connection, Post, Feed. The interviewer said 'Walk me through getFeed().

## Interview-ready answer

## Problem understanding
The problem focuses on designing a social media feed system where users have profiles, connections, and posts, and the key operation is to implement a `getFeed()` method. This method should return a user's feed, typically a list of posts visible to that user, aggregated from their connections' posts. Key constraints include handling potentially large numbers of connections and posts, maintaining freshness and relevance of the feed, and ensuring efficient retrieval time. Design goals are clarity, scalability, and maintainability.

## Interview answer
To design `getFeed()`, first understand the feed semantics — it generally includes posts from the user’s connections, ordered by recency or relevance.

**Key components:**

- **User:** Represents an individual with a Profile.
- **Profile:** User details.
- **Connection:** Represents friendships or follow relationships.
- **Post:** Content created by users.
- **Feed:** A list of posts aggregated for a particular user.

**Core Design for `getFeed()`:**

1. **Data retrieval:**
   - Get the list of connections for the user.
   - Fetch recent posts from those connections.
   
2. **Feed generation approaches:**

   - **Pull model:**
     - At request time, retrieve connections’ posts, merge and sort by timestamp or relevance.
     - Simple but expensive if user has many connections or posts.
   
   - **Push model (precomputed feed):**
     - When a user or their connections post, push updates or create feed copies asynchronously.
     - Fast retrieval, but complex to maintain consistency.
   
3. **Design choice depends on scale and system goals:**
   - For a small-scale system or prototype, pull model works fine.
   - For high-scale, latency-sensitive systems, push/precompute or hybrid models should be considered.
   
4. **Handling ordering & pagination:**
   - Sort by timestamp descending.
   - Support pagination with cursors or page numbers.
   
5. **Caching & performance:**
   - Cache frequently accessed feeds.
   - Use indices on posts for efficient retrieval.

6. **Concurrency & consistency:**
   - Ensure posts are visible after being published (read-after-write consistency).
   - Handle updates/deletions gracefully.

**Summary:**
`getFeed()` fetches connections, retrieves recent posts, merges and sorts them, and returns a window for the user.

## Java implementation

```java
import java.util.*;
import java.util.stream.Collectors;

class User {
    private final String id;
    private final Profile profile;
    private final Set<User> connections = new HashSet<>();
    private final List<Post> posts = new ArrayList<>();

    public User(String id, Profile profile) {
        this.id = id;
        this.profile = profile;
    }

    public String getId() { return id; }

    public void addConnection(User user) {
        connections.add(user);
    }

    public void addPost(Post post) {
        posts.add(post);
    }

    public Set<User> getConnections() {
        return Collections.unmodifiableSet(connections);
    }

    // Returns user's own posts
    public List<Post> getPosts() {
        return Collections.unmodifiableList(posts);
    }
}

class Profile {
    private final String name;
    public Profile(String name) { this.name = name; }
    public String getName() { return name; }
}

class Post implements Comparable<Post> {
    private final String content;
    private final long timestamp; // epoch millis for ordering
    private final User author;

    public Post(String content, long timestamp, User author) {
        this.content = content;
        this.timestamp = timestamp;
        this.author = author;
    }

    public long getTimestamp() { return timestamp; }
    public User getAuthor() { return author; }
    public String getContent() { return content; }

    @Override
    public int compareTo(Post other) {
        return Long.compare(other.timestamp, this.timestamp); // Desc order
    }
}

class FeedService {

    /**
     * Generates a feed for given user by merging latest posts from connections.
     * @param user the user whose feed is requested
     * @param maxFeedPosts max number of posts to return
     * @return list of posts sorted by timestamp descending
     */
    public List<Post> getFeed(User user, int maxFeedPosts) {
        // 1. Collect posts from connections
        List<Post> allPosts = new ArrayList<>();

        for (User connection : user.getConnections()) {
            allPosts.addAll(connection.getPosts());
        }
        // Optionally add user's own posts as well:
        allPosts.addAll(user.getPosts());

        // 2. Sort posts by timestamp descending
        Collections.sort(allPosts);

        // 3. Return top maxFeedPosts
        return allPosts.stream()
                .limit(maxFeedPosts)
                .collect(Collectors.toList());
    }
}
```

**Notes on implementation:**

- For simplicity, posts are stored in memory per user. In real systems, posts would be stored in databases or search indexes.
- Sorting is done on the combined list. For large data, a priority queue or merging sorted lists approach can be more efficient.
- No concurrency control is shown but should be considered if posts/connections are modified concurrently.
- Pagination can be added by passing an offset or cursor.
- No caching or async feed precomputation is implemented here.

## Key follow-up questions

1. **Q: How would you scale this feed system for millions of users?**  
   **A:** Use a push-based system to precompute feeds asynchronously when posts are created, store them in fast read stores (e.g., Redis). Use fan-out on write with queues. Employ caching and partition data by user ID.

2. **Q: How do you handle the order of posts beyond simple timestamps?**  
   **A:** Introduce relevance scoring, decay functions, likes or comments aggregation. Store a composite score for sorting. Use machine learning ranked models or heuristics.

3. **Q: What data structures can optimize merging posts from many connections?**  
   **A:** Use min-heaps or priority queues to merge pre-sorted post streams efficiently without sorting the entire combined list.

4. **Q: How to deal with deleted posts or connections?**  
   **A:** Implement soft deletes with tombstones to avoid inconsistency. Invalidate caches or recompute feed upon deletion events.

5. **Q: How to ensure feed freshness with eventual consistency?**  
   **A:** Use write-through caches, event-driven updates, and trade off slight latency for higher throughput. Inform users of “loading” or “new posts” indicators.

## Takeaways
- The `getFeed()` design centers on balancing simplicity and scalability.
- Pull model is simpler but less efficient at scale; push model is complex but performs better.
- Sorting and merging streams efficiently is a common challenge.
- Feed systems require careful handling of consistency, freshness, and personalization.
- Caching, pagination, and asynchronous processing are important enhancements in production-grade systems.
- Effective interview answers articulate trade-offs, design rationale, and are supported by clean code examples.
