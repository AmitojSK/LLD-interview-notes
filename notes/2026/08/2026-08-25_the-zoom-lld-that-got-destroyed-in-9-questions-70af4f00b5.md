# 💀 The Zoom LLD That Got Destroyed in 9 Questions

> Automatically generated interview-preparation note.

## Original problem

Design Zoom — The Candidate Drew 3 Classes. The Interviewer Asked 9 Questions. It Was Over.

## Interview-ready answer

## Problem understanding
The problem is to design a simplified version of Zoom, a popular video conferencing application. This is a classic low-level design (LLD) problem for interviews where candidates must model core components like meetings, participants, and session management. The simple initial approach of drawing only a few classes typically falls short, as robust system design requires careful thought about interactions, scalability, features, and failure modes.

Key challenges include:
- Managing meeting sessions (creation, joining, leaving, ending).
- Supporting multiple participants concurrently.
- Handling video/audio streams or their abstractions.
- Supporting scheduling, invitations, and access control.
- Dealing with user presence, roles (host, co-host, attendee).
- Handling network and device disruptions.
- Ensuring scalability and real-time updates.

## Interview answer

### Core domain model and responsibilities:
1. **User**: Represents a participant with an identity, presence status, and role in the meeting.
2. **Meeting**: Represents a conferencing session with unique meeting ID, participants, host, and state (active/ended).
3. **MeetingManager**: Manages lifecycle operations – create meeting, join/leave participant, end meeting, and notify participants.
4. **VideoStream (optional abstraction)**: Abstracts video/audio streams per participant.
5. **Scheduler (optional)**: For scheduling future meetings.

### Design considerations:
- Users should be able to join or leave meetings asynchronously.
- Host controls (mute/unmute, participant management).
- Real-time updates require observer patterns or event-driven architecture.
- Handling multiple concurrent meetings and scalability concerns.
- Extensibility to support messaging, screen sharing, and recording.

### Common pitfalls:
- Drawing only 2-3 classes (e.g., just Meeting, User, and MeetingManager) often oversimplifies the problem.
- Not addressing concurrent user state changes and network failures.
- Ignoring real-time communication mechanisms.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

enum ParticipantRole {
    HOST,
    CO_HOST,
    ATTENDEE
}

enum MeetingState {
    CREATED,
    ACTIVE,
    ENDED
}

class User {
    private final String userId;
    private String name;

    public User(String userId, String name) {
        this.userId = userId;
        this.name = name;
    }

    public String getUserId() { return userId; }
    public String getName() { return name; }
}

class Participant {
    private final User user;
    private ParticipantRole role;
    private boolean isMuted;

    public Participant(User user, ParticipantRole role) {
        this.user = user;
        this.role = role;
        this.isMuted = false;
    }

    public User getUser() { return user; }
    public ParticipantRole getRole() { return role; }
    public boolean isMuted() { return isMuted; }
    public void mute() { isMuted = true; }
    public void unmute() { isMuted = false; }
    public void setRole(ParticipantRole role) { this.role = role; }
}

class Meeting {
    private final String meetingId;
    private final Map<String, Participant> participants = new ConcurrentHashMap<>();
    private MeetingState state;
    private String title;
    private final User host;

    public Meeting(String meetingId, User host, String title) {
        this.meetingId = meetingId;
        this.host = host;
        this.title = title;
        this.state = MeetingState.CREATED;
        addParticipant(host, ParticipantRole.HOST);
    }

    public String getMeetingId() { return meetingId; }
    public MeetingState getState() { return state; }
    public void start() { state = MeetingState.ACTIVE; }
    public void end() { state = MeetingState.ENDED; }

    public void addParticipant(User user, ParticipantRole role) {
        if (state == MeetingState.ENDED) {
            throw new IllegalStateException("Cannot join ended meeting");
        }
        participants.put(user.getUserId(), new Participant(user, role));
        // Notify others of new participant could be added here
    }

    public void removeParticipant(String userId) {
        participants.remove(userId);
        // Notify others participant left
    }

    public Participant getParticipant(String userId) {
        return participants.get(userId);
    }

    public Collection<Participant> getAllParticipants() {
        return participants.values();
    }

    // Additional methods to mute/unmute, transfer host, etc.
}

class MeetingManager {
    private final Map<String, Meeting> meetings = new ConcurrentHashMap<>();

    public Meeting createMeeting(User host, String title) {
        String meetingId = UUID.randomUUID().toString();
        Meeting meeting = new Meeting(meetingId, host, title);
        meetings.put(meetingId, meeting);
        return meeting;
    }

    public void startMeeting(String meetingId) {
        Meeting meeting = getMeeting(meetingId);
        meeting.start();
    }

    public void endMeeting(String meetingId) {
        Meeting meeting = getMeeting(meetingId);
        meeting.end();
        meetings.remove(meetingId);
    }

    public void joinMeeting(String meetingId, User user) {
        Meeting meeting = getMeeting(meetingId);
        meeting.addParticipant(user, ParticipantRole.ATTENDEE);
    }

    public void leaveMeeting(String meetingId, String userId) {
        Meeting meeting = getMeeting(meetingId);
        meeting.removeParticipant(userId);
    }

    private Meeting getMeeting(String meetingId) {
        Meeting meeting = meetings.get(meetingId);
        if (meeting == null) throw new IllegalArgumentException("Meeting does not exist");
        return meeting;
    }
}
```

## Key follow-up questions
- How do you handle concurrent joins/leaves and conflict resolution?
- How do you model and manage video/audio streams?
- How to support roles, permissions, and host controls?
- How to notify participants about state changes in real-time?
- How would you scale your service for millions of users and multiple meetings?
- What failure modes do you expect and how do you handle them (network failure, participant disconnected)?
- How to support scheduled meetings and recurring meetings?
- How to design the authentication and authorization for accessing meetings?
- How to integrate chat, screen share, and recording features into the design?

## Takeaways
- LLD problems require a clear domain model reflecting core entities, their attributes, and interactions.
- Start with a few essential classes but anticipate and design for concurrency, roles, and state transitions.
- Plan for extensibility: additional features like chat, screen sharing often require new classes or architectural components.
- Real-time communication requires event-driven design or observer patterns beyond simple CRUD operations.
- Consider failure handling, scalability, and security from the outset.
- Simple diagrams are useful but insufficient; be prepared to dive deeper into APIs, concurrent behavior, and system design details.
