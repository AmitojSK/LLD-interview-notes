# 🍽️ His Dining Philosophers Deadlocked After 3 Seconds — Every Single Run

> Automatically generated interview-preparation note.

## Original problem

My design looked clean — Philosopher, Chopstick, DiningTable. Each Chopstick wraps a ReentrantLock. The interviewer said 'Walk me through what happens when all 5 philosophers pick up their left chopstick at the same instant.' I said 'Each philosopher calls left.lock() then right.lock(). They'll just wait until the right chopstick is available.'

## Interview-ready answer

## Problem understanding

The Dining Philosophers problem is a classical concurrency problem used to illustrate synchronization challenges in multithreaded environments. The typical scenario involves N philosophers sitting around a circular table, each with a chopstick on their left and right. To eat, a philosopher needs to acquire both adjacent chopsticks (locks). The main challenge is to prevent **deadlock**, where every philosopher picks up their left chopstick and waits indefinitely for the right one, which is held by their neighbor.

The described design uses a `ReentrantLock` wrapped by each Chopstick and philosophers acquiring locks in the order: left chopstick first, then right. If all philosophers try to pick up their left chopstick simultaneously, they all hold one lock and wait on the second, which causes a **deadlock** shortly after they start eating attempts.

## Interview answer

The deadlock occurs due to a circular waiting condition: all philosophers grab their left chopstick (lock) simultaneously, causing each to wait indefinitely for the right chopstick held by their neighbor. To prevent this, we must break one of the four Coffman deadlock conditions:

1. **Mutual exclusion** — required, since chopsticks are shared resources.
2. **Hold and wait** — philosophers hold one chopstick while waiting for another.
3. **No preemption** — locks cannot be forcibly taken from a philosopher.
4. **Circular wait** — the circular chain of waiting philosophers.

Common deadlock prevention strategies in this problem include:

- **Impose an ordering:** Number chopsticks and always acquire locks in a consistent global order (e.g., lowest-numbered chopstick first, then higher). This breaks circular wait.
- **Allow at most N-1 philosophers to try to eat concurrently:** This prevents a circular locking condition.
- **TryLock with timeout/fallback:** If a philosopher can't pick up both chopsticks, release the first and retry later.
- **Asymmetric locking scheme:** For example, odd-numbered philosophers pick left then right, even-numbered pick right then left.

Explaining this shows understanding of deadlock causes and standard solutions, which is often expected in interviews.

## Java implementation

Here’s a concise but complete idiomatic Java implementation of the Dining Philosophers problem with **lock ordering** to prevent deadlock:

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Chopstick {
    private final Lock lock = new ReentrantLock();
    private final int id;

    public Chopstick(int id) {
        this.id = id;
    }

    public boolean pickUp() {
        return lock.tryLock();
    }

    public void putDown() {
        lock.unlock();
    }

    public int getId() {
        return id;
    }
}

class Philosopher implements Runnable {
    private final Chopstick left;
    private final Chopstick right;
    private final int id;

    public Philosopher(int id, Chopstick left, Chopstick right) {
        this.id = id;
        this.left = left;
        this.right = right;
    }

    @Override
    public void run() {
        try {
            while (true) {
                think();
                eat();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void think() throws InterruptedException {
        System.out.println("Philosopher " + id + " is thinking.");
        Thread.sleep((long) (Math.random() * 100));
    }

    private void eat() throws InterruptedException {
        // Acquire locks in order based on chopstick ID to prevent circular wait
        Chopstick first = left.getId() < right.getId() ? left : right;
        Chopstick second = left.getId() < right.getId() ? right : left;

        // Try to acquire the first chopstick lock
        first.lock.lock();
        try {
            // Wait briefly to increase likelihood of concurrency issues if not fixed
            Thread.sleep(10);

            // Acquire second chopstick lock
            second.lock.lock();
            try {
                System.out.println("Philosopher " + id + " is eating.");
                Thread.sleep((long) (Math.random() * 100));
            } finally {
                second.lock.unlock();
            }
        } finally {
            first.lock.unlock();
        }
    }
}

public class DiningTable {
    public static void main(String[] args) {
        int n = 5;
        Chopstick[] chopsticks = new Chopstick[n];
        for (int i = 0; i < n; i++) {
            chopsticks[i] = new Chopstick(i);
        }
        Thread[] philosophers = new Thread[n];
        for (int i = 0; i < n; i++) {
            Chopstick left = chopsticks[i];
            Chopstick right = chopsticks[(i + 1) % n];
            philosophers[i] = new Thread(new Philosopher(i, left, right));
            philosophers[i].start();
        }
    }
}
```

### Key points about the implementation

- Locks are acquired in **increasing chopstick ID order** rather than “always left then right.” This prevents circular wait.
- We use `ReentrantLock` directly here for simplicity.
- If we acquired first lock and can't get the second, with strict ordering, this will not deadlock.
- This approach assumes IDs are consistent across philosophers.
- The program runs indefinitely; to make it testable, you can run with limited iterations or add stop conditions.

## Key follow-up questions

- **Why does deadlock occur here?** Explain the circular wait and hold-and-wait conditions.
- **How does locking chopsticks in a global order break deadlock?** How does this guarantee no cycles?
- **What are alternate designs to avoid deadlock?** For example, using `tryLock()`, or odd/even philosopher rules.
- **What failure modes might still exist?** For example, starvation if unfair locking happens.
- **Is fairness guaranteed by ReentrantLock?** By default, no, but `new ReentrantLock(true)` enables fairness.
- **How would you test your concurrency design?** Discuss unit and stress testing.

## Takeaways

- The Dining Philosophers problem exemplifies classic deadlock caused by circular waiting on resources.
- Acquiring locks in a consistent global order is the simplest and most common practical solution to avoid deadlocks.
- Alternatives include limiting concurrency (N-1 philosopher rule), or using try locks to back off and retry.
- Understanding underlying classical concurrency problems and existing solutions is key for system design interviews.
- Implementations must consider deadlock, livelock, and starvation.
- Adding fairness policies, timed lock attempts, or ordered lock acquisition are critical concurrency design patterns.

Mastering such patterns will improve your system design and multithreading sophistication in interviews and real backend development.
