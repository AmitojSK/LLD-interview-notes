# 💈 His Sleeping Barber Lost 3 Customers Every Second — And He Couldn't Figure Out Why

> Automatically generated interview-preparation note.

## Original problem

My design looked clean — BarberShop, Barber, Customer, WaitingRoom. The barber loops checking a shared list for customers. The interviewer said 'Walk me through what happens when no customers are in the shop.' I said The barber checks the list in a while-loop. When it's empty, he sleeps for 100ms and checks again. Pretty standard polling.

## Interview-ready answer

## Problem understanding

The problem models the classic *Sleeping Barber* synchronization scenario:

- A barber shop has a single barber, a waiting room with limited chairs, and arriving customers.
- When customers arrive, they either sit in the waiting room if chairs are available or leave immediately if full.
- The barber serves customers one at a time.
- If no customers are present, the barber should wait (sleep) without consuming CPU.
- When a customer arrives and the barber is sleeping, the barber should be notified to wake up and serve them.
- The goal is to design a system to manage customer arrivals and the barber’s sleeping/waking with proper concurrency controls and minimal wasted resources (e.g., CPU cycles).

Constraints and design goals:

- Correct mutual exclusion and synchronization between barber and multiple customers arriving concurrently.
- Efficient waiting: avoid busy-wait polling loops with fixed sleep intervals.
- Proper handling of customers during empty waiting room.
- Liveness: no customers should be lost unnecessarily, and barber wakes promptly.
- Clean OO design encapsulating the barber shop behavior.

## Interview answer

The classic Sleeping Barber problem is a concurrency synchronization problem where the barber sleeps if no customers are present and wakes up when a customer arrives. Polling the customer list with sleeping intervals is a naive approach that wastes CPU and may cause latency.

A better design uses explicit thread synchronization primitives (`wait()`/`notify()` or higher-level constructs like Semaphores or Locks/Conditions):

### Core design

1. **Shared queue for waiting customers** with maximum capacity equal to waiting room chairs.
2. **Barber thread**:
   - Acquires a lock.
   - While the customer queue is empty:
     - Calls `wait()` to sleep (releases the lock and sleeps atomically).
   - Once notified and queue is not empty:
     - Dequeues a customer and cuts hair.
3. **Customer threads**:
   - Acquire the same lock.
   - If waiting room has a chair:
     - Enqueue themselves into the waiting queue.
     - Call `notify()` or `notifyAll()` to wake the barber if sleeping.
   - Else leaves (shop full).
   - Releases lock and waits for haircut.

### Trade-offs and reasoning

- **Polling with Thread.sleep(100ms)** wastes CPU and introduces latency up to the sleep interval.
- **wait()/notify()** is efficient and standard for thread coordination.
- Using a **BlockingQueue** can further simplify, where the barber thread `take()` blocks if empty.
- This design avoids race conditions and ensures prompt barber wake-up.
- Synchronization must guard all access to shared queue and state.

### Edge cases

- Multiple customers arriving concurrently must be safely enqueued or rejected.
- Barber must recheck condition after waking to avoid spurious wake-ups.
- Handle shop closing and termination conditions gracefully.

## Java implementation

```java
import java.util.LinkedList;
import java.util.Queue;

public class BarberShop {
    private final int capacity;
    private final Queue<Customer> waitingRoom = new LinkedList<>();
    private final Object lock = new Object();
    private boolean barberSleeping = false;

    public BarberShop(int capacity) {
        this.capacity = capacity;
    }

    // Barber thread: infinite loop serving customers
    public void barberWork() throws InterruptedException {
        while (true) {
            Customer customer;
            synchronized (lock) {
                while (waitingRoom.isEmpty()) {
                    barberSleeping = true;
                    System.out.println("Barber is sleeping...");
                    lock.wait(); // wait until a customer arrives
                    barberSleeping = false;
                }
                customer = waitingRoom.poll();
            }
            cutHair(customer);
        }
    }

    // Customer tries to enter and wait or leaves if full
    public boolean enterShop(Customer customer) {
        synchronized (lock) {
            if (waitingRoom.size() < capacity) {
                waitingRoom.add(customer);
                System.out.println(customer + " takes a chair.");

                // Wake barber if sleeping
                if (barberSleeping) {
                    lock.notify();
                }
                return true;
            } else {
                System.out.println(customer + " leaves: no free chairs.");
                return false;
            }
        }
    }

    private void cutHair(Customer customer) {
        System.out.println("Cutting hair of " + customer);
        try {
            Thread.sleep(500); // cut hair for 500ms
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("Finished haircut of " + customer);
    }

    static class Customer {
        private final String name;

        public Customer(String name) { this.name = name; }

        @Override
        public String toString() { return "Customer[" + name + "]"; }
    }

    public static void main(String[] args) {
        BarberShop shop = new BarberShop(3);
        Thread barberThread = new Thread(() -> {
            try {
                shop.barberWork();
            } catch (InterruptedException e) {
                System.out.println("Barber interrupted");
            }
        });

        barberThread.start();

        // Simulate customers arriving at random intervals
        for (int i = 1; i <= 10; i++) {
            final int id = i;
            new Thread(() -> {
                Customer c = new Customer("C" + id);
                boolean gotChair = shop.enterShop(c);
                if (!gotChair) {
                    System.out.println(c + " couldn't get a chair and left.");
                }
            }).start();

            try {
                Thread.sleep(200); // customers arriving every 200ms
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### Explanation

- Synchronization uses a private lock object.
- `waitingRoom` guarded by the lock.
- Barber calls `wait()` if no customers, releasing lock and sleeping efficiently.
- Customers `notify()` barber when entering.
- The `barberSleeping` boolean helps prevent unnecessary `notify()` calls but can be omitted with just `notify()` usage.
- Main simulates customers arriving in separate threads.
- Proper handling of interrupts and thread lifecycle is necessary for production code.

## Key follow-up questions

1. **Q:** Why is using `Thread.sleep()` to poll customer queue bad?
   **A:** It wastes CPU cycles wasting resources during sleep intervals and introduces latency because the barber only reacts after the sleep period, potentially delaying serving customers.

2. **Q:** How does `wait()`/`notify()` improve on this?
   **A:** They allow threads to block and release locks until notified, with no CPU wasted polling, leading to more efficient and responsive behavior.

3. **Q:** What happens if multiple customers arrive simultaneously?
   **A:** Synchronization on the shared lock ensures mutual exclusion. Only one thread manipulates the queue at a time, preventing race conditions.

4. **Q:** Can there be spurious wakeups in this design?
   **A:** Yes. That's why the barber uses a loop (`while (waitingRoom.isEmpty()) wait()`) to re-check condition after waking.

5. **Q:** How would you handle multiple barbers?
   **A:** Extend the design to support multiple barber threads, each waiting on customers; a thread-safe queue and proper signaling ensure correct handoff.

6. **Q:** What if the shop closes for the day?
   **A:** Add a `closed` flag and notify all waiting threads. Barber threads can exit when no customers and shop closed.

## Takeaways

- Avoid polling/sleep loops for wait conditions—use proper synchronization primitives.
- `wait()`/`notify()` requires a loop guard to handle spurious wakes safely.
- Always protect shared mutable state with locks.
- Design concurrent systems to be efficient, correct, and fair under concurrent access.
- The Sleeping Barber classic problem is an excellent example for interviewing concurrency design and thread coordination.
- Be prepared to discuss trade-offs, what happens on edge conditions, and how to extend your design.
