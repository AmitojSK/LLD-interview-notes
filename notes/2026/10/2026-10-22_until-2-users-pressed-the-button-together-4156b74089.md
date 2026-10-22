# Until 2 users pressed the button together..

> Automatically generated interview-preparation note.

## Original problem

What happens when multiple users request the lift simultaneously from different floors?

## Interview-ready answer

## Problem understanding
The problem involves designing a lift (elevator) system capable of handling multiple simultaneous requests from different floors. The system should handle concurrent user requests efficiently, decide the sequence of floor stops, and avoid conflicts or inefficient routes. The design goals are to ensure responsiveness, fairness, and optimized elevator movement with minimal waiting and travel time.

Constraints:
- Multiple users pressing the lift call button simultaneously on different floors.
- The lift can be moving in one direction or idle.
- Requests from inside the lift to move to target floors.
- Must handle concurrency and prioritize stops efficiently.

## Interview answer
To handle multiple simultaneous requests from users on different floors, the lift system needs a scheduling mechanism that queues and prioritizes requests based on elevator direction and current position.

Key design points:
- Maintain two sorted sets/queues of requests: one for floors above the current position and one for floors below.
- Process requests in the current moving direction first to minimize back-and-forth travel.
- If idle, select the closest requested floor and start moving in that direction.
- Add new requests as they arrive: 
   - If the new request is along the current path, ensure it is handled in the current trip.
   - Otherwise, queue it for the next trip.
- Handle concurrency using thread-safe data structures or synchronization.
- Implement state management: Elevator states like MOVING_UP, MOVING_DOWN, IDLE.
- On reaching a floor, clear all requests for that floor.

Trade-offs:
- The simpler approach may lead to starvation of requests on the opposite side; can be addressed by alternating directions after completing current requests.
- Prioritizing the direction can optimize travel at the cost of potentially increasing wait time for some floors.
- More complex scheduling (like shortest seek time first) can be considered but at the cost of complexity.

## Java implementation

```java
import java.util.NavigableSet;
import java.util.TreeSet;
import java.util.concurrent.locks.ReentrantLock;

public class ElevatorSystem {
    private enum State { MOVING_UP, MOVING_DOWN, IDLE }

    private final ReentrantLock lock = new ReentrantLock();
    private State state = State.IDLE;
    private int currentFloor = 0;

    private final NavigableSet<Integer> upRequests = new TreeSet<>();
    private final NavigableSet<Integer> downRequests = new TreeSet<>((a, b) -> b - a);

    // Call elevator to a floor
    public void callElevator(int floor) {
        lock.lock();
        try {
            if (floor > currentFloor) {
                upRequests.add(floor);
            } else if (floor < currentFloor) {
                downRequests.add(floor);
            } else {
                // Already at floor: open door logic can be triggered here
            }
            if (state == State.IDLE) {
                determineNextState();
            }
        } finally {
            lock.unlock();
        }
    }

    // Request from inside the elevator to go to a particular floor
    public void requestFloor(int floor) {
        callElevator(floor); // Same logic applies
    }
    
    // Simulate the elevator moving continuously
    public void step() {
        lock.lock();
        try {
            if (state == State.IDLE) return;

            switch (state) {
                case MOVING_UP:
                    currentFloor++;
                    if (upRequests.contains(currentFloor)) {
                        upRequests.remove(currentFloor);
                        openDoor();
                    }
                    if (upRequests.isEmpty()) {
                        if (!downRequests.isEmpty()) {
                            state = State.MOVING_DOWN;
                        } else {
                            state = State.IDLE;
                        }
                    }
                    break;
                case MOVING_DOWN:
                    currentFloor--;
                    if (downRequests.contains(currentFloor)) {
                        downRequests.remove(currentFloor);
                        openDoor();
                    }
                    if (downRequests.isEmpty()) {
                        if (!upRequests.isEmpty()) {
                            state = State.MOVING_UP;
                        } else {
                            state = State.IDLE;
                        }
                    }
                    break;
                default:
                    break;
            }
        } finally {
            lock.unlock();
        }
    }

    private void openDoor() {
        // Door open/close logic here
        System.out.println("Opening door at floor " + currentFloor);
        // simulate delay if needed
    }

    private void determineNextState() {
        if (!upRequests.isEmpty()) {
            state = State.MOVING_UP;
        } else if (!downRequests.isEmpty()) {
            state = State.MOVING_DOWN;
        } else {
            state = State.IDLE;
        }
    }

    public int getCurrentFloor() {
        return currentFloor;
    }

    public String getState() {
        return state.toString();
    }
}
```

**Explanation:**
- `upRequests` and `downRequests` maintain floors requested above and below current position.
- The elevator moves up or down according to the active state.
- When the elevator reaches a requested floor, it "opens the door" and removes that floor from the queue.
- The system locks operations to ensure safe concurrency.
- If idle and new requests come, it picks the direction accordingly.

Edge cases:
- Requests at the current floor.
- No requests (IDLE state).
- Rapid requests causing changing directions frequently (could be optimized).

## Key follow-up questions
1. Question: How does your design handle simultaneous requests at the same time on multiple floors?
   Answer: Requests are added to thread-safe data structures (`TreeSet` inside a lock), so simultaneous calls are serialized safely and queued. The elevator processes them in the order dictated by the direction and nearest stops.

2. Question: What happens if a user inside the elevator requests a floor opposite the current direction?
   Answer: The system queues that request on the opposite direction's set. The elevator finishes current direction stops first and then reverses to service these pending requests.

3. Question: How do you avoid starvation of requests on one side?
   Answer: After completing requests in one direction, the elevator reverses direction if there are pending requests on the other side, ensuring all requests get served eventually.

4. Question: Can this design support multiple elevators?
   Answer: This current design is single elevator focused. For multiple elevators, we would need a dispatcher component that assigns requests to an elevator based on proximity, load, and direction to optimize overall system efficiency.

5. Question: How would you handle elevator faults or emergency stops?
   Answer: Additional states can be introduced for fault detection (e.g. OUT_OF_SERVICE), and the system can notify users or reroute requests. Fault recovery and safety mechanisms would also be added beyond basic scheduling.

## Takeaways
- Efficient elevator scheduling requires prioritizing requests in the current direction to minimize travel time.
- Using two sorted sets for upward and downward requests helps implement direction-based servicing.
- Thread safety is critical for concurrent user requests; locks or concurrent data structures help achieve this.
- Simple state management (MOVING_UP, MOVING_DOWN, IDLE) is effective for controlling elevator flow.
- Starvation can occur if not carefully handled; alternating directions and serving opposite queues helps fairness.
- While this design works for a single elevator, a multi-elevator system requires more complex dispatching logic.
