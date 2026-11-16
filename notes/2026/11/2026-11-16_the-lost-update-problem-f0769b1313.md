# ❌ The Lost Update Problem

> Automatically generated interview-preparation note.

## Original problem

Imagine you have $100. Simultaneously, you deposit $50 at an ATM while your spouse deposits $50 via an app. If your code isn't thread-safe, the CPU might interleave these operations so that one deposit overwrites the other. **Result? You have $150 instead of $200.**

## Interview-ready answer

## Problem understanding

The problem describes the classic **Lost Update Problem** in concurrent programming, specifically in the context of updating shared mutable state (e.g., a bank account balance). Two threads or processes simultaneously update the same value without proper synchronization, causing one update to overwrite the other and thus losing data.

**Constraints and design goals:**

- The shared resource is a bank account balance (an integer or decimal).
- Multiple concurrent updates (deposits) can occur at the same time.
- Updates must be atomic and isolated to prevent lost updates.
- The solution must be thread-safe, ensuring correctness despite concurrency.
- Minimize performance bottlenecks and allow reasonable scalability.

## Interview answer

To solve the lost update problem, we need to ensure that concurrent deposits do not interfere with each other, which requires **atomicity** and **mutual exclusion** in updating the balance.

### Key design points:

- **Atomicity:** Updates to the balance must be atomic so that the entire read-update-write sequence behaves as a single indivisible step.
  
- **Synchronization:** Use synchronization primitives like locks, synchronized blocks, or atomic variables to prevent race conditions.

### Possible solutions:

1. **Synchronized method/block:**  
   Synchronize the deposit operation so only one thread modifies the account balance at a time.
   
2. **Atomic variables (e.g., `AtomicInteger`, `AtomicLong`):**  
   Use atomic classes that provide thread-safe methods like `getAndAdd()`.
   
3. **Compare-and-swap (CAS) loop:**  
   Implement manual CAS to retry an update if another thread modifies the balance concurrently.

4. **Database-level locking/transaction:**  
   If the application is backed by a database, use ACID transactions with proper isolation levels to prevent lost updates.

### Trade-offs:

- **Coarse locking** can simplify code but may hurt concurrency and throughput.
- **Atomic variables** offer finer-grained control and better performance but only work well for simple values and updates.
- CAS loops require careful implementation to avoid livelocks.
- Database transactions introduce latency but are necessary in distributed or persisted settings.

### Summary:

The core idea is to protect the read-modify-write sequence so updates don’t overwrite one another, ensuring the final balance reflects all deposits correctly.

## Java implementation

Here is a simple idiomatic Java implementation of a thread-safe `BankAccount` using synchronization and an atomic variable approach.

```java
public class BankAccount {
    private int balance;

    public BankAccount(int initialBalance) {
        this.balance = initialBalance;
    }

    // Solution 1: Synchronized deposit
    public synchronized void deposit(int amount) {
        // Read current, add amount, write back
        balance += amount;
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

Alternatively, using `AtomicInteger`:

```java
import java.util.concurrent.atomic.AtomicInteger;

public class BankAccountAtomic {
    private AtomicInteger balance;

    public BankAccountAtomic(int initialBalance) {
        this.balance = new AtomicInteger(initialBalance);
    }

    public void deposit(int amount) {
        balance.getAndAdd(amount); // atomic update
    }

    public int getBalance() {
        return balance.get();
    }
}
```

### Explanation

- In the `BankAccount` class, the `deposit()` and `getBalance()` methods are synchronized to ensure mutual exclusion. Any thread entering `deposit` locks the object and prevents other threads from interleaving, guaranteeing atomic update.

- In the `BankAccountAtomic` class, `AtomicInteger.getAndAdd()` guarantees atomic addition, eliminating the need for explicit synchronization.

### Edge cases and failure modes

- Both approaches assume deposit amounts are positive; you might add input validation if needed.

- Synchronization causes thread blocking; atomic variables minimize blocking but are limited to simple operations.

- For more complex updates (e.g., multiple field changes), finer synchronization or transaction management is required.

- Beware of overflow if balance or deposit amounts are large.

## Key follow-up questions

1. **Q: What problems arise if the deposit method is not synchronized or atomic?**  
   A: Without synchronization, concurrent threads can read the same balance, add their amounts separately, and overwrite each other’s updates, causing lost updates and incorrect balances.

2. **Q: Why might you choose `AtomicInteger` over `synchronized`?**  
   A: `AtomicInteger` typically offers better performance by using low-level atomic CPU instructions, avoiding the overhead and contention of explicit locks. Suitable for simple numeric state updates.

3. **Q: How would you handle withdrawing money ensuring no overdraft occurs concurrently?**  
   A: Use synchronization or atomic operations with a check-then-update logic. For example, check balance first, then update atomically. In `AtomicInteger`, this could require a CAS loop retrying if balance changed. Alternatively, synchronize the whole withdrawal method.

4. **Q: How does database transaction isolation prevent lost update problems?**  
   A: Transactions with proper isolation levels (e.g., SERIALIZABLE or REPEATABLE READ) lock rows or detect conflicts, preventing two concurrent updates from overwriting each other without coordination.

5. **Q: What concurrency issues remain if you only synchronize the deposit method?**  
   A: If other methods modify balance without synchronization, race conditions persist. Also, synchronizing just deposit does not guarantee safe behavior if balance is directly accessible or update logic is fragmented.

6. **Q: Could you solve this with immutability?**  
   A: Immutability alone doesn't solve concurrent mutations but can simplify safe state sharing. You’d need an external synchronization or atomic reference replacement pattern for concurrent updates.

## Takeaways

- The lost update is a classic symptom of race conditions when multiple threads concurrently mutate shared state without atomicity.
- Use synchronization or atomic variables to ensure atomic read-modify-write operations.
- Choose synchronization solutions based on simplicity, contention, and performance needs.
- For complex transactions (withdrawals, transfers), proper locking or database transactions are essential.
- Always consider edge cases, such as invalid inputs and overflow.
- Understanding core concurrency concepts—atomicity, visibility, and ordering—is essential for writing correct, thread-safe code.
