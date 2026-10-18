# The candidate had all the right buzzwords

> Automatically generated interview-preparation note.

## Original problem

I\’ll use Composite for files and folders, Factory for object creation, Proxy for access control, Decorator for permissions, Observer for notifications, Strategy for storage, and Command for undo/redo.

## Interview-ready answer

## Problem understanding

This problem involves designing a file system or similar hierarchical structure supporting various functionalities. The core challenge is to apply relevant design patterns to manage files and folders (composite structure), create objects, control access, handle permissions dynamically, notify observers of changes, select different storage strategies, and support undo/redo operations.

Constraints and design goals typically include:

- Handling hierarchical file/folder relationships cleanly.
- Abstracting object creation for flexibility and encapsulation.
- Controlling access permissions securely and dynamically.
- Supporting extensible permission models without altering core file/folder classes.
- Reacting to system events, such as file changes, with proper notifications.
- Allowing interchangeable storage backends (local disk, cloud, etc.).
- Enabling undo/redo for file system commands (create, delete, rename, move).

## Interview answer

To design a file system managing files and folders with advanced features, we can leverage well-known design patterns for clean and maintainable architecture:

1. **Composite pattern**: Model files and folders as components in a tree structure. Both expose a common interface (e.g., `FileSystemNode`) with operations like `add()`, `remove()`, and `getChildren()`. This encapsulates uniform handling of files and folders.

2. **Factory pattern**: Abstract file or folder creation through a factory. This decouples client code from concrete classes, provides flexibility in instantiating different file types, and supports future extensions.

3. **Proxy pattern**: Introduce an access control proxy that sits between clients and files/folders. Before delegating calls, it checks permissions, thereby centralizing access control without mixing with business logic.

4. **Decorator pattern**: Add permissions or metadata to file/folder objects dynamically by wrapping them with decorators. This prevents bloating the core classes and accommodates flexible permission layers.

5. **Observer pattern**: Attach observers (listeners) to files/folders or the file system to receive notifications on changes (content modification, permission updates). Useful for triggering UI refresh, logging, or caching.

6. **Strategy pattern**: Allow interchangeable storage implementations (local disk, distributed storage, cloud). The file system delegates storage operations to a strategy chosen at runtime or configuration.

7. **Command pattern**: Encapsulate actions (create, delete, rename, move) as command objects supporting execution, undo, and redo. This decouples operation logic from the UI and provides robust undo/redo capabilities.

### Trade-offs and reasoning:

- Using multiple patterns improves modularity and testability but increases initial complexity.
- Proxy and decorator patterns introduce additional wrapping layers; care must be taken to avoid performance hits and complexity.
- The observer pattern triggers are crucial but need careful management to prevent memory leaks (e.g., weak references).
- Command pattern improves flexibility but requires maintaining command history and handling failure/retry semantics.
- Strategy pattern allows switching storage providers but may require adapter code for different APIs.

Overall, combining these patterns strikes a balance between flexibility, maintainability, and scalability in a complex file system design.

## Java implementation

Below is a simplified skeleton illustrating key pattern integrations.

// Common interface for Files and Folders (Composite)
```java
public interface FileSystemNode {
    String getName();
    void add(FileSystemNode node);        // meaningful for folders
    void remove(FileSystemNode node);
    List<FileSystemNode> getChildren();   // meaningful for folders
    void display(int indent);              // for demo, print structure
}
```

// Leaf: File
```java
public class FileLeaf implements FileSystemNode {
    private String name;
    public FileLeaf(String name) { this.name = name; }

    @Override
    public String getName() { return name; }

    @Override
    public void add(FileSystemNode node) {
        throw new UnsupportedOperationException("Files cannot contain children");
    }

    @Override
    public void remove(FileSystemNode node) {
        throw new UnsupportedOperationException("Files cannot contain children");
    }

    @Override
    public List<FileSystemNode> getChildren() {
        return Collections.emptyList();
    }

    @Override
    public void display(int indent) {
        System.out.println(" ".repeat(indent) + name);
    }
}
```

// Composite: Folder
```java
public class FolderComposite implements FileSystemNode {
    private String name;
    private List<FileSystemNode> children = new ArrayList<>();

    public FolderComposite(String name) { this.name = name; }

    @Override
    public String getName() { return name; }

    @Override
    public void add(FileSystemNode node) { children.add(node); }

    @Override
    public void remove(FileSystemNode node) { children.remove(node); }

    @Override
    public List<FileSystemNode> getChildren() { return Collections.unmodifiableList(children); }

    @Override
    public void display(int indent) {
        System.out.println(" ".repeat(indent) + name + "/");
        for (FileSystemNode child : children) {
            child.display(indent + 2);
        }
    }
}
```

// Factory for file system nodes
```java
public class FileSystemFactory {
    public static FileSystemNode createFile(String name) {
        return new FileLeaf(name);
    }
    public static FileSystemNode createFolder(String name) {
        return new FolderComposite(name);
    }
}
```

// Proxy for access control
```java
public class AccessControlProxy implements FileSystemNode {
    private FileSystemNode realNode;
    private PermissionChecker permissionChecker;

    public AccessControlProxy(FileSystemNode realNode, PermissionChecker permissionChecker) {
        this.realNode = realNode;
        this.permissionChecker = permissionChecker;
    }

    private void checkAccess() {
        if (!permissionChecker.hasAccess(realNode)) {
            throw new SecurityException("Access denied to " + realNode.getName());
        }
    }

    @Override
    public String getName() {
        checkAccess();
        return realNode.getName();
    }

    @Override
    public void add(FileSystemNode node) {
        checkAccess();
        realNode.add(node);
    }

    @Override
    public void remove(FileSystemNode node) {
        checkAccess();
        realNode.remove(node);
    }

    @Override
    public List<FileSystemNode> getChildren() {
        checkAccess();
        return realNode.getChildren();
    }

    @Override
    public void display(int indent) {
        checkAccess();
        realNode.display(indent);
    }
}
```

// Decorator for permissions
```java
public abstract class FileSystemDecorator implements FileSystemNode {
    protected FileSystemNode decoratedNode;

    public FileSystemDecorator(FileSystemNode node) {
        this.decoratedNode = node;
    }

    @Override public String getName() { return decoratedNode.getName(); }
    @Override public void add(FileSystemNode node) { decoratedNode.add(node); }
    @Override public void remove(FileSystemNode node) { decoratedNode.remove(node); }
    @Override public List<FileSystemNode> getChildren() { return decoratedNode.getChildren(); }
    @Override public void display(int indent) { decoratedNode.display(indent); }
}

public class PermissionDecorator extends FileSystemDecorator {
    private Set<String> permissions = new HashSet<>();

    public PermissionDecorator(FileSystemNode node) {
        super(node);
    }

    public void addPermission(String permission) {
        permissions.add(permission);
    }

    public boolean hasPermission(String permission) {
        return permissions.contains(permission);
    }
}
```

// Observer for notifications (simplified)
```java
public interface FileSystemObserver {
    void onChanged(FileSystemNode node);
}

public class FileSystemSubject {
    private List<FileSystemObserver> observers = new ArrayList<>();

    public void addObserver(FileSystemObserver observer) {
        observers.add(observer);
    }

    public void notifyObservers(FileSystemNode node) {
        for (FileSystemObserver o : observers) {
            o.onChanged(node);
        }
    }
}
```

// Storage strategy interface
```java
public interface StorageStrategy {
    void save(FileSystemNode node);
    FileSystemNode load(String path);
}

public class LocalStorageStrategy implements StorageStrategy {
    @Override
    public void save(FileSystemNode node) {
        // local disk write logic
    }
    @Override
    public FileSystemNode load(String path) {
        // local disk read logic
        return null;
    }
}
```

// Command pattern
```java
public interface Command {
    void execute();
    void undo();
}

public class CreateFileCommand implements Command {
    private FileSystemNode parent;
    private FileSystemNode file;

    public CreateFileCommand(FileSystemNode parent, String filename) {
        this.parent = parent;
        this.file = FileSystemFactory.createFile(filename);
    }

    @Override
    public void execute() {
        parent.add(file);
    }

    @Override
    public void undo() {
        parent.remove(file);
    }
}
```

Additional classes such as `PermissionChecker` would encapsulate the access control logic; observers and command history can be managed by a controller component.

## Key follow-up questions

1. **Question:** How would you handle permissions inheritance in the file/folder hierarchy?  
   **Answer:** Permissions can be modeled in decorators with the ability to inherit from parent folders if not explicitly defined. During access checks, traverse upward the parent hierarchy until a permission setting is found or fallback to defaults.

2. **Question:** How do you ensure thread safety if multiple users concurrently modify files and folders?  
   **Answer:** Use appropriate synchronization or concurrent data structures when modifying the composite tree and while executing commands. Also, transactions or versioning can be introduced to maintain consistency.

3. **Question:** How would you prevent memory leaks with Observer pattern in this context?  
   **Answer:** Use weak references for observers or require explicit unsubscription to ensure observers are garbage collected when no longer in use.

4. **Question:** Could the Proxy pattern introduce performance overhead? How to mitigate?  
   **Answer:** Yes, each proxy call adds an extra layer. To mitigate, minimize heavy computations in the proxy and consider caching permission results or batch checking access.

5. **Question:** How do you implement undo/redo for more complex multi-step commands (e.g., moving a folder)?  
   **Answer:** Composite commands encapsulate multiple sub-commands. Each sub-command supports undo. The top-level command executes and undoes all sub-commands in order, maintaining atomicity as much as possible.

6. **Question:** What challenges arise from combining Decorator and Proxy patterns?  
   **Answer:** The layers can complicate debugging and stack traces. Clear separation of concerns and careful ordering (proxy can wrap decorator or vice versa depending on requirements) avoids confusion.

## Takeaways

- Leveraging multiple design patterns allows a clean separation of concerns for complex system architectures.
- Composite elegantly handles hierarchical structures of files/folders with uniform interfaces.
- Factory abstracts creation, proxy controls access transparently, and decorator adds flexibility in augmenting objects.
- Observer supports reactive programming but requires careful lifecycle management.
- Strategy enables extensible storage backends with pluggable implementations.
- Command encapsulates user operations with undo/redo capabilities, essential for rich UI experiences.
- Trade-offs include complexity overhead, layered wrapping performance, and lifecycle pitfalls.
- Clear documentation and test coverage are necessary to maintain quality in such multi-pattern systems.
