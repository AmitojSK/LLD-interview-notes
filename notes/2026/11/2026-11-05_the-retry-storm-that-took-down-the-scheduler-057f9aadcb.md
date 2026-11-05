# 🌪️ The Retry Storm That Took Down the Scheduler

> Automatically generated interview-preparation note.

## Original problem

A payment job fails due to a transient network glitch.

## Interview-ready answer

## Problem understanding
The problem involves a payment job failing temporarily (e.g., a transient network glitch), triggering retries. However, poorly designed retry mechanisms can cause a "retry storm," where numerous retry attempts happen in quick succession, overwhelming the scheduler or downstream systems. The goal is to design a robust, scalable retry mechanism that handles transient failures gracefully without causing cascading failures or overloading system resources.

Constraints and design goals include:  
- Handling transient errors via retries with backoff (timing control).  
- Avoiding thundering herd problems from multiple failing jobs retrying simultaneously.  
- Ensuring system stability and fairness in job scheduling.  
- Configurability of retry policies (max attempts, backoff strategy).  
- Observability for failures and retries (logging, metrics).  
- Thread-safety and concurrency control in the scheduler implementation.

## Interview answer

**Core design points:**

1. **Retry Policy Interface:**
   Implement a pluggable retry policy to control retry behavior such as max retries, backoff strategy (fixed delay, exponential backoff with jitter), and retryable error types.

2. **Backoff Strategy:**
   Use exponential backoff combined with random jitter to spread retry requests over time, avoiding synchronized retries that cause retry storms.

3. **Scheduler Control:**
   - Limit the number of concurrent retries globally or per job to avoid overload.
   - Maintain retry queues with prioritization to avoid starving new jobs.
   - Use delay queues (e.g., `DelayQueue` in Java) or scheduled executors to schedule retries at their intended time.

4. **State Management:**
   Keep retry count, last attempt time, and job state atomically managed per job, ensuring thread-safe updates.

5. **Failure Classification:**
   Distinguish transient vs permanent failures. Retry only transient failures to avoid endless loops.

6. **Circuit Breaker Integration:**
   Optionally, implement circuit breakers to stop retrying jobs after repeated failures, allowing manual or external intervention.

7. **Observability:**
   Log retry attempts, failures, and successes, and expose metrics for monitoring retry storms or anomalies.

**Trade-offs:**

- **Complexity vs simplicity:** Advanced backoff and jitter add complexity but improve system stability.
- **Delay vs responsiveness:** Longer backoff increases delay for job completion but reduces load.
- **Memory:** Keeping state per job may increase resource usage.
- **Latency:** Exponential backoff increases latency for recovery but prevents overloading.
  
By combining these approaches, the system avoids retry storms while ensuring transient faults are handled effectively.

## Java implementation

```java
import java.util.Random;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

// Enum for failure types
enum FailureType {
    TRANSIENT, PERMANENT
}

// RetryPolicy defines max attempts and backoff strategy
interface RetryPolicy {
    int getMaxAttempts();
    long nextBackoffMillis(int retryCount);
    boolean isRetryable(FailureType failureType);
}

// Exponential backoff with jitter implementation
class ExponentialBackoffRetryPolicy implements RetryPolicy {
    private final int maxAttempts;
    private final long baseDelayMillis;
    private final long maxDelayMillis;
    private final Random random = new Random();

    public ExponentialBackoffRetryPolicy(int maxAttempts, long baseDelayMillis, long maxDelayMillis) {
        this.maxAttempts = maxAttempts;
        this.baseDelayMillis = baseDelayMillis;
        this.maxDelayMillis = maxDelayMillis;
    }

    @Override
    public int getMaxAttempts() {
        return maxAttempts;
    }

    @Override
    public long nextBackoffMillis(int retryCount) {
        // Exponential backoff: base * 2^(retryCount-1)
        long delay = baseDelayMillis * (1L << (retryCount - 1));
        delay = Math.min(delay, maxDelayMillis);
        // Add jitter: random between 75% and 125%
        double jitterFactor = 0.75 + 0.5 * random.nextDouble();
        return (long) (delay * jitterFactor);
    }

    @Override
    public boolean isRetryable(FailureType failureType) {
        return failureType == FailureType.TRANSIENT;
    }
}

// Represents a payment job with retry state
class PaymentJob implements Comparable<PaymentJob> {
    private final String jobId;
    private final Runnable task;
    private final RetryPolicy retryPolicy;

    private AtomicInteger retryCount = new AtomicInteger(0);
    private long nextRetryTimeMillis = 0;

    public PaymentJob(String jobId, Runnable task, RetryPolicy retryPolicy) {
        this.jobId = jobId;
        this.task = task;
        this.retryPolicy = retryPolicy;
    }

    public String getJobId() { return jobId; }

    // Schedule next retry time in millis
    public void scheduleNextRetry() {
        int attempt = retryCount.incrementAndGet();
        if (attempt > retryPolicy.getMaxAttempts()) {
            nextRetryTimeMillis = -1; // no more retries
        } else {
            long backoff = retryPolicy.nextBackoffMillis(attempt);
            nextRetryTimeMillis = System.currentTimeMillis() + backoff;
        }
    }

    public boolean canRetry() {
        return nextRetryTimeMillis > 0 && retryCount.get() < retryPolicy.getMaxAttempts();
    }

    public long getNextRetryTimeMillis() {
        return nextRetryTimeMillis;
    }

    public int getRetryCount() {
        return retryCount.get();
    }

    public Runnable getTask() {
        return task;
    }

    // Required to order jobs in delay queue (by next retry time)
    @Override
    public int compareTo(PaymentJob other) {
        return Long.compare(this.nextRetryTimeMillis, other.nextRetryTimeMillis);
    }
}

// Delayed wrapper for PaymentJob to use in DelayQueue
class DelayedPaymentJob implements Delayed {
    private final PaymentJob job;

    public DelayedPaymentJob(PaymentJob job) {
        this.job = job;
    }

    public PaymentJob getJob() {
        return job;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        long delayMillis = job.getNextRetryTimeMillis() - System.currentTimeMillis();
        return unit.convert(delayMillis, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed other) {
        if (other instanceof DelayedPaymentJob) {
            return job.compareTo(((DelayedPaymentJob) other).getJob());
        }
        return Long.compare(getDelay(TimeUnit.MILLISECONDS), other.getDelay(TimeUnit.MILLISECONDS));
    }
}

// Scheduler that manages job executions and retries
class RetryScheduler {
    private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(4);
    private final DelayQueue<DelayedPaymentJob> retryQueue = new DelayQueue<>();

    public RetryScheduler() {
        // Worker thread to take jobs from retry queue and execute
        Thread retryWorker = new Thread(this::processRetries);
        retryWorker.setDaemon(true);
        retryWorker.start();
    }

    // Execute a job with retry logic
    public void executeJob(PaymentJob job) {
        executor.submit(() -> {
            try {
                job.getTask().run();
                System.out.println("Job " + job.getJobId() + " succeeded");
                // success, no retry needed
            } catch (Exception e) {
                FailureType failureType = classifyFailure(e);
                if (job.retryPolicy.isRetryable(failureType) && job.canRetry()) {
                    job.scheduleNextRetry();
                    System.out.println("Job " + job.getJobId() + " failed transiently. Retry #"
                            + job.getRetryCount() + " scheduled in " +
                            (job.getNextRetryTimeMillis() - System.currentTimeMillis()) + "ms");
                    retryQueue.offer(new DelayedPaymentJob(job));
                } else {
                    System.out.println("Job " + job.getJobId() + " failed permanently or retries exhausted.");
                    // Handle permanent failure (alert, dead letter queue, etc.)
                }
            }
        });
    }

    // Continuously processes retry queue
    private void processRetries() {
        while (true) {
            try {
                DelayedPaymentJob delayedJob = retryQueue.take(); // block until next retry ready
                executeJob(delayedJob.getJob());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    // Example failure classifier: simplistic
    private FailureType classifyFailure(Exception e) {
        if (e instanceof java.net.SocketTimeoutException ||
                e instanceof java.net.ConnectException) {
            return FailureType.TRANSIENT;
        }
        return FailureType.PERMANENT;
    }

    public void shutdown() {
        executor.shutdownNow();
    }
}
```

### Explanation

- `ExponentialBackoffRetryPolicy` controls backoff with max attempts and jitter to prevent retry storms.
- `PaymentJob` encapsulates a job and retry state, knows how to schedule next retry.
- `RetryScheduler` manages execution and retry queue using a thread pool and a delay queue for scheduling retries at correct times.
- Exceptions are classified, transient failures trigger retry with backoff, permanent failures stop retrying.
- The delay queue delays re-submitting retry attempts, smoothing out spikes in load.
- Thread-safe `AtomicInteger` maintains retry count; the scheduler uses concurrency-safe queue structures.

**Failure modes handled:**  
- Transient network failure triggers retry with increasing delay.  
- Permanent failure or retry exhaustion stops retries.  
- Scheduler remains stable by delaying retries and limiting concurrency via thread pool.

## Key follow-up questions

1. **Q:** How would you handle retries for different failure types or externally configured retry policies?  
   **A:** Make retry policies pluggable and configurable via dependency injection or config files. Allow failure classification to be customizable. For example, inject classifiers or policy factories per job type.

2. **Q:** What if the system restarts? How do you ensure retry state is not lost?  
   **A:** Persist retry state (count, next retry time) externally (database, durable queue) so that jobs can resume retries after restarts. The scheduler must reload retry jobs from persistent storage on startup.

3. **Q:** How do you avoid retry storms from large numbers of jobs failing simultaneously (e.g., outage)?  
   **A:** Exponential backoff with jitter spreads out retries. Additional global throttling or circuit breakers can detect mass failure and pause or slow retries globally to protect system stability.

4. **Q:** What are alternatives to polling retry via delay queues?  
   **A:** Use message brokers with delayed message support (e.g., Kafka with delayed topics) or scheduled jobs persistent in DB with a poller that picks only jobs due for retry.

5. **Q:** How do you monitor and alert on retry storms or failing jobs?  
   **A:** Instrument retry attempts, success/failure counts, and latency via metrics systems (e.g., Prometheus). Alert when retry counts spike abnormally or when jobs exhaust retries too frequently.

## Takeaways

- Naive retry loops cause retry storms that can cascade failures and overwhelm schedulers.  
- Use exponential backoff combined with jitter to space retries and reduce retry storms.  
- Separate transient vs permanent failures to avoid useless retries.  
- Delay queues and scheduled executor services are common patterns to handle retries efficiently.  
- Persist retry state for reliability across restarts.  
- Add observability for retries and failures to detect issues early.  
- Design trade-offs center around balancing response latency vs system stability.
