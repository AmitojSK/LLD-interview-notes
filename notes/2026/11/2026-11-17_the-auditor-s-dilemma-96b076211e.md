# THE AUDITOR'S DILEMMA

> Automatically generated interview-preparation note.

## Original problem

Imagine you have $100. Simultaneously, you deposit $50 at an ATM while your spouse deposits $50 via an app. If your code isn't thread-safe, the CPU might interleave these operations so that one deposit overwrites the other. **Result? You have $150 instead of $200.**

## Interview-ready answer

## Problem understanding
We need to design a thread-safe account deposit system where multiple concurrent updates (deposits) to the same bank account can happen simultaneously. The core challenge is to ensure atomicity and consistency of operations so that race conditions do not cause lost updates. For example, starting with a balance of $100, two concurrent deposits of $50 each should result in a final balance of $200, not $150 due to an interleaving race where one update overwrites the other.

Key constraints and goals:  
- Support concurrent deposits from multiple threads safely  
- Avoid lost updates and ensure the balance reflects all completed deposits  
- Maintain data consistency and correctness under concurrency  
- Efficiency and scalability under contention are desirable but correctness trumps performance  

## Interview answer
To ensure thread safety when concurrently modifying an account balance, we must guarantee that updates are atomic and isolated. There are several approaches:

1. **Synchronized methods or blocks**  
   Use `synchronized` keyword on deposit operations or on a common lock object to serialize updates. This ensures only one thread can modify the balance at a time, preventing lost updates.  
   - Pros: Simple, correct  
   - Cons: Can lead to contention and reduced throughput under high concurrency  

2. **`java.util.concurrent.atomic` classes (e.g., AtomicInteger, AtomicLong)**  
   Use atomic operations like `getAndAdd` that update the balance without blocking. These implementations use low-level CPU atomic instructions (CAS).  
   - Pros: Efficient, non-blocking, scalable  
   - Cons: Limited to numeric types and simple arithmetic, cannot easily handle complex logic  

3. **Locking with `ReentrantLock`**  
   Explicit locks can provide flexibility such as try-lock, fairness policies, etc.  
   - Pros: More control than synchronized  
   - Cons: More verbose code  

4. **Using higher-level transactional or optimistic concurrency control**  
   For example, use database transactions or versioning. This is more relevant in distributed or persistent stores.

**Trade-offs:**  
- Synchronization is easiest but may bottleneck concurrency.  
- Atomic variables provide high performance but only for simple atomic updates.  
- Locking gives flexibility but complexity.  
- For banking systems, correctness is paramount, so sometimes simplicity is preferred to ensure correctness.

Given the problem (simple concurrent deposits), using `AtomicInteger` or `AtomicLong` to represent the balance is an effective and elegant solution.

## Java implementation

```java
import java.util.concurrent.atomic.AtomicInteger;

public class BankAccount {
    private final AtomicInteger balance;

    public BankAccount(int initialBalance) {
        this.balance = new AtomicInteger(initialBalance);
    }

    // Deposit amount atomically
    public void deposit(int amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("Deposit amount must be non-negative");
        }
        balance.getAndAdd(amount);
    }

    // Retrieve current balance
    public int getBalance() {
        return balance.get();
    }
}
```

### Explanation:
- The `AtomicInteger` balance ensures that `getAndAdd` is atomic—no two threads can interleave and cause lost updates.  
- Validation checks for negative deposit amounts to prevent invalid states.  
- `getBalance()` is a simple atomic read.

### Edge cases and failure modes
- Negative deposits can be rejected as above.  
- Overflow of integer if very large deposits occur — could switch to `AtomicLong` or use `BigInteger` with external synchronization if needed.  
- The current design does not include withdrawal or other features; those need similar thread-safe treatment.  
- Does not handle persistence or crashes; it’s an in-memory example.  

## Key follow-up questions

1. **Question:** What happens if multiple threads call `deposit` concurrently without synchronization?  
   **Answer:** Without synchronization or atomic operations, race conditions can occur where some deposits overwrite others, leading to lost updates and incorrect balances.

2. **Question:** How does `AtomicInteger.getAndAdd()` ensure atomicity?  
   **Answer:** It uses low-level CPU instructions such as Compare-And-Swap (CAS) to guarantee that the update is atomic, retrying if interference is detected, ensuring no lost updates.

3. **Question:** How would you also support withdrawals in a thread-safe manner?  
   **Answer:** You could implement withdrawal methods similarly but also need to ensure that withdrawals do not overdraw the account, which requires atomic check-and-update operations (e.g., using `compareAndSet` loops or explicit locks to check balance before updating).

4. **Question:** What if deposits come from distributed systems and you need persistence?  
   **Answer:** You would typically rely on ACID transactional guarantees from a database, possibly using optimistic concurrency control, locks, or transactional frameworks to maintain consistency.

5. **Question:** Are there any downsides to using `AtomicInteger` for account balance?  
   **Answer:** It's limited to simple numeric updates; complex transactions or invariant enforcement across multiple fields require more sophisticated concurrency control.

## Takeaways
- Concurrent updates to shared data require atomic and thread-safe operations to prevent lost updates.  
- Java provides simple atomic classes (`AtomicInteger`, etc.) ideal for numeric counter-like operations.  
- Synchronization is simpler but can reduce scalability.  
- Accounting logic often requires additional validation and invariants that may complicate concurrency.  
- Always validate inputs and consider the full lifecycle of operations (deposits, withdrawals, etc.) consistently.  
- Consider real-life constraints like persistence and distributed systems for complex scenarios.
