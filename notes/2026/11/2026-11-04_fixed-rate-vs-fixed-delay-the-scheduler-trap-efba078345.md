# 🧨 Fixed Rate vs Fixed Delay: The Scheduler Trap

> Automatically generated interview-preparation note.

## Original problem

Fixed-rate is a timing contract. Fixed-delay is a safety contract.

## Interview-ready answer

## Problem understanding
The problem involves understanding two scheduling strategies commonly used in task schedulers and timers: **fixed-rate** and **fixed-delay** scheduling. Both manage periodic task execution, but differ fundamentally in their timing semantics:

- **Fixed-rate scheduling** aims to execute tasks at consistent intervals based on a reference start time, regardless of task execution duration delays.
- **Fixed-delay scheduling** ensures a fixed delay between the completion of one task execution and the start of the next.

The constraints are:

- Tasks may run longer or shorter, influencing timing outcomes.
- The system needs to guarantee timing precision (fixed-rate) or timing safety and avoidance of task overlap (fixed-delay).
- Handling potential delays, concurrent executions, and cumulative drift over time.

Design goals:

- Understand when to use each scheduling type based on timing requirements and task execution properties.
- Avoid pitfalls like task overlap and unintended drift.
- Provide a clear conceptual and practical framework for implementing these schedules in systems such as Java’s `ScheduledExecutorService`.

## Interview answer
The key difference between **fixed-rate** and **fixed-delay** scheduling lies in the definition of the timing contract:

### Fixed-rate scheduling: Timing contract
- The goal is to run tasks at **precise time intervals**, measured from a fixed start time.
- For example, if the start time is T=0 and interval is 5 seconds, the task should ideally run at times 0, 5, 10, 15, ...
- If a task execution takes longer than the interval, the scheduler tries to catch up by executing the next task immediately to maintain the rate.
- This can cause **task overlap** or concurrent runs if tasks take longer than the period.
- Suitable for **periodic tasks requiring strict cadence**, such as data sampling, animations, or heartbeat signals.

### Fixed-delay scheduling: Safety contract
- The delay between the end of one execution and the start of the next is fixed.
- For example, a delay of 5 seconds means if a task finishes at time T=7, the next starts at T=12.
- This avoids task overlap because the next execution waits until the previous fully completes plus the fixed delay.
- The actual *interval* between starts can vary, especially if task execution time varies. So the periodicity drifts if executions are long.
- Suitable for **tasks where sequential execution matters, and no concurrent executions should happen**, like cleanup jobs or retries.

### Trade-offs and pitfalls:
- **Fixed-rate** may lead to concurrent executions if tasks are slow, possibly causing thread starvation or data races.
- **Fixed-delay** prevents concurrency but may cause schedule drift.
- Choosing depends on tolerance for drift, necessity of strict intervals, and the problems from concurrent task runs.

In Java, these semantics are exposed by `ScheduledExecutorService` via `scheduleAtFixedRate()` and `scheduleWithFixedDelay()`. It's important to pick the one aligned with your timing and safety requirements.

## Java implementation
```java
import java.util.concurrent.*;

public class SchedulerComparison {

    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

    // Fixed-rate scheduling example:
    public void fixedRateTask(Runnable task, long initialDelay, long period, TimeUnit unit) {
        scheduler.scheduleAtFixedRate(task, initialDelay, period, unit);
    }

    // Fixed-delay scheduling example:
    public void fixedDelayTask(Runnable task, long initialDelay, long delay, TimeUnit unit) {
        scheduler.scheduleWithFixedDelay(task, initialDelay, delay, unit);
    }

    // Shutdown scheduler gracefully
    public void shutdown() {
        scheduler.shutdown();
        try {
            if (!scheduler.awaitTermination(5, TimeUnit.SECONDS)) {
                scheduler.shutdownNow();
            }
        } catch (InterruptedException e) {
            scheduler.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    // Example task to simulate work
    public static Runnable createTask(String name, long workTimeMillis) {
        return () -> {
            System.out.println("Task " + name + " started at " + System.currentTimeMillis());
            try {
                Thread.sleep(workTimeMillis);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("Task " + name + " finished at " + System.currentTimeMillis());
        };
    }

    public static void main(String[] args) throws InterruptedException {
        SchedulerComparison sc = new SchedulerComparison();

        // Fixed rate scheduling example:
        sc.fixedRateTask(createTask("A", 3000), 0, 5000, TimeUnit.MILLISECONDS);

        // Fixed delay scheduling example:
        sc.fixedDelayTask(createTask("B", 3000), 0, 5000, TimeUnit.MILLISECONDS);

        // Let the scheduler run for some time
        Thread.sleep(20000);

        sc.shutdown();
    }
}
```
### Explanation:
- `scheduleAtFixedRate` enforces a fixed cadence, attempts to keep tasks aligned with the schedule even if previous executions overlap.
- `scheduleWithFixedDelay` waits until the previous execution finishes plus a fixed delay before restarting.
- The example tasks deliberately take 3 seconds to run with a 5-second interval/delay to demonstrate behavior.
- Edge cases include:
  - Tasks running longer than the fixed rate could cause overlapping tasks.
  - In fixed delay, variable task duration shifts the effective interval lazily.
- Proper shutdown ensures no abrupt termination.

## Key follow-up questions

1. **Question**: What happens if a fixed-rate scheduled task takes longer than the period?
   **Answer**: The scheduler tries to start the next execution immediately after the previous completes, which can cause tasks to overlap or run concurrently, potentially leading to resource contention or race conditions.

2. **Question**: Can `scheduleAtFixedRate` cause task starvation?
   **Answer**: Yes, if tasks consistently overrun the period and the scheduler attempts to catch up on delayed executions, threads may be occupied continuously, starving other tasks.

3. **Question**: How do you prevent overlapping executions in fixed-rate scheduling?
   **Answer**: You can manage it by using locks inside your task to ensure mutual exclusion or consider using fixed-delay scheduling when task overlap is unacceptable.

4. **Question**: Why would one choose fixed-delay over fixed-rate?
   **Answer**: Fixed-delay guarantees no task overlap and is safer for tasks that must finish before starting again, even at the cost of drifting timings.

5. **Question**: How do these scheduling approaches handle daylight saving time or system clock changes?
   **Answer**: Both rely on relative timing typically based on system monotonic clocks for scheduling; however, sudden system time changes may affect fixed-rate more visibly, potentially causing unexpected timing behavior.

## Takeaways
- Understand the distinction: fixed-rate enforces **strict periodicity**; fixed-delay enforces **safe sequencing**.
- Use fixed-rate for tasks needing precise timing (e.g., media sync, periodic data collection).
- Use fixed-delay for tasks where safety and serialized execution matter, tolerating some timing drift.
- Beware of overlapping executions and resource exhaustion in fixed-rate scheduling.
- Java's `ScheduledExecutorService` cleanly supports both, making it easy to implement once design goals are clear.
- Proper shutdown and error handling are crucial to prevent resource leaks and ensure graceful termination.
