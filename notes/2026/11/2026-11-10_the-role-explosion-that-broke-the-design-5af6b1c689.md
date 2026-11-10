# 👑 The Role Explosion That Broke the Design

> Automatically generated interview-preparation note.

## Original problem

A classic LLD interview trap.

## Interview-ready answer

## Problem understanding

The "Role Explosion" problem in a system design context typically refers to the rapid and unmanageable growth in the number of user roles and permissions, which breaks the design's scalability and maintainability. This often occurs when roles are used too granularly or rigidly, leading to duplication, complex hierarchies, and difficulty in managing role-based access controls (RBAC).

The problem constraints are:

- System must support multiple user roles with a variety of permissions.
- Roles can grow exponentially if not managed properly.
- Design should be scalable, maintainable, and flexible to changes.
- Must prevent permission explosion — a combinatorial blowup of role-permission configurations.

Design goals:

- Provide clear separation between users, roles, and permissions.
- Avoid creating a new role for every combination of permissions.
- Support role hierarchies or permission inheritance to reduce duplication.
- Enable dynamic permission management without requiring code changes.
- Ensure efficient permission checking at runtime.

## Interview answer

### Core design

1. **Entities**:
   - **User**: Assignable to one or more roles.
   - **Role**: Defines a collection of permissions.
   - **Permission**: Atomic access rights, e.g., `READ_REPORTS`, `EDIT_USER`.

2. **Avoid role explosion** by:
   - Using **permissions as the smallest unit**, and combining these into roles.
   - Supporting **role hierarchies or inheritance**, where roles can include other roles.
   - Using **attribute-based access control (ABAC)** for dynamic, context-based permissioning as an extension.
   - Implementing **groups** or **teams** to aggregate users beyond roles.

3. **Design approaches**:
   - **Flat RBAC**: Simple role-to-permission mappings. Risk of explosion if roles become too specific.
   - **Hierarchical RBAC**: Roles inherit permissions from parent roles, reducing duplications.
   - **Permission sets**: Define roles dynamically from sets of permissions, rather than fixing many roles.
   - **Policies**: Define fine-grained access rules referencing user attributes, resources, or environment.

4. **Trade-offs**:
   - Hierarchies increase complexity but reduce duplication.
   - ABAC adds flexibility but at the cost of complexity and performance.
   - Over-normalization might complicate runtime checks.
   - Storing permissions in cache improves performance but needs invalidation strategy.

### Reasoning

- The root cause of role explosion is modeling roles as fixed permission bundles without inheritance or modularity.
- Design must balance expressiveness (fine control) and manageability (limited role count).
- Plan for evolution: anticipate adding/removing permissions and roles without redesign.
- Provide admin tools for permission auditing and role management.

## Java implementation

Below is a simple design using hierarchical RBAC with roles, permissions, and users. It supports role inheritance to address role explosion:

```java
import java.util.*;

class Permission {
    private final String name;

    public Permission(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    // equals and hashCode based on name
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Permission)) return false;
        Permission other = (Permission) obj;
        return Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}

class Role {
    private final String name;
    private final Set<Permission> permissions = new HashSet<>();
    private final Set<Role> inheritedRoles = new HashSet<>();

    public Role(String name) {
        this.name = name;
    }

    public void addPermission(Permission p) {
        permissions.add(p);
    }

    public void inheritRole(Role role) {
        // Avoid cycles, could add cycle detection here
        inheritedRoles.add(role);
    }

    public Set<Permission> getAllPermissions() {
        Set<Permission> all = new HashSet<>(permissions);
        for (Role r : inheritedRoles) {
            all.addAll(r.getAllPermissions());
        }
        return all;
    }

    public String getName() {
        return name;
    }
}

class User {
    private final String id;
    private final Set<Role> roles = new HashSet<>();

    public User(String id) {
        this.id = id;
    }

    public void assignRole(Role role) {
        roles.add(role);
    }

    public boolean hasPermission(Permission p) {
        for (Role role : roles) {
            if (role.getAllPermissions().contains(p)) {
                return true;
            }
        }
        return false;
    }

    public String getId() {
        return id;
    }
}
```

### Explanation:

- `Permission` is atomic and immutable.
- `Role` contains permissions and may inherit from other roles.
- `getAllPermissions()` recursively gets all permissions considering inheritance.
- `User` holds roles and checks permissions by scanning all assigned roles.
- This design avoids exploding roles by letting inherited roles reuse permissions.

### Concurrency and scalability:

- This design is read-heavy: cache computed permissions for roles/users if performance matters.
- Permission and role modifications must be synchronized or done off-line with reloading.
- Cycle detection should be added in `inheritRole` to prevent infinite loops.

### Failure modes & edge cases:

- Cyclic role inheritance causing stack overflow.
- Missing permissions due to incomplete role assignments.
- Permission revocation delayed due to caching.
- Large inheritance graphs might affect performance; caching helps.

## Key follow-up questions

1. **Q:** How do you prevent cyclic inheritance in roles?  
   **A:** Implement cycle detection during the `inheritRole` call by checking if the role to inherit already inherits (directly or indirectly) the current role, preventing cyclic dependencies.

2. **Q:** What if a user can have conflicting permissions from different roles?  
   **A:** In RBAC, permissions are usually additive. If conflicts exist (e.g., allow vs deny), introduce explicit deny permissions and define evaluation order or priority. Alternatively, use an ABAC model or policy engines.

3. **Q:** How do you handle permission changes on roles dynamically?  
   **A:** Update the role permissions and invalidate any cached permissions. Users referencing that role will see updated permissions on next access. Use event-driven cache invalidation or versioning to maintain consistency.

4. **Q:** How would you extend this design to support context-aware access control?  
   **A:** Add an attribute-based layer where policies use user attributes, resource attributes, and environment context (time, location). This adds complexity but allows fine-grained dynamic decisions (ABAC).

5. **Q:** How do you efficiently check permissions in a system with thousands of users and roles?  
   **A:** Cache computed permissions for each user or role. Use bitsets or bloom filters to optimize permission checks. Also, consider external policy engines or directory services optimized for authorization queries.

## Takeaways

- The root cause of role explosion is poor abstraction: flatten roles and permissions or create rich inheritance.
- Use hierarchical RBAC to reuse permissions and reduce redundancy.
- Design should separate users, roles, and permissions distinctly.
- Always prepare for evolution, avoiding hardcoded roles or permissions.
- Include cycle detection and caching to maintain correctness and performance.
- Be aware of conflict resolution and policy complexity, possibly moving toward ABAC when RBAC limits are reached.
- In interviews, clearly state trade-offs, scalability considerations, and failure modes while proposing clean, maintainable solutions.
