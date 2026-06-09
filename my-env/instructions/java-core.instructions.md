---
description: 'Guidelines for building high-quality Java applications targeting JDK 21 LTS, covering modern language features, architecture, testing, and static analysis'
applyTo: '**/*.java'
---

# Java Development Guidelines (JDK 21 LTS)

Write clean, maintainable, and modern Java code targeting **JDK 21 LTS**.

> **Scope & Safety**
> - Always prioritize what is explicitly defined in the project (build files, toolchain, parent BOM, and repository conventions) before applying generic guidance.
> - Target: JDK 21 LTS stable features only. Do **not** generate code using preview or JDK 22+ features unless the build files explicitly target that version and the user opts in.
> - Never include real secrets, API keys, or passwords in examples. Use placeholders: `{{API_KEY}}`, `{{DB_PASSWORD}}`, `user@example.com`.
> - All generated code must pass **SonarLint** and **Checkstyle** analysis.
>
> **Conflict Resolution Order**
> 1. Repository build configuration (Maven/Gradle Java version, dependencies)
> 2. Static analysis rules (SonarQube, Checkstyle)
> 3. These guidelines
> 4. Google Java Style Guide
>
> **See also**: `java-functional-v2_instructions.md` — immutability, pure functions, `Result<T,E>`, Functor/Monad patterns, higher-order functions, and functional architecture.

---

## General Instructions

- **Immutability First**: Use `record`, `final` fields, and `List.of()` / `Map.of()`. Design classes to be immutable by default (*Effective Java*, Item 17).
- **Declarative over Imperative**: Use Streams and Lambdas for collection processing. Pipelines must be side-effect free.
- **Modern Syntax**: Use Switch Expressions, Pattern Matching, Text Blocks, and Records to reduce boilerplate.
- **Null Safety**: Never return `null` for collections (return `List.of()`). Use `Optional<T>` only as a return type, never in fields or parameters. Use `Objects.requireNonNull()` for mandatory arguments.
- **Composition over Inheritance**: Use `sealed` interfaces for strict hierarchies and favor composition (*Effective Java*, Item 18).
- **Logging Boundary**: Log only at the application layer (services, controllers, adapters). Never log inside pure domain/business logic.
- **Static Analysis**: Avoid high cognitive complexity. Always use try-with-resources for `AutoCloseable`. When a Sonar rule conflicts with style, the Sonar rule takes precedence to keep CI green.
- **Comments and Javadoc**: Prefer self-explanatory code. Only require extensive Javadoc when explicitly requested or required by project conventions.

---

## JDK 21 Stable Feature Baseline

Use these features freely — they are **not** preview:

| Feature | JEP | Stable Since |
| :--- | :--- | :--- |
| Records | JEP 395 | JDK 16 |
| Sealed Classes | JEP 409 | JDK 17 |
| Pattern Matching for `instanceof` | JEP 394 | JDK 16 |
| Pattern Matching for `switch` | JEP 441 | **JDK 21** |
| Record Patterns | JEP 440 | **JDK 21** |
| Text Blocks | JEP 378 | JDK 15 |
| `var` (local type inference) | JEP 286 | JDK 10 |
| Sequenced Collections | JEP 431 | **JDK 21** |
| `Stream.toList()` | — | JDK 16 |

> **Not available in JDK 21:** Unnamed Variables `_` (stable JDK 22), `Stream.gather()` (stable JDK 24), Primitive Patterns in switch (JDK 25). Do not generate these for JDK 21 targets.
>
> **Preview in JDK 21 — do not use in production:** Structured Concurrency (JEP 453), Scoped Values (JEP 446), Unnamed Classes (JEP 445).

---

## Best Practices

### 1. Data Modeling with Records

Use `record` for transparent, immutable data carriers. Use compact constructors for validation.

```java
// Good: Immutable value object with invariant validation
public record Money(BigDecimal amount, String currency) {
    public Money {
        Objects.requireNonNull(amount, "amount");
        Objects.requireNonNull(currency, "currency");
        if (amount.signum() < 0) throw new IllegalArgumentException("Amount cannot be negative");
    }
}
```

- Do not use `get` prefix on record accessors (`order.id()`, not `order.getId()`).
- Use Record Patterns (JDK 21) for clean deconstruction in `switch` and `instanceof`.

```java
// Good: Record deconstruction in switch (JDK 21)
return switch (payment) {
    case CreditCard(var num, var expiry) when isExpired(expiry) -> throw new PaymentException("Expired card");
    case CreditCard(var num, var expiry)                        -> processCredit(num);
    case DigitalWallet(var provider)                            -> processDigital(provider);
};
```

### 2. Sealed Hierarchies for Domain Modeling

Use `sealed` interfaces to model Algebraic Data Types (ADTs) where the set of subtypes is fixed.

```java
// Good: Closed type hierarchy with exhaustive handling
public sealed interface OrderResult permits OrderResult.Success, OrderResult.Failure {
    record Success(Order order)                    implements OrderResult {}
    record Failure(String reason, ErrorCode code)  implements OrderResult {}
}

// Exhaustive switch — no default branch needed
var message = switch (result) {
    case OrderResult.Success(var order)        -> "Loaded: " + order.id();
    case OrderResult.Failure(var reason, var code) -> "Error [%s]: %s".formatted(code, reason);
};
```

- Never add a `default` clause when switching over a sealed hierarchy — it hides unhandled variants when new subtypes are added.

### 3. Sequenced Collections (JDK 21)

Use `SequencedCollection` APIs when element order matters.

- Use `list.getFirst()` / `list.getLast()` — avoid `list.get(0)` / `list.get(list.size() - 1)`.
- Use `list.reversed()` instead of manual reversal.

### 4. Streams and Functional Pipelines

- Use `Stream.toList()` (JDK 16+) for unmodifiable results — not `Collectors.toList()`.
- Use `IntStream`, `LongStream`, `DoubleStream` to avoid boxing overhead (*Effective Java*, Item 45).
- Keep lambdas short. Extract multi-line lambdas into named private methods.

```java
// Good: Clear declarative pipeline
List<String> activeNames = users.stream()
    .filter(User::isActive)
    .map(User::name)
    .toList();

// Avoid: Over-nested functional composition
users.stream().collect(groupingBy(User::getDept,
    mapping(User::getName, filtering(n -> n.length() > 5, toList()))));
```

For advanced pipeline composition, named predicates, and `Function`/`Predicate` combinators, see `java-functional-v2_instructions.md`.

### 5. Enum Patterns

Use Enums over `int` or `String` constants. Prefer behavioral enums with functional fields over large `switch` blocks (*Effective Java*, Item 34).

```java
// Good: Strategy Enum
public enum Operation {
    PLUS ((x, y) -> x + y),
    MINUS((x, y) -> x - y);

    private final BinaryOperator<Double> operator;
    Operation(BinaryOperator<Double> op) { this.operator = op; }
    public double apply(double x, double y) { return operator.apply(x, y); }
}
```

---

## Code Standards

### Naming Conventions

Follow the [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html).

| Identifier | Style | Example |
| :--- | :--- | :--- |
| Classes / Records / Interfaces / Sealed types | `UpperCamelCase` | `OrderService`, `OutOfStock` |
| Methods | `lowerCamelCase` verb | `calculateTotal`, `isAvailable` |
| Record components | `lowerCamelCase` noun (no `get` prefix) | `id`, `createdAt` |
| Constants (`static final`) | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Local variables / `var` | `lowerCamelCase`, descriptive | Avoid `data`, `list1` |
| Boolean method / predicate | question form | `isActive()`, `hasDiscount()` |
| Test method | `lowerCamelCase` | `methodName_StateUnderTest_ExpectedBehavior` |

- Use `var` only when the inferred type is **immediately obvious** from the right-hand side.
- For `Function` and `Predicate` naming conventions, see `java-functional-v2_instructions.md`.

### Exception Handling

- Prefer unchecked exceptions for programming errors (*Effective Java*, Item 70).
- Use checked exceptions only for truly recoverable business conditions.
- Never catch `Throwable` or bare `Exception`. Catch the most specific subtype.
- Catch blocks must log, rethrow, or handle — never silently swallow.
- Always chain the original cause to preserve the full stack trace.
- Restore the interrupt flag when catching `InterruptedException`.

```java
// Good: Try-with-resources + specific exception + cause chaining
try (var lines = Files.lines(path)) {
    return lines.map(this::parse).toList();
} catch (IOException e) {
    throw new UncheckedIOException("Failed to read config: " + path, e);
}

// Good: InterruptedException always restores the interrupt flag
try {
    return httpClient.send(request, bodyHandler);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // mandatory — restore the flag
    throw new IntegrationException("Request interrupted: " + request.uri(), e);
}
```

Define a sealed exception hierarchy per domain so adapters can discriminate precisely:

```java
// Good: Sealed exception hierarchy mirrors domain structure
sealed class DomainException extends RuntimeException
    permits UserNotFoundException, OrderProcessingException {
    DomainException(String message)                  { super(message); }
    DomainException(String message, Throwable cause) { super(message, cause); }
}
```

For modeling **expected business failures as values** (not exceptions), see `Result<T,E>` in `java-functional-v2_instructions.md`.

### Documentation (Javadoc)

- Document the **contract and invariants**, not the implementation (*Effective Java*, Item 56).
- Use `@param`, `@return`, and `@throws` on all public API members when Javadoc is required.
- Use `@Override` on every method that overrides or implements.
- Use `@Nullable` / `@NotNull` (JSR-305) to assist static analysis.
- Mark deprecated APIs with `@Deprecated(since="x.y", forRemoval=true)`.
- Use `@implSpec` to declare that a method is a pure function (no I/O, no mutation).

### Collection Standards

- Declare variables as `List`, `Set`, `Map` — never `ArrayList` or `HashMap` (*Effective Java*, Item 64).
- Use `List.of()`, `Set.of()`, `Map.of()` for read-only collections.
- Store defensive copies in constructors: `this.items = List.copyOf(items)`.
- Never expose internal mutable collections through accessor methods.

### Resource Management

Use try-with-resources for **all** `AutoCloseable` types (files, sockets, JDBC connections).

```java
// Good
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement(SQL)) {
    // use conn and stmt
}

// Avoid: close() not reached if an exception is thrown before it — resource leak (Sonar S2095)
Connection conn = dataSource.getConnection();
conn.close();
```

### Annotations

- Use `@Override` on every method that overrides or implements a supertype member.
- Use `@Nullable` / `@NotNull` (JSR-305) to signal nullability to static analysis.
- Never suppress warnings without a comment explaining the specific, justified reason.

---

## Defensive Programming

Treat every input as potentially incorrect until proven otherwise. Make invalid states unrepresentable and fail loudly at the earliest possible point — never silently.

### 1. Input Validation

Validate all public API arguments immediately at the entry point.

- Use `Objects.requireNonNull(value, "descriptive message")` for mandatory references.
- Use guard clauses with `IllegalArgumentException` for domain invariants.
- Validate at the **boundary** — don't propagate unvalidated data into domain logic.

```java
// Good: Validation at the entry point — invalid state never enters the domain
public OrderService(InventoryService inventory, AuditLog auditLog) {
    this.inventory = Objects.requireNonNull(inventory, "inventory must not be null");
    this.auditLog  = Objects.requireNonNull(auditLog,  "auditLog must not be null");
}

// Avoid: Validation absent — NullPointerException surfaces deep in the call stack
Result<Order, OrderError> place(Cart cart, Customer customer) {
    return inventory.check(cart.items()) // NPE here if cart is null
        .flatMap(this::createOrder);
}
```

### 2. Invariant Enforcement

Use compact constructors in Records to encode invariants at construction time. A constructed object is always valid.

```java
// Good: Invariants encoded in the type — Email is always valid after construction
record Email(String value) {
    Email {
        Objects.requireNonNull(value, "email must not be null");
        if (!value.contains("@"))
            throw new IllegalArgumentException("Invalid email: " + value);
        value = value.strip().toLowerCase();
    }
}

// Avoid: Invariant enforced externally — invalid state is constructable and spreadable
record Email(String value) {}
```

### 3. Defensive Copies

When accepting or returning mutable objects, always copy them.

```java
// Good: Defensive copy at construction — caller mutations have no effect
record Department(String name, List<Employee> employees) {
    Department {
        Objects.requireNonNull(employees, "employees must not be null");
        employees = List.copyOf(employees);
    }
}

// Good: Defensive copy on return
public final class Schedule {
    private final List<LocalDate> holidays = new ArrayList<>();
    public List<LocalDate> holidays() { return List.copyOf(holidays); }
}

// Avoid: Record is shallowly immutable — caller retains a reference to the internal list
record Department(String name, List<Employee> employees) {}
```

### 4. Exception Handling as a Defensive Layer

- Catch the **most specific** subtype — never `Exception` or `Throwable`.
- Always chain the original cause to preserve the full stack trace.
- Never return `null`, an empty string, or zero as a silent fallback from a catch block.

---

## Common Bug Patterns

### 1. Shallow Immutability in Records

Records with mutable components are not truly immutable without defensive copies.

```java
// Avoid: Caller can mutate the internal list
public record Order(List<String> items) {}

// Good: Defensive copy in compact constructor
public record Order(List<String> items) {
    public Order { items = List.copyOf(items); }
}
```

### 2. Non-Exhaustive Pattern Matching on Sealed Types

Use `switch` expressions, not `if/instanceof` chains, for sealed hierarchies.

```java
// Avoid: Silent failure if a new Shape subtype is added
if (shape instanceof Circle c) { ... }
else if (shape instanceof Square s) { ... }

// Good: Compiler-enforced exhaustiveness
return switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Square s -> s.side() * s.side();
};
```

### 3. Side Effects in Stream Pipelines

Never modify external state inside `.map()`, `.filter()`, or `.flatMap()` (*Effective Java*, Item 48).

```java
// Avoid: Side effect makes the stream thread-unsafe
List<String> results = new ArrayList<>();
users.stream().filter(User::isActive).forEach(u -> results.add(u.name()));

// Good: Pure pipeline
List<String> results = users.stream()
    .filter(User::isActive)
    .map(User::name)
    .toList();
```

### 4. `Optional.get()` Without Guard

```java
// Avoid
String name = findUser(id).get();

// Good
String name = findUser(id).orElseThrow(() -> new UserNotFoundException(id));
```

### 5. Identity Checks on Value-Based Classes

Never use `==` or `synchronized` on `Optional`, `LocalDate`, `Duration`, or similar value-based classes.

```java
// Avoid
if (optA == optB) { ... }        // Undefined behavior
synchronized (localDate) { ... } // May throw or deadlock

// Good
if (optA.equals(optB)) { ... }
```

### 6. Inefficient SequencedCollection Access

```java
// Avoid: O(n) on LinkedList
String first = linkedList.get(0);

// Good (JDK 21+)
String first = linkedList.getFirst();
```

---

## Common Code Smells

### 1. Primitive Obsession

Replace raw primitives used as domain concepts with self-validating Records.

```java
// Avoid: String passed anywhere, no validation guarantee
void process(String email, double amount) { ... }

// Good: Self-validating domain types
record Email(String value) {
    public Email { Objects.requireNonNull(value); /* add regex */ }
}
void process(Email email, Money amount) { ... }
```

### 2. Deep Nesting (Arrow Code)

Use guard clauses and pattern matching to flatten logic.

```java
// Avoid: Three levels of nesting
if (user != null) {
    if (user.isActive()) {
        if (user.hasRole(ADMIN)) { ... }
    }
}

// Good: Guard clauses flatten the happy path
if (user == null || !user.isActive() || !user.hasRole(ADMIN)) return;
// proceed with admin logic
```

### 3. High Cognitive Complexity

Keep methods under ~20 lines. Extract private helpers. Decompose into Records for intermediate state.

### 4. The "Return Null" Habit

- **Collections**: Return `List.of()`, `Set.of()`, or `Map.of()` (*Effective Java*, Item 54).
- **Single values**: Return `Optional<T>` to force the caller to handle absence.

### 5. Inheritance Abuse

Use `extends` only for true "is-a" relationships. Use composition + interfaces for code reuse (*Effective Java*, Item 18).

### 6. Long Parameter Lists

Keep method parameters to **≤ 4**. Group related parameters into a Record.

```java
// Avoid
void registerUser(String name, String email, String role, Locale locale,
                  boolean sendWelcome, String referralCode) { ... }

// Good
record UserRegistration(String name, String email, String role, Locale locale) {}
void registerUser(UserRegistration registration) { ... }
```

### 7. Anti-Patterns Table

| Anti-Pattern | Reason | Solution |
| :--- | :--- | :--- |
| `Optional.get()` without guard | Throws `NoSuchElementException` | Use `orElseThrow()`, `map()`, or `ifPresent()` |
| `Collectors.toList()` | Returns mutable list | Use `Stream.toList()` (JDK 16+) |
| `orElse(methodCall())` | Method always evaluated eagerly | Use `orElseGet(() -> methodCall())` |
| `default` in sealed `switch` | Hides unhandled variants on type additions | Remove `default`; let the compiler enforce |
| `System.currentTimeMillis()` in logic | Non-deterministic, untestable | Pass `Instant` or `Clock` as argument |

---

## Architecture Guidelines

### 1. Domain Modeling

Model the domain with `sealed` interfaces and `record` classes. The compiler enforces exhaustive handling.

```java
public sealed interface PaymentMethod permits CreditCard, PayPal, Crypto {}
public record CreditCard(String number, String vaultId) implements PaymentMethod {}
public record PayPal(String email)                      implements PaymentMethod {}
public record Crypto(String walletAddress, String currency) implements PaymentMethod {}
```

For pure domain core, I/O-at-the-edges architecture, and `Result<T,E>` error modeling, see `java-functional-v2_instructions.md`.

### 2. Encapsulation and Access Control

- Mark classes `final` unless designed for extension (*Effective Java*, Item 15).
- Use `sealed` to allow controlled extension within a module or package.
- Keep implementation classes package-private; expose only interfaces or records.
- Use `module-info.java` in large systems to enforce API boundaries.

### 3. Dependency Inversion and Composition

- Depend on interfaces, not implementations (*Effective Java*, Item 64).
- Inject dependencies via constructors (prefer records for stateless service components).

### 4. Service Layer Design

- Design services as **stateless** to ensure thread-safety and scalability.
- Use records as DTOs between layers (Controller → Service → Repository).

### 5. Error Handling Strategy

- **Expected business failures**: model as sealed `Result<T, E>` types — see `java-functional-v2_instructions.md`.
- **Infrastructure/programming errors**: use unchecked exceptions with proper cause chaining.

---

## Performance

### 1. Profile Before Optimizing

Use **Java Flight Recorder** (`-XX:StartFlightRecording`) and **JMH** for benchmarks. Never use `System.currentTimeMillis()` (*Effective Java*, Item 67).

### 2. Avoid Unnecessary Boxing

Use `IntStream`, `LongStream`, `DoubleStream` for numeric processing.

```java
// Avoid: Boxes 1 million integers
List<Integer> boxed = IntStream.range(0, 1_000_000).boxed().toList();

// Good: Stays primitive
int[] primitive = IntStream.range(0, 1_000_000).toArray();
```

### 3. Collection Initialization

Pre-size `ArrayList` and `HashMap` when the size is known.

```java
var map = new HashMap<String, User>(expectedSize * 4 / 3 + 1);
```

### 4. String Handling

- Use `+` for simple concatenation (compiler uses `StringConcatFactory`).
- Use `StringBuilder` only inside loops.
- Use `.formatted()` or Text Blocks for multiline strings.

### 5. Streams vs Imperative Loops

Streams introduce pipeline object overhead. For **small collections or tight inner loops**, an imperative loop has lower constant overhead. Switch to imperative only if JMH profiling shows measurable overhead on a hot path.

### 6. GC Selection

- **G1GC**: reliable default for most applications.
- **ZGC (Generational)**: `-XX:+UseZGC -XX:+ZGenerational` (stable JDK 21) for low-latency with large heaps.

---

## Testing Standards

### 1. Tooling Stack

- **Framework**: JUnit 5 — use `@ParameterizedTest`, `@Nested`, `@DisplayName`.
- **Assertions**: AssertJ — fluent assertions. Avoid `junit.jupiter.api.Assertions`.
- **Mocking**: Mockito — use `@Mock`, `@InjectMocks`, `MockitoExtension`.

### 2. Structure: AAA Pattern

- **Naming**: `methodName_Scenario_ExpectedBehavior`.
- **Isolation**: Tests must not share mutable state. Reset with `@BeforeEach`.
- **Readability**: Use `@DisplayName` to describe business requirements in plain language.

### 3. Testing Records and Sealed Classes

- Do **not** test auto-generated `equals`, `hashCode`, or `toString` on records — test compact constructor invariants instead.
- Use `switch` expressions in tests to ensure all sealed subtypes are covered.

```java
@Test
@DisplayName("Money compact constructor rejects negative amounts")
void constructor_NegativeAmount_ThrowsException() {
    assertThatThrownBy(() -> new Money(new BigDecimal("-1.00"), "USD"))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("cannot be negative");
}
```

### 4. AssertJ Fluent Assertions

- **Collections**: `containsExactly()`, `hasSize()`, `allSatisfy()`.
- **Optionals**: `isPresent()`, `contains()`, `isEmpty()`.
- **Exceptions**: `assertThatThrownBy()` with cause chain verification.

### 5. Parameterized Tests

```java
@ParameterizedTest
@CsvSource({"10, 2, 5", "20, 4, 5", "100, 10, 10"})
void divide_ValidInputs_ReturnsExpected(int a, int b, int expected) {
    assertThat(calculator.divide(a, b)).isEqualTo(expected);
}
```

### 6. Mockito — Infrastructure Boundaries Only

Use Mockito only at infrastructure boundaries (Repositories, HTTP clients). Never mock Records or domain value objects — instantiate them directly.

```java
// Avoid: Mocking domain Records — instantiate them directly
Order mockOrder = mock(Order.class);

// Good: Infrastructure boundary test
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repository;
    @InjectMocks OrderService orderService;

    @Test
    void placeOrder_WhenItemsAvailable_ReturnsConfirmedOrder() {
        var order = new Order(List.of(new Item("SKU-1", 2)));
        when(repository.save(order)).thenReturn(order.withStatus(CONFIRMED));
        assertThat(orderService.placeOrder(order).status()).isEqualTo(CONFIRMED);
    }
}
```

For pure function testing (no mocks, no setup), see `java-functional-v2_instructions.md`.

---

## Build and Verification

| Build Tool | Command |
| :--- | :--- |
| Maven | `mvn clean verify` |
| Gradle (macOS/Linux) | `./gradlew build` |
| Gradle (Windows) | `gradlew.bat build` |
| SonarScanner | `sonar-scanner -Dsonar.projectKey=<key>` |

- Maven: `<java.version>21</java.version>`
- Gradle: `sourceCompatibility = JavaVersion.VERSION_21`
- A green build with failing static analysis is **not acceptable** for merge.

---

## Additional Resources

- [JDK 21 Release Notes](https://openjdk.org/projects/jdk/21/)
- [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
- [JEP 409 — Sealed Classes](https://openjdk.org/jeps/409)
- [JEP 395 — Records](https://openjdk.org/jeps/395)
- [JEP 431 — Sequenced Collections](https://openjdk.org/jeps/431)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [AssertJ Documentation](https://assertj.github.io/doc/)
- [SonarQube Java Rules](https://rules.sonarsource.com/java/)
- *Effective Java* — Joshua Bloch
