# You're preparing for LLD wrong

> Automatically generated interview-preparation note.

## Original problem

The engineer in that story had studied hard. He just studied the wrong layer. There's a difference between knowing that Proxy .....

## Interview-ready answer

## Problem understanding
The core problem involves distinguishing between studying low-level design (LLD) and misunderstanding the focus of LLD preparation. The narrative highlights a common mistake: deeply studying infrastructure or protocol-specific internals (like Proxy design in networking) rather than focusing on the key concepts, abstractions, and design patterns crucial for system design and software architecture interviews.

**Constraints and goals:**

- Understand what aspects of LLD are most relevant for backend system design interviews.
- Clarify the difference between deep protocol knowledge (e.g., Proxy in networking) versus reusable design patterns and clean architecture.
- Emphasize how to prepare effectively for an LLD interview to demonstrate problem-solving skills, modular design, and scalability.

## Interview answer
When preparing for an LLD interview, many candidates err by focusing too narrowly on specific technical details or knowledge about a particular layer (e.g., how proxies work in network stacks or detailed protocol internals). While such knowledge could be valuable for specialized roles, it is not the primary focus in most backend LLD interviews.

The goal of LLD interviews is to evaluate your ability to:

- Translate requirements into modular, loosely coupled classes and interfaces.
- Design extensible systems by applying Object-Oriented Design (OOD) principles and design patterns.
- Handle scalability, concurrency, and fault tolerance in your design.
- Communicate your design clearly and justify your choices with tradeoff analysis.

Understanding how a proxy design might work on a network level isn't as helpful as, for example, knowing the Proxy design pattern in software engineering—how and when to apply it, and its pros and cons.

**Key focus areas should be:**

- Core OOD principles (SOLID, DRY, KISS).
- Design patterns (Factory, Singleton, Observer, Strategy, Proxy, etc.).
- Class and object relationships: composition vs. inheritance.
- Handling state, concurrency, and data consistency.
- Breaking down complexity via clear APIs and module boundaries.

Effectively, LLD preparation is about designing maintainable, testable, and extensible software components/classes — not about deep-dive familiarity with each protocol or system layer.

## Java implementation
Below is an idiomatic example illustrating the **Proxy design pattern** in Java, demonstrating the kind of design knowledge expected in an LLD interview rather than network-level proxy internals.

```java
// Subject interface
public interface Image {
    void display();
}

// RealSubject
public class RealImage implements Image {
    private String filename;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();
    }

    private void loadFromDisk() {
        System.out.println("Loading " + filename);
    }

    @Override
    public void display() {
        System.out.println("Displaying " + filename);
    }
}

// Proxy class controlling access to RealImage, lazy-loading it
public class ProxyImage implements Image {
    private RealImage realImage;
    private String filename;

    public ProxyImage(String filename) {
        this.filename = filename;
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename); // Expensive operation deferred
        }
        realImage.display();
    }
}
```

**Explanation:**

- `Image` interface defines the contract.
- `RealImage` is the heavy object (loading image from disk).
- `ProxyImage` delays creation of `RealImage` until needed, acting as a surrogate.
  
This pattern improves efficiency by deferring expensive operations, and a candidate should explain trade-offs: adding proxy overhead vs delayed resource consumption.

**Concurrency and edge cases:**

- If accessed from multiple threads, consider the proxy's thread-safety.
- For example, synchronize initialization of `realImage` if lazy-loading concurrently.
- Handle exceptions during loading gracefully.

This example communicates core LLD skills: defining interfaces, using design patterns, considering performance and concurrency implications.

## Key follow-up questions
1. **Q:** How would you modify the Proxy pattern to be thread-safe?  
   **A:** Use synchronized blocks or `volatile` with double-checked locking to ensure safe lazy initialization in multithreaded environments. Alternatively, use initialization-on-demand holder idiom or eager initialization depending on requirements.

2. **Q:** Can you explain the difference between the Proxy pattern and the Decorator pattern?  
   **A:** Proxy controls access to an object, often adding lazy loading or security checks, and typically matches the interface of the real object. Decorator adds additional behavior or responsibilities to the object dynamically. Both share interface but differ in intent.

3. **Q:** When is using a Proxy pattern not appropriate?  
   **A:** If the object creation cost is negligible, or if access control or delayed initialization adds unnecessary complexity, using Proxy might be overengineering.

4. **Q:** How do you ensure that your LLD scales when requirements grow?  
   **A:** By designing loosely coupled, cohesive modules with clear interfaces, and applying SOLID principles; also by anticipating extension points and minimizing changes to existing code via abstractions.

5. **Q:** How do you balance deep technical knowledge with design skills in an interview?  
   **A:** Focus on communicating your understanding of abstraction and design patterns, and how you apply them to solve problems, rather than memorizing implementation details of specific protocols or systems.

## Takeaways
- LLD interviews test your ability to design modular, extensible software systems, not deep protocol knowledge.
- Focus study on OOD principles and commonly used design patterns with clear understanding of their purpose and tradeoffs.
- Prepare to discuss concurrency, testing, scalability, and failure modes at the design level.
- Demonstrating clarity and reasoning is as important as naming patterns or concepts.
- Avoid overfitting your study to a particular "layer" of technology; broaden your design fundamentals instead.
