---
description: 'JDK 25 LTS addendum to java-functional-v2_instructions.md — covers only stable functional features introduced between JDK 22 and JDK 25. Apply together with java-functional-v2_instructions.md.'
applyTo: '**/*.java'
---

# Java Functional Programming — JDK 25 LTS Addendum

> **How to use this file**
> This file complements `java-functional-v2_instructions.md` (JDK 21 LTS baseline) and `java-core-jdk25_instructions.md` (core JDK 25 additions).
> All JDK 21 principles — immutability, pure functions, `Result<T, E>`, sealed ADTs, `Optional` chains — remain in force and are not repeated here.
> Apply the rules in this file only when the project build explicitly targets JDK 25 (`<java.version>25</java.version>` or `sourceCompatibility = JavaVersion.VERSION_25`).
>
> **Preview in JDK 25 — do not use in production:** Structured Concurrency (JEP 505), Primitive Types in Patterns (JEP 507), Stable Values (JEP 502), PEM Encodings (JEP 470).

---

## JDK 25 Additions to the Functional Baseline

| Feature | JEP | Impact on Functional Code |
| :--- | :--- | :--- |
| Unnamed Variables & Patterns `_` | JEP 456 (stable JDK 22) | Cleaner pattern deconstruction in `Result`, sealed ADTs, and lambda parameters |
| `Stream.gather()` — Stream Gatherers | JEP 485 (stable JDK 24) | Custom stateful intermediate stream operations without breaking the pipeline |
| Scoped Values | JEP 506 | Immutable, bounded context — replaces `ThreadLocal` in functional pipelines |
| Flexible Constructor Bodies | JEP 513 | Validation before `super()` without static helpers in class hierarchies |

For `_` basics (catch blocks, simple lambda parameters), see `java-core-jdk25_instructions.md`.

---

## Unnamed Variables (`_`) in Functional Patterns

The unnamed variable integrates naturally with `Result<T,E>` and sealed ADTs to eliminate noise when only some fields of a record variant are needed.

### In `Result<T, E>` switches

```java
// JDK 21: named binding even when unused
String respond(Result<Order, OrderError> result) {
    return switch (result) {
        case Result.Ok(var order)                         -> "Order: " + order.id();
        case Result.Err(OrderError.OutOfStock(var items)) -> "No stock";  // 'items' unused
        case Result.Err(OrderError.InvalidCart(var msg))  -> "Invalid: " + msg;
    };
}

// JDK 25: _ signals the field is intentionally ignored
String respond(Result<Order, OrderError> result) {
    return switch (result) {
        case Result.Ok(var order)                        -> "Order: " + order.id();
        case Result.Err(OrderError.OutOfStock(_))        -> "No stock";
        case Result.Err(OrderError.InvalidCart(var msg)) -> "Invalid: " + msg;
    };
}
```

### In sealed ADT pattern matching

```java
// Good: Partial deconstruction — only 'reason' is needed from Failure
String message(PaymentResult result) {
    return switch (result) {
        case PaymentResult.Success(var id, var amt) -> "Paid %s: %s".formatted(id, amt);
        case PaymentResult.Failure(var reason, _)   -> "Failed: " + reason;
    };
}
```

### Naming convention — `_` addendum

Update the naming table from `java-functional-v2_instructions.md`:

| Identifier | JDK 21 | JDK 25 |
| :--- | :--- | :--- |
| Unused catch parameter | `IOException ignored` | `IOException _` |
| Unused lambda parameter | `ignored` or `__` | `_` |
| Unused pattern binding | `var unused` | `_` in deconstruction |

---

## Stream Gatherers — `Stream.gather()` (JEP 485)

`Stream.gather(Gatherer)` is a new intermediate operation for custom, stateful transformations within a declarative pipeline. It fills the gap between `map`/`filter`/`flatMap` and breaking out into an imperative loop.

**When to use `gather()` over standard operations:**
- Windowing (sliding or tumbling windows over elements)
- Stateful scanning (running totals, state machines)
- Custom folding without splitting the pipeline into multiple passes

### Built-in Gatherers

```java
// Sliding window of size 3 — keeps pipeline declarative
List<List<Integer>> windows = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.windowSliding(3))
    .toList();
// Result: [[1,2,3], [2,3,4], [3,4,5]]

// Fixed-size tumbling (non-overlapping) windows
List<List<Integer>> chunks = Stream.of(1, 2, 3, 4, 5, 6)
    .gather(Gatherers.windowFixed(2))
    .toList();
// Result: [[1,2], [3,4], [5,6]]

// Running total as a scan — stateful, stays in the pipeline
List<Integer> runningTotals = Stream.of(1, 2, 3, 4)
    .gather(Gatherers.scan(() -> 0, Integer::sum))
    .toList();
// Result: [1, 3, 6, 10]
```

### Custom Gatherers

Implement `Gatherer<T, A, R>` for domain-specific stateful operations. A Gatherer has four optional components: `initializer`, `integrator`, `combiner`, and `finisher`.

```java
// Good: Custom Gatherer — emit only elements where value increased (monotonic filter)
Gatherer<Integer, int[], Integer> onlyIncreasing = Gatherer.of(
    () -> new int[]{Integer.MIN_VALUE},         // initializer: holds previous value
    (state, element, downstream) -> {           // integrator
        if (element > state[0]) {
            state[0] = element;
            return downstream.push(element);
        }
        return true;                            // skip non-increasing values
    }
);

List<Integer> increasing = Stream.of(1, 3, 2, 5, 4, 6)
    .gather(onlyIncreasing)
    .toList();
// Result: [1, 3, 5, 6]
```

**Rules for Gatherers:**
- Keep the `integrator` pure — avoid external state mutations inside it.
- Document whether the Gatherer is **stateless**, **stateful sequential**, or **stateful parallel-safe**.
- Prefer `Gatherers.*` built-in factory methods (`windowSliding`, `windowFixed`, `scan`, `fold`) before writing custom implementations.
- Avoid `gather()` for simple transformations that `map`, `filter`, or `flatMap` already cover.

```java
// Avoid: gather() for what filter() already does
stream.gather(Gatherer.of(
    (state, e, down) -> { if (e > 0) down.push(e); return true; }
));

// Good: use filter() directly
stream.filter(e -> e > 0)
```

### Performance Notes

- `Gatherers.windowSliding()` and `Gatherers.windowFixed()` buffer elements internally — profile memory allocation with JFR for large streams.
- Custom Gatherers with a `combiner` support parallel streams. Without a `combiner`, the stream runs sequentially even if `parallel()` is called — document this explicitly.
- Prefer sequential streams for stateful Gatherers unless the domain permits unordered processing.

---

## Scoped Values (JEP 506)

`ScopedValue` provides immutable, bounded context within a thread and its callees. In functional pipelines, it eliminates the need to thread contextual parameters (request ID, user identity, locale) through every method signature without breaking referential transparency.

### `ScopedValue` vs `ThreadLocal`

| Concern | `ThreadLocal` | `ScopedValue` |
| :--- | :--- | :--- |
| Mutability | Mutable — `.set()` / `.remove()` | Immutable — bound once per `where()` scope |
| Lifetime | Thread lifetime (risk of leaks) | Bounded to the `where().run()` / `where().call()` scope |
| Functional purity | Breaks referential transparency | Preserves purity — scope is explicit and bounded |
| Virtual thread safety | Risky (inherits across children) | Safe — scope is clearly delimited |

```java
// Good: Scoped value for immutable request context
public class RequestContext {
    public static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

    public void handle(User user, Runnable task) {
        ScopedValue.where(CURRENT_USER, user).run(task);
    }
}

// Any method within the scope can read the value — no parameter threading
class AuditService {
    void audit(String action) {
        var user = RequestContext.CURRENT_USER.get();
        log.info("Action '{}' by {}", action, user.id());
    }
}

// Good: Multiple values bound in the same scope
ScopedValue.where(CURRENT_USER, user)
           .where(REQUEST_ID, requestId)
           .run(() -> processRequest());

// Good: orElse for when the value may not be bound
var user = RequestContext.CURRENT_USER.orElse(User.anonymous());
```

**Rules:**
- Bind only at the outermost entry point (controller, message consumer). Inner domain logic must only read, never rebind.
- Do **not** use `ScopedValue` as mutable shared state — it is immutable by design.
- `ThreadLocal` remains valid for mutable, thread-local accumulation (formatters, parsers) — do not replace it wholesale.

---

## Flexible Constructor Bodies (JEP 513) — Domain Modeling Angle

In functional domain modeling, class-based hierarchies that extend abstract types previously required static helper methods to validate constructor arguments before `super()`. JDK 25 eliminates this friction.

```java
// JDK 21: static helper required
public class NonEmptyList<T> extends AbstractList<T> {
    public NonEmptyList(List<T> items) {
        super(validate(items));
    }
    private static <T> List<T> validate(List<T> items) {
        if (items.isEmpty()) throw new IllegalArgumentException("List must not be empty");
        return items;
    }
}

// JDK 25: validation directly before super()
public class NonEmptyList<T> extends AbstractList<T> {
    public NonEmptyList(List<T> items) {
        if (items.isEmpty()) throw new IllegalArgumentException("List must not be empty");
        super(items);
    }
}
```

- Records with compact constructors remain the preferred idiom for value objects.
- Use this feature only in class hierarchies where extension is genuinely necessary.
- For the general mechanics, see `java-core-jdk25_instructions.md`.

---

## Additional Resources

- [JEP 456 — Unnamed Variables & Patterns](https://openjdk.org/jeps/456)
- [JEP 485 — Stream Gatherers](https://openjdk.org/jeps/485)
- [JEP 506 — Scoped Values](https://openjdk.org/jeps/506)
- [JEP 513 — Flexible Constructor Bodies](https://openjdk.org/jeps/513)
- [Gatherers Javadoc](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/stream/Gatherers.html)
