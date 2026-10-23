# The Elevator Skipped

> Automatically generated interview-preparation note.

## Original problem

The lift kept changing directions inefficiently under heavy traffic.

## Interview-ready answer

## Problem understanding

The problem involves designing an elevator (lift) control system that operates efficiently under heavy traffic. The key issue is that the elevator changes direction too frequently or inefficiently, leading to delays and poor user experience. Constraints include managing multiple concurrent requests and optimizing direction changes to minimize wait and travel times. Design goals are to improve the elevator's scheduling algorithm to handle high traffic volumes efficiently, reduce unnecessary direction switches, and ensure fairness and responsiveness.

## Interview answer

To address inefficiencies in elevator direction changes under heavy traffic:

- **Core idea:** Implement an elevator scheduling algorithm that reduces direction flipping by grouping stops in the current travel direction before reversing. 

- **Approach:**
  1. **Request Queues**: Maintain two separate data structures (e.g., sorted sets or priority queues) for requests above the current floor and requests below.
  2. **Direction State**: Keep track of the elevator's current direction (`UP` or `DOWN`).
  3. **Serve Requests in One Direction**: The elevator first serves all requests in the current direction (either all upwards or downwards), stopping at requested floors in sorted order.
  4. **Change Direction When Needed**: Only switch direction when no more requests remain in the current direction.
  5. **Batch Requests**: By batching requests in one direction, minimize unnecessary reversals and reduce overall travel.

- **Considerations:**
  - When floors are requested in both directions, the decision on which set to serve first can affect wait times.
  - Implementing look-ahead or adaptive scheduling (e.g., nearest car algorithm) can improve efficiency.
  - For multiple elevators, add coordination logic.

- **Trade-offs:**
  - Prioritizing single-direction travel optimizes throughput but may increase wait for some passengers.
  - Complex scheduling mechanisms can lead to higher computational overhead.
  - System must balance between fairness and efficiency.

## Java implementation

```java
import java.util.*;

public class ElevatorController {
    private int currentFloor;
    private Direction currentDirection;
    private final NavigableSet<Integer> upRequests;
    private final NavigableSet<Integer> downRequests;

    public enum Direction {
        UP, DOWN, IDLE
    }

    public ElevatorController(int startFloor) {
        this.currentFloor = startFloor;
        this.currentDirection = Direction.IDLE;
        this.upRequests = new TreeSet<>();
        this.downRequests = new TreeSet<>(Collections.reverseOrder());
    }

    // Add a floor request
    public synchronized void requestFloor(int floor) {
        if (floor == currentFloor) {
            // Immediate stop or ignore if already there
            serveFloor(floor);
            return;
        }
        if (floor > currentFloor) {
            upRequests.add(floor);
        } else {
            downRequests.add(floor);
        }
        if (currentDirection == Direction.IDLE) {
            // Decide direction when idle
            currentDirection = floor > currentFloor ? Direction.UP : Direction.DOWN;
        }
    }

    // Elevator main loop step: decide next floor to move to
    public synchronized Integer getNextFloor() {
        if (currentDirection == Direction.UP) {
            Integer next = upRequests.ceiling(currentFloor + 1);
            if (next != null) {
                return next;
            } else if (!downRequests.isEmpty()) {
                currentDirection = Direction.DOWN;
                return downRequests.first();
            }
        } else if (currentDirection == Direction.DOWN) {
            Integer next = downRequests.ceiling(currentFloor - 1);
            if (next != null) {
                return next;
            } else if (!upRequests.isEmpty()) {
                currentDirection = Direction.UP;
                return upRequests.first();
            }
        } else {
            // IDLE state: try to find closest request
            if (!upRequests.isEmpty()) {
                currentDirection = Direction.UP;
                return upRequests.first();
            }
            if (!downRequests.isEmpty()) {
                currentDirection = Direction.DOWN;
                return downRequests.first();
            }
            return null; // no requests
        }
        return null; // no more requests
    }

    // Move elevator to next floor and serve requests
    public synchronized void moveAndServe() {
        Integer nextFloor = getNextFloor();
        if (nextFloor == null) {
            currentDirection = Direction.IDLE;
            return;
        }
        currentFloor = nextFloor;
        serveFloor(nextFloor);
    }

    private void serveFloor(int floor) {
        upRequests.remove(floor);
        downRequests.remove(floor);
        // Simulate door open/close etc.
        System.out.println("Serving floor: " + floor);
    }

    public int getCurrentFloor() {
        return currentFloor;
    }

    public Direction getCurrentDirection() {
        return currentDirection;
    }
}
```

### Explanation

- `upRequests` holds floors above the current floor, sorted ascending.
- `downRequests` holds floors below, sorted descending to serve closest first.
- Requests are added to appropriate sets based on floor relative to the elevator.
- Elevator continues in the current direction serving all requests before switching.
- If no requests in current direction, direction switches if opposing requests exist.
- When idle, the closest request sets direction.
- `moveAndServe` simulates one step of movement and servicing.

### Concurrency & Edge Cases

- Synchronized methods ensure thread safety for concurrent requests.
- Immediate stops for requests on current floor.
- If requests come while elevator is moving in opposite direction, they queue but won't cause immediate reversal.
- Supports dynamic addition of requests.

## Key follow-up questions

1. **Question:** How would you handle multiple elevators in the system?
   **Answer:** Introduce a centralized scheduler that assigns requests to the most suitable elevator based on current positions, directions, and loads. Use algorithms like nearest car or partition floors among elevators. The system must synchronize state and balance load to minimize wait times.

2. **Question:** How do you handle prioritizing emergency or VIP requests?
   **Answer:** Add priority levels to requests. Emergency or VIP requests can preempt the current schedule, causing the elevator to interrupt and serve those floors immediately or reroute accordingly, possibly with a dedicated service state.

3. **Question:** What if the elevator can have multiple stops on a floor? How do you handle passengers boarding and alighting efficiently?
   **Answer:** Maintain separate request queues per direction and track passenger boarding/alighting requests explicitly. The elevator can wait at a floor until all boarding and alighting complete, controlled with timers or sensors.

4. **Question:** How do you address starvation, where requests in one direction never get served because of continuous requests in opposite direction?
   **Answer:** Implement fairness policies like limiting continuous travel in one direction or scheduling periodic direction changes to serve waiting passengers on other floors, preventing indefinite postponement.

5. **Question:** How would you adapt your design for an express elevator that only stops on key floors?
   **Answer:** Maintain a set of eligible stops for the express elevator and filter requests accordingly. Scheduling logic ensures the elevator skips non-key floors and optimizes serving express stops while considering passenger transfers.

## Takeaways

- Efficient elevator scheduling under heavy traffic requires batching requests by direction to reduce frequent direction changes.
- Maintaining two sorted data structures for up/down requests simplifies decision-making and improves throughput.
- Balancing between throughput and passenger wait times is crucial.
- Handling concurrency and dynamic requests requires synchronization and careful state management.
- Additional system features like multi-elevator coordination, priority handling, and starvation avoidance improve practical usability.
- Simple yet robust scheduling algorithms scale better and are easier to maintain in real systems.
