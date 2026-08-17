# 💈 His Sleeping Barber Lost 3 Customers Every Second — And He Couldn't Figure Out Why

> Automatically generated interview-preparation note.

## Original problem

My design looked clean — BarberShop, Barber, Customer, WaitingRoom. The barber loops checking a shared list for customers. The interviewer said 'Walk me through what happens when no customers are in the shop.' I said The barber checks the list in a while-loop. When it's empty, he sleeps for 100ms and checks again. Pretty standard polling.

## Interview-ready answer

## Problem understanding
The problem is a classic "Sleeping Barber" concurrency scenario often used to test understanding of thread synchronization, wait/notify mechanisms, and resource-sharing in multithreaded environments.

- **Entities:** Barber, Customer, BarberShop, WaitingRoom.
- **Scenario:** The barber waits for customers. If none are present, he "sleeps" (waits).
- **Current approach:** The barber thread polls a shared list of waiting customers repeatedly, sleeping briefly when empty.
- **Issue:** This polling approach wastes CPU cycles and can cause performance inefficiencies or lost customers if proper synchronization isn't done.

The interviewer is testing the candidate's understanding of event-driven waiting and thread synchronization rather than naive polling.

## Interview answer
Polling a shared customer list with periodic sleeps is a suboptimal approach:

- It wastes CPU resources repeatedly waking up and checking for customers.
- It introduces latency: customers arriving right after the barber checks might wait unnecessarily until the barber wakes from sleeping.
- It risks race conditions if synchronization around the customer list and barber state is not handled properly.
- More importantly, it doesn't use Java's built-in thread signaling mechanisms to coordinate between threads.

A better design uses Java's **wait()** and **notify()/notifyAll()** methods on a shared monitor object.

- When no customers are present, the barber thread calls `wait()`, releasing the lock and sleeping indefinitely.
- When a customer arrives, the customer thread adds itself to the waiting list *while holding the lock* and then calls `notify()` to wake up the barber.
- This eliminates the polling loop and immediately wakes the barber only when needed.
  
This approach is more efficient, reduces latency, and avoids busy-waiting.

## Java implementation
Below is a simplified idiomatic implementation of Sleeping Barber using `wait()`/`notify()`:

```java
import java.util.LinkedList;
import java.util.Queue;

public class BarberShop {
    private final int maxSeats;
    private final Queue<Customer> waitingRoom = new LinkedList<>();

    public BarberShop(int maxSeats) {
        this.maxSeats = maxSeats;
    }

    public synchronized boolean enterShop(Customer customer) {
        if (waitingRoom.size() == maxSeats) {
            System.out.println(customer.name + " leaves: no free seats.");
            return false;
        }
        waitingRoom.offer(customer);
        System.out.println(customer.name + " sits in waiting room.");
        notify();  // Wake up barber if waiting
        return true;
    }

    public synchronized Customer nextCustomer() throws InterruptedException {
        while (waitingRoom.isEmpty()) {
            System.out.println("Barber is waiting for customers...");
            wait();
        }
        Customer customer = waitingRoom.poll();
        System.out.println("Barber calls " + customer.name);
        return customer;
    }
}

class Barber implements Runnable {
    private final BarberShop shop;

    public Barber(BarberShop shop) {
        this.shop = shop;
    }

    @Override
    public void run() {
        try {
            while (true) {
                Customer customer = shop.nextCustomer();
                cutHair(customer);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Barber stopped");
        }
    }

    private void cutHair(Customer customer) throws InterruptedException {
        System.out.println("Cutting hair of " + customer.name);
        Thread.sleep(500);  // Simulate time to cut hair
        System.out.println("Finished haircut for " + customer.name);
    }
}

class Customer {
    public final String name;

    public Customer(String name) {
        this.name = name;
    }
}
```

### Explanation
- `enterShop()` is synchronized and adds a customer or rejects if full, then `notify()`s the barber.
- `nextCustomer()` makes the barber wait if no customers are available.
- The barber thread blocks without polling, automatically wakes up when notified.
- This avoids CPU waste and race conditions.

## Key follow-up questions
- What happens if multiple customers arrive at once? How to handle concurrency?
- How to handle the case of multiple barbers?
- What potential deadlocks can occur with improper `wait()`/`notify()` usage?
- How to handle customer timeouts in waiting room?
- What happens if the barber is interrupted? How should the system react to shutdown gracefully?
- Why prefer `notify()` over `notifyAll()` or vice-versa in this scenario?
- How do you ensure fairness among customers?

## Takeaways
- Polling loops with fixed sleep intervals waste CPU and create latency.
- Use Java intrinsic locks with `wait()` and `notify()` to build event-driven waiting.
- Proper synchronization on the shared customer queue is critical to avoid race conditions.
- The sleeping barber problem is a classic example demonstrating effective inter-thread communication.
- Designing low-latency, efficient concurrency systems requires understanding Java thread coordination primitives.
- Always consider thread lifecycle management and graceful shutdown handling in production-quality systems.
