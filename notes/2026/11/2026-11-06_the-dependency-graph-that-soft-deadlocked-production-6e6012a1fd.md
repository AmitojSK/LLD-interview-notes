# 🕸️ The Dependency Graph That Soft-Deadlocked Production

> Automatically generated interview-preparation note.

## Original problem

The scheduler supports dependencies: Job C runs only after Job A and Job B complete. Until the dependency graph becomes real.

## Interview-ready answer

## Problem understanding

We need to design a scheduler system that handles job dependencies given as a directed acyclic graph (DAG). Each job can start only after all of its dependencies (jobs it depends on) have completed. The scheduler must avoid deadlocks (especially soft deadlocks where jobs wait indefinitely), and ensure efficient, correct execution respecting the DAG constraints. The design goals include:

- Representing and managing the dependency graph correctly.
- Scheduling jobs such that dependent jobs run only after their dependencies complete.
- Detecting and preventing deadlocks or cyclic dependencies.
- Handling dynamic job completions and triggering dependent jobs.
- Ensuring scalability and concurrency correctness.

## Interview answer

To build a dependency-aware scheduler:

1. **Data Structures:**
   - Represent jobs as nodes in a graph.
   - Use adjacency lists to track dependencies (edges from dependencies to dependents).
   - Maintain an in-degree count (number of unmet dependencies) for each job.

2. **Initialization:**
   - Build the graph from input dependency data.
   - Calculate initial in-degree counts for each job.
   - Add all jobs with zero in-degree (no unmet dependencies) to a ready queue.

3. **Scheduling Logic:**
   - Use a thread-safe queue or worker pool to execute ready jobs.
   - When a job completes:
     - Decrement in-degree of all dependent jobs.
     - If dependent job's in-degree reaches zero, add it to the ready queue.
   
4. **Deadlock Prevention:**
   - Enforce acyclic constraints during graph setup.
   - Optionally, use cycle detection algorithms (DFS or Tarjan's algorithm) to validate the dependency graph before execution.

5. **Concurrency Considerations:**
   - Synchronize accesses to shared data like in-degree counts and ready queues using concurrent data structures or synchronization primitives.
   - Use thread pools to run jobs in parallel when possible (jobs with no mutual dependencies).

6. **Failures & Robustness:**
   - If jobs can fail, define failure handling (e.g., retry, abort dependents).
   - Handle partial execution and cancellation gracefully.

This design ensures that jobs only run after their dependencies complete, avoids soft deadlocks by verifying the absence of cycles and managing the execution order carefully. It also enables parallel execution of independent jobs.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class DependencyScheduler {

    static class Job implements Runnable {
        private final String id;
        private final Runnable task;

        public Job(String id, Runnable task) {
            this.id = id;
            this.task = task;
        }

        public String getId() {
            return id;
        }

        @Override
        public void run() {
            System.out.println("Running job " + id);
            task.run();
            System.out.println("Completed job " + id);
        }
    }

    private final ConcurrentHashMap<String, Job> jobs = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, List<String>> dependents = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, AtomicInteger> inDegree = new ConcurrentHashMap<>();
    private final ExecutorService executor;
    private final BlockingQueue<String> readyQueue = new LinkedBlockingQueue<>();

    public DependencyScheduler(int workerThreads) {
        this.executor = Executors.newFixedThreadPool(workerThreads);
    }

    // Add job without dependencies initially
    public void addJob(Job job) {
        jobs.putIfAbsent(job.getId(), job);
        dependents.putIfAbsent(job.getId(), new ArrayList<>());
        inDegree.putIfAbsent(job.getId(), new AtomicInteger(0));
    }

    // Define that jobId depends on dependencyId
    public void addDependency(String jobId, String dependencyId) {
        // Add edge dependencyId -> jobId
        dependents.computeIfAbsent(dependencyId, k -> new ArrayList<>()).add(jobId);
        // Increment jobId's in-degree
        inDegree.computeIfAbsent(jobId, k -> new AtomicInteger(0)).incrementAndGet();
        // Ensure all jobs exist in data structures
        inDegree.putIfAbsent(dependencyId, new AtomicInteger(0));
        dependents.putIfAbsent(jobId, new ArrayList<>());
    }

    // Detect cycle using DFS: returns true if cycle exists
    private boolean hasCycle() {
        Set<String> visited = new HashSet<>();
        Set<String> recursionStack = new HashSet<>();

        for (String jobId : jobs.keySet()) {
            if (dfsCycleDetect(jobId, visited, recursionStack)) {
                return true;
            }
        }
        return false;
    }

    private boolean dfsCycleDetect(String node, Set<String> visited, Set<String> recursionStack) {
        if (recursionStack.contains(node)) {
            return true; // cycle detected
        }
        if (visited.contains(node)) {
            return false;
        }
        visited.add(node);
        recursionStack.add(node);

        for (String dep : dependents.getOrDefault(node, Collections.emptyList())) {
            if (dfsCycleDetect(dep, visited, recursionStack)) {
                return true;
            }
        }
        recursionStack.remove(node);
        return false;
    }

    // Start execution
    public void start() {
        if (hasCycle()) {
            throw new IllegalStateException("Dependency graph contains cycle, cannot schedule");
        }

        // Enqueue all jobs with zero in-degree
        for (Map.Entry<String, AtomicInteger> entry : inDegree.entrySet()) {
            if (entry.getValue().get() == 0) {
                readyQueue.offer(entry.getKey());
            }
        }

        // Consume readyQueue and schedule jobs
        while (!readyQueue.isEmpty() || !executor.isTerminated()) {
            String jobId = null;
            try {
                // Wait for available ready job
                jobId = readyQueue.poll(100, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
            if (jobId != null) {
                Job job = jobs.get(jobId);
                executor.submit(() -> {
                    try {
                        job.run();
                    } finally {
                        onJobCompletion(job.getId());
                    }
                });
            } else {
                // No ready job currently - check if all done
                if (((ThreadPoolExecutor) executor).getActiveCount() == 0 && readyQueue.isEmpty()) {
                    executor.shutdown();
                }
            }
        }
    }

    private void onJobCompletion(String jobId) {
        List<String> dependentJobs = dependents.getOrDefault(jobId, Collections.emptyList());
        for (String dependentJobId : dependentJobs) {
            AtomicInteger count = inDegree.get(dependentJobId);
            if (count != null && count.decrementAndGet() == 0) {
                readyQueue.offer(dependentJobId);
            }
        }
    }

    public static void main(String[] args) {
        DependencyScheduler scheduler = new DependencyScheduler(4);

        // Define jobs
        scheduler.addJob(new Job("A", () -> sleep(100)));
        scheduler.addJob(new Job("B", () -> sleep(200)));
        scheduler.addJob(new Job("C", () -> sleep(150)));

        // Define dependencies: C depends on A and B
        scheduler.addDependency("C", "A");
        scheduler.addDependency("C", "B");

        scheduler.start();
    }

    private static void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException ignored) {}
    }
}
```

### Explanation

- Jobs are defined with a unique string ID.
- The dependency graph tracks dependents for each job and maintains in-degree counts.
- Before scheduling, a cycle check prevents soft deadlocks.
- As jobs complete, their dependents' in-degree is decremented; when it hits zero, dependent jobs become ready.
- A thread pool runs jobs in parallel safely.
- The ready queue is thread-safe (`LinkedBlockingQueue`) to coordinate between job completions and scheduling.

**Failure modes:** If the graph contains cycles, the scheduler refuses to start. If a job throws an exception, it's logged but currently does not propagate failure to dependent jobs. This could be enhanced according to requirements.

## Key follow-up questions

1. **Q:** How do you handle dynamic addition of jobs or dependencies after the scheduler has started?  
   **A:** The current design doesn't support dynamic changes post-start. To handle that, synchronization is needed when modifying graph structures. Also, we need logic to detect and resolve cycles dynamically and possibly re-update the ready queue accordingly.

2. **Q:** What if a job fails or throws an exception?  
   **A:** We can implement failure handling strategies like retries, marking dependent jobs as failed or skipped, or cascading failures. This depends on job criticality and business rules.

3. **Q:** How do you detect cycles in large graphs efficiently?  
   **A:** Use Tarjan's algorithm or Kahn's algorithm for cycle detection. For large graphs, incremental cycle detection or partial graph validation may be required.

4. **Q:** How do you ensure thread safety in a highly concurrent environment?  
   **A:** Use concurrent data structures (`ConcurrentHashMap`, `AtomicInteger`, `BlockingQueue`) and minimize locking. Critical sections should be carefully designed to prevent race conditions without harming scalability.

5. **Q:** Can the scheduler be distributed?  
   **A:** Distributing across machines requires consensus on graph state and job execution status. Techniques could include distributed locks, coordination via tools like Zookeeper, and persistent state.

## Takeaways

- Representing dependencies as a DAG with in-degree counts enables efficient scheduling.
- Cycle detection is critical to avoid soft deadlocks.
- Concurrent and thread-safe design is essential for correctness and scalability.
- Proper failure handling and dynamic updates complicate the design but are necessary for production readiness.
- The problem abstracts to a classic topological ordering combined with execution scheduling.
