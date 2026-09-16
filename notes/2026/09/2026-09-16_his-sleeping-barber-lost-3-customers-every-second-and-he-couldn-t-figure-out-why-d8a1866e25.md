# 💈 His Sleeping Barber Lost 3 Customers Every Second — And He Couldn't Figure Out Why

> Automatically generated interview-preparation note.

## Original problem

My design looked clean — BarberShop, Barber, Customer, WaitingRoom. The barber loops checking a shared list for customers. The interviewer said 'Walk me through what happens when no customers are in the shop.' I said The barber checks the list in a while-loop. When it's empty, he sleeps for 100ms and checks again. Pretty standard polling.

## Interview-ready answer

## Problem understanding
The problem is a classic concurrency and synchronization scenario known as the "Sleeping Barber" problem. It involves coordinating access between multiple customers and a single barber in a barber shop with a limited waiting room capacity. Key challenges include:

- Ensuring the barber doesn't waste CPU cycles polling for customers constantly.
- Allowing customers to wait if seats are available or leave if the waiting room is full.
- Waking the barber promptly when a customer arrives.
- Managing concurrency and shared resources (e.g., customer queue) safely.
- Avoiding race conditions, deadlocks, and wasted CPU (busy waiting or polling).

Design goals:
- Efficient synchronization between the barber and customers.
- Clear, maintainable design encapsulating roles: BarberShop, Barber, Customer, WaitingRoom.
- Minimal latency between customer arrival and barber servicing.
- Correct handling of empty waiting room scenario (barber sleeps without polling).

## Interview answer
This problem is a classic synchronization scenario best solved using proper thread signaling constructs rather than naïve polling. The key issue with the provided naive design is busy waiting: the barber constantly checks for customers and sleeps for fixed time intervals, wasting CPU and introducing delay.

### Core Design
- **WaitingRoom**: A bounded queue with a fixed capacity (number of chairs). Customers either wait or leave if full.
- **Barber**: Sleeps (wait states) when no customers; gets notified (woken up) when customers arrive.
- **Customer**: Tries to enter the waiting room. If full, they leave; else they join the queue.
- **BarberShop**: Coordinates barber and customers, managing the shared waiting room and synchronization.

### Synchronization Strategy
- Use **wait/notify** or **higher-level Java concurrency utilities**.
- The barber thread waits on a monitor/condition variable when no customers are available.
- Customers notify the barber when they arrive.
- This avoids polling by suspending the barber thread until there's work.
- Access to the waiting queue must be thread-safe.

### Trade-offs:
- Using raw `wait/notify` can be error-prone but lightweight.
- Using `java.util.concurrent` classes (like `BlockingQueue`) simplifies design, making the barber block waiting for customers.
- Polling wastes CPU and introduces latency.
- Ensuring proper dismissal of customers when full is important.

### Walkthrough
- When the shop opens, the barber thread calls `waitingRoom.takeCustomer()` which blocks if empty.
- When a customer arrives, they call `waitingRoom.enter(customer)` which returns false if full.
- If the customer enters, the barber is unblocked (either implicitly in a blocking queue or explicitly by notify).
- The barber serves the customer (simulate haircut).
- Loop continues.

This approach removes the sleeping and polling loop in the barber entirely.

## Java implementation

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

class BarberShop {
    private final BlockingQueue<Customer> waitingRoom;

    public BarberShop(int waitingChairs) {
        waitingRoom = new ArrayBlockingQueue<>(waitingChairs);
    }

    public boolean enter(Customer customer) {
        // Returns false immediately if waitingRoom is full
        return waitingRoom.offer(customer);
    }

    public Customer nextCustomer() throws InterruptedException {
        // Blocks if no customers, barber sleeps here without polling
        return waitingRoom.take();
    }
}

class Barber implements Runnable {
    private final BarberShop shop;
    private volatile boolean working = true;

    public Barber(BarberShop shop) { this.shop = shop; }

    public void stopWorking() {
        working = false;
        Thread.currentThread().interrupt(); // Optional, if blocked
    }

    @Override
    public void run() {
        try {
            while (working) {
                Customer customer = shop.nextCustomer(); // Blocks if none
                System.out.println("Starting haircut for " + customer.getName());
                doHaircut(customer);
                System.out.println("Finished haircut for " + customer.getName());
            }
        } catch (InterruptedException e) {
            // Thread interrupted; clean shutdown or continue
            Thread.currentThread().interrupt();
        }
    }

    private void doHaircut(Customer customer) throws InterruptedException {
        // Simulate haircut duration
        TimeUnit.MILLISECONDS.sleep(500);
    }
}

class Customer {
    private final String name;
    public Customer(String name) { this.name = name; }
    public String getName() { return name; }
}

// Example usage
public class SleepingBarberDemo {
    public static void main(String[] args) throws InterruptedException {
        BarberShop shop = new BarberShop(3); // 3 waiting chairs
        Barber barber = new Barber(shop);
        Thread barberThread = new Thread(barber);
        barberThread.start();

        // Simulate customers arriving
        for (int i = 1; i <= 10; i++) {
            Customer c = new Customer("Customer#" + i);
            if (shop.enter(c)) {
                System.out.println(c.getName() + " is waiting.");
            } else {
                System.out.println(c.getName() + " left (no seats).");
            }
            Thread.sleep(100); // New customer every 100ms
        }

        barber.stopWorking();
        barberThread.join();
    }
}
```

### Explanation
- `ArrayBlockingQueue` provides a thread-safe bounded queue.
- `barber.nextCustomer()` blocks indefinitely if no customers, so the barber thread "sleeps" without polling.
- `shop.enter()` uses `offer` to avoid blocking customers when full, returning false if no seats.
- Clean shutdown by interrupting the barber thread.
- This approach cleanly handles busy boundaries and avoids CPU waste.

## Key follow-up questions
1. **Q:** How do you avoid race conditions in your solution?  
   **A:** The `ArrayBlockingQueue` manages thread-safe access internally, so we don’t need explicit locks. Using well-tested concurrent data structures avoids many concurrency pitfalls.

2. **Q:** What happens when the barber finishes all customers?  
   **A:** The barber’s call to `take()` blocks until a new customer arrives; this means the barber thread sleeps without polling, efficiently waiting for work.

3. **Q:** Why is polling bad here, and what problems does it cause?  
   **A:** Polling wastes CPU cycles and introduces latency. The barber might miss customers arriving immediately after its check, causing delays or lost customers when the waiting room is small and busy.

4. **Q:** How would you handle multiple barbers?  
   **A:** Use a shared waiting queue where multiple barber threads block on `take()`. Each barber processes customers concurrently, and the queue manages fair access.

5. **Q:** Can customers be served out of order in your design?  
   **A:** No, `ArrayBlockingQueue` maintains FIFO order, so customers are served in the order of arrival.

## Takeaways
- Avoid simple polling or sleep loops for thread synchronization; prefer blocking synchronization.
- Use standard concurrency utilities (`BlockingQueue`, `Semaphore`, etc.) to simplify thread coordination.
- Bounded queues naturally model limited waiting room capacity.
- Proper thread signaling (`wait/notify` or blocking methods) ensures efficient resource usage and responsiveness.
- This problem is a good exercise in understanding classic synchronization patterns and designing for concurrency correctness and efficiency.
