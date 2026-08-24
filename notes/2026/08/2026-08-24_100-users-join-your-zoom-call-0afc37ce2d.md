# 💀 100 Users Join Your Zoom Call

> Automatically generated interview-preparation note.

## Original problem

My design looked clean. User → Meeting → VideoConference → State. Then the interviewer asked...

## Interview-ready answer

## Problem understanding

The problem involves designing a system to support a video conferencing application where multiple users join a Zoom-like call. The initial design model is:

- **User** → **Meeting** → **VideoConference** → **State**

This hierarchy suggests a user participates in a meeting, which is associated with a video conference session, and the session maintains some state.

The interviewer’s question upon realizing 100 users joined the call likely points to scalability, real-time state management, resource allocation, and concurrency challenges when supporting many participants simultaneously.

## Interview answer

When designing a video conferencing system that needs to support hundreds of simultaneous users, key considerations include:

1. **Scalability and Performance**  
   - Maintaining the state of all users (e.g., video/audio streams, mute/unmute status) requires efficient data structures and distributed state management.  
   - The system should not rely on a single centralized state object because a bottleneck forms at high user counts.  

2. **Concurrency and Real-Time State Updates**  
   - Users join, leave, and change state concurrently, requiring thread-safe updates and event-driven architecture (e.g., event streams or message queues).  
   - Minimize locking and blocking to ensure responsiveness.

3. **Data Model Improvements**  
   - Flatten or decouple layers for better performance:  
     - Instead of **User → Meeting → VideoConference → State**, consider a **Meeting** that manages a **ParticipantManager** with lightweight **Participant** objects representing users.  
     - State should be partitioned and possibly sharded to handle many participants.

4. **Resource Management and Limits**   
   - Enforce maximum participants per meeting or dynamically allocate resources based on demand (like using microservices).  
   - Consider bandwidth, server CPU, and memory constraints.

5. **Media Stream Management**  
   - Video conferencing heavily depends on media server infrastructure (SFU/MCU).  
   - The design abstracts state from actual media transmission, but the backend must integrate with media servers that scale.

6. **Failure Modes and Fault Tolerance**  
   - Handle user disconnects, network partitions, and reconnections gracefully.  
   - Use heartbeat and state sync mechanisms.

## Java implementation

Below is a simplified, thread-safe example of how to manage participants in a meeting supporting large numbers of users. It focuses on the backend state model without media processing:

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.Set;

public class Meeting {
    private final String meetingId;
    // Thread-safe map of UserId to Participant state
    private final ConcurrentHashMap<String, Participant> participants = new ConcurrentHashMap<>();
    private final int maxParticipants;

    public Meeting(String meetingId, int maxParticipants) {
        this.meetingId = meetingId;
        this.maxParticipants = maxParticipants;
    }

    public boolean join(User user) {
        if (participants.size() >= maxParticipants) {
            return false; // Meeting full
        }
        Participant participant = new Participant(user);
        Participant existing = participants.putIfAbsent(user.getUserId(), participant);
        return existing == null;
    }

    public boolean leave(String userId) {
        Participant removed = participants.remove(userId);
        return removed != null;
    }

    public Participant getParticipant(String userId) {
        return participants.get(userId);
    }

    public Set<String> getAllParticipantIds() {
        return participants.keySet();
    }

    // Participant class maintains lightweight state per user
    public static class Participant {
        private final User user;
        private volatile boolean videoEnabled;
        private volatile boolean audioEnabled;
        private final AtomicInteger lastPingTimestamp = new AtomicInteger();

        public Participant(User user) {
            this.user = user;
            this.videoEnabled = true;
            this.audioEnabled = true;
        }

        public User getUser() { return user; }

        public boolean isVideoEnabled() { return videoEnabled; }
        public void setVideoEnabled(boolean enabled) { this.videoEnabled = enabled; }

        public boolean isAudioEnabled() { return audioEnabled; }
        public void setAudioEnabled(boolean enabled) { this.audioEnabled = enabled; }

        public void setLastPingTimestamp(int timestamp) {
            lastPingTimestamp.set(timestamp);
        }

        public int getLastPingTimestamp() {
            return lastPingTimestamp.get();
        }
    }
}

class User {
    private final String userId;
    private final String username;

    public User(String userId, String username) {
        this.userId = userId;
        this.username = username;
    }

    public String getUserId() { return userId; }
    public String getUsername() { return username; }
}
```

**Notes:**  
- We use `ConcurrentHashMap` to avoid blocking synchronizations.  
- `Participant` state is kept minimal; all mutation uses volatile or atomic types for safe concurrent access.  
- This model facilitates efficient add/remove operations even under high concurrency.  
- Real video/audio transmission is beyond this scope but would require integration with media servers.

## Key follow-up questions

- How would you handle synchronization across distributed systems if the meeting state is shared across multiple backend servers?  
- How might the design change when considering video frame distribution (e.g., Selective Forwarding Unit or SFU)?  
- How would you support muting/unmuting and propagating those state changes in near real-time?  
- What data models or caching strategies would you use to optimize latency for 100+ users?  
- How do you monitor and enforce meeting size limits or resource quotas?  
- How would you handle network failures and reconnection logic in participants?

## Takeaways

- Low-level data structures for managing participants in-memory should prioritize thread safety and concurrent access to scale to 100+ users.  
- The initial layered design should evolve into a model with better concurrency and distribution capabilities.  
- Real-time state management and failure resilience are critical in video conferencing.  
- Abstraction of user, meeting, and participant is essential but needs to balance simplicity with performance and extensibility.  
- Always consider integration points with media servers and infrastructure beyond mere backend state representation when scaling video calls.
