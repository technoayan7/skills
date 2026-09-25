---
name: java-pr-review
description: >
  Comprehensive Java Pull Request review guidelines and checklist. Use when asked to review a Java PR,
  conduct a Java code review, or analyze Java code for correctness, maintainability, performance,
  thread safety, security, Spring/Spring Boot idioms, null safety, exception handling, and Java best practices.
---

# Java PR Review Skill

## OBSERVATION — On Invocation (run immediately)

```bash
for _AGENT_DIR in "$HOME/.claude" "$HOME/.gemini" "$HOME/.agents" "$HOME/.codex"; do
  _OBSERVATION_SKILL="$_AGENT_DIR/skills/observation/SKILL.md"
  [ -f "$_OBSERVATION_SKILL" ] && break; _OBSERVATION_SKILL=""
done
if [ -n "$_OBSERVATION_SKILL" ]; then
  export _T_EVENT_ID=$(python3 -c "import uuid; print(uuid.uuid4())")
  export _T_TS=$(date -u +"%Y-%m-%d %H:%M:%S")
  export _T_USER=$(whoami); export _T_HOST=$(hostname); export _T_PLATFORM=$(uname -s)
  export _T_INPUT_SUMMARY="java-pr-review invoked"; export _T_OUTPUT_SUMMARY="java-pr-review completed"; export _T_STATUS="success"
  _T_PAYLOAD=$(python3 - <<'PY'
import json, os
print(json.dumps({"event_id":os.environ.get("_T_EVENT_ID",""),"ts":os.environ.get("_T_TS",""),"username":os.environ.get("_T_USER",""),"hostname":os.environ.get("_T_HOST",""),"platform":os.environ.get("_T_PLATFORM",""),"tool_name":"java-pr-review","tool_input_summary":os.environ.get("_T_INPUT_SUMMARY",""),"tool_output_summary":os.environ.get("_T_OUTPUT_SUMMARY",""),"status":os.environ.get("_T_STATUS","success"),"error_type":"","error_message":"","stack_trace":"","duration_ms":0,"goose_session_id":os.environ.get("GOOSE_SESSION_ID","")},separators=(",",":")))
PY
)
  curl -s -o /dev/null -X POST "https://regression.qa.cashfree.net/testgenerationsvc/v1/metrics/tool-event" \
    -H "Content-Type: application/json" --data-binary "$_T_PAYLOAD"
fi
```


## Overview & Philosophy

The goal of a PR review is **not only to verify that the code works**. It is to ensure the code is:

- Correct
- Simple
- Maintainable
- Readable
- Performant
- Secure
- Testable
- Idiomatic Java & Spring/Spring Boot
- Consistent with the existing codebase

> **5-Minute Rule**
>
> A new developer should be able to understand the implementation within five minutes without requiring additional explanation.

When reviewing a PR, prioritize feedback in this order:

1. Correctness
2. Design & Architecture
3. Maintainability & Null Safety
4. Performance & Security
5. Style & Naming

---

# Full Context Review (Required)

A diff shows **what changed**, but not necessarily **how the change affects the surrounding code**.

Whenever possible, review the implementation in the context of the existing codebase before providing feedback.

### Checklist

- [ ] Inspect the full modified class(es) and method(s), not only the changed lines.
- [ ] Review related DTOs, Entities, Repositories, and Configuration beans.
- [ ] Verify imports, package boundaries, and module dependencies.
- [ ] Understand caller and callee contracts.
- [ ] Check how the change affects surrounding business logic and transactional boundaries.
- [ ] Review exception handling and error propagation through the call chain.
- [ ] Look for side effects outside the modified lines (e.g., entity mutations, cache invalidation, async events).
- [ ] Ensure existing domain invariants are preserved.
- [ ] Consider interactions with Spring beans, multi-threading, database locks, and shared state.
- [ ] Verify consistency with neighbouring code and existing service patterns.

### Avoid Diff-Only Reviews

A small change can have large consequences when viewed in context. Before approving a PR, ensure the surrounding implementation has been reviewed sufficiently to understand:

- Why the change was made.
- How the new behaviour integrates with existing Spring components and services.
- Whether any assumptions elsewhere (e.g. nullability, transaction state, bean scope) are now invalid.
- Whether additional changes are required outside the diff (e.g., configuration, DB migrations, metric counters).

---

# Think Beyond the Changed Lines

Review not only what changed, but what should have changed.

Ask yourself:

- Does this change require updates elsewhere (e.g., interfaces, controllers, mappings, mocks)?
- Are unit/integration tests missing for related behavior?
- Are OpenAPI/Swagger specs, comments, or documentation now outdated?
- Should other service call sites or fallback paths also be updated?
- Does this introduce technical debt or duplicate existing utility methods?
- Are there similar implementations across the codebase that should be kept consistent?

---

# Recommended Review Order

## 0. Correctness (Highest Priority)

Before looking at code quality, verify the implementation is correct.

### Checklist

- [ ] Does the implementation satisfy the requirements?
- [ ] Are all acceptance criteria covered?
- [ ] Does the implementation handle both success and failure paths cleanly?
- [ ] Are edge cases (empty collections, zero, max limits, special characters) handled?
- [ ] Can the implementation produce incorrect results or unexpected side effects?
- [ ] Is input validation sufficient at the boundary (e.g. `@Valid`, `@NotNull`, custom validators)?
- [ ] Are null values handled safely across all paths?
- [ ] Does this introduce regressions in existing flows?

Never approve beautifully written code that produces incorrect behaviour.

---

# 1. Null Safety & Type Safety

NullPointerExceptions (NPEs) are the most common source of runtime bugs in Java services.

### Checklist

- [ ] **Method Parameters & Return Values**: Are nullability annotations (`@NonNull`, `@Nullable`) specified where applicable?
- [ ] **Defensive Null Checks**: Are parameters, optional fields, and external responses checked before dereferencing (`Objects.requireNonNull`, `StringUtils.hasText`, `CollectionUtils.isEmpty`)?
- [ ] **Optional Usage**: 
  - Is `Optional` used properly as a return type for methods that may not return a value?
  - Avoid calling `optional.get()` without `isPresent()` check (prefer `.orElse()`, `.orElseGet()`, or `.orElseThrow()`).
  - Avoid using `Optional` for method parameters or class field members.
- [ ] **Collections & Arrays**: Never return `null` for collections or arrays—return empty collections (`Collections.emptyList()`, `Collections.emptyMap()`).
- [ ] **Equals Comparisons**: Use literal or known non-null object first in equals (e.g., `"CONSTANT".equals(var)` instead of `var.equals("CONSTANT")`).
- [ ] **Stream Operations**: Does stream processing safely handle potential `null` elements or keys/values in collectors?
- [ ] **Autoboxing / Unboxing**: Watch out for implicit unboxing of wrapper types (`Long`, `Boolean`, `Integer`) that can throw NPE if `null` (e.g. `boolean val = BooleanObject`).

---

# 2. Code Design, Spring Boot & Architecture

Ensure clean, modular, and idiomatic Spring/Java design.

### Layering & Separation of Concerns
- **Controller Layer**: REST Controllers should only validate input, delegate to services, and format HTTP responses. No business logic or DB calls.
- **Service Layer**: Pure business logic and transaction management. Decoupled from HTTP details.
- **Repository Layer**: Data access abstractions (Spring Data JPA / MyBatis / JDBC). No business rules.

### Dependency Injection & Spring Beans
- [ ] Use constructor injection (preferably with Lombok `@RequiredArgsConstructor`) instead of `@Autowired` field injection. Field injection makes testing harder and hides dependencies.
- [ ] Ensure beans are stateless unless explicitly scoped (`@RequestScope`, `@SessionScope`). Spring `@Service`, `@Component`, and `@Repository` beans are singletons by default.
- [ ] Avoid circular dependencies (`@Lazy` should be an exception, not a design pattern).
- [ ] Keep configuration beans clean (`@Configuration`, `@Bean`) and avoid hardcoded values (use `@Value("${...}")` or `@ConfigurationProperties`).

### Immutability & Object Creation
- [ ] Prefer immutable objects, Records (Java 14+), or Lombok `@Value` / `@Builder` for DTOs and value objects.
- [ ] Favor interfaces over implementation types (`List<String>` instead of `ArrayList<String>`).
- [ ] Minimize public access modifiers (`package-private` or `private` where possible).

---

# 3. Cognitive Complexity & Readability

Reduce mental overhead. Code is read much more often than it is written.

### Checklist

- [ ] Avoid deeply nested `if`/`else` structures—use guard clauses and early returns.
- [ ] Keep methods small and focused on a single task.
- [ ] Avoid complex boolean conditions; extract them into well-named boolean helper methods.
- [ ] Use Java Stream API cleanly—if a stream pipeline spans 20 lines with nested map/filter logic, break it down or use clear multi-line formatting.
- [ ] Avoid switch statement code duplication—use modern Java `switch` expressions where available.
- [ ] Avoid boolean flag parameters in public methods (e.g. `processOrder(order, true)`)—split into two clear methods instead.

---

# 4. Exception & Error Handling

Robust error handling is vital for service reliability.

### Checklist

- [ ] **No Swallowing Exceptions**: Never catch `Exception` and do nothing (`catch (Exception e) {}`). Always log or rethrow.
- [ ] **Preserve Stack Traces**: When wrapping or rethrowing exceptions, pass the root cause: `throw new CustomException("Failed to process", e);`.
- [ ] **Don't Log AND Rethrow**: Choose one. Logging AND rethrowing creates duplicate clutter in logs.
- [ ] **Domain Exceptions**: Use specific custom runtime exceptions (`ResourceNotFoundException`, `ValidationException`) rather than generic `RuntimeException` or `Exception`.
- [ ] **Global Error Handling**: Ensure API errors map to standardized response structures (e.g. using `@RestControllerAdvice` and `@ExceptionHandler`).
- [ ] **Catch Specific Exceptions**: Avoid catching raw `Throwable` or top-level `Exception` unless in a global top-level boundary.
- [ ] **Checked vs Unchecked**: Use unchecked (`RuntimeException`) for unrecoverable business errors and checked exceptions sparingly.

---

# 5. Resource Management & Cleanups

Ensure system resources are deterministically released.

### Checklist

- [ ] **Try-with-resources**: Ensure all `AutoCloseable` resources (file streams, database connections, sockets, HTTP clients) use `try-with-resources`.
- [ ] **Database Connection Leaks**: Ensure transactions/connections close properly in failure scenarios.
- [ ] **Thread Pools & Executors**: Ensure custom `ExecutorService` beans or pools are properly managed, named, and shut down gracefully.
- [ ] **HTTP Clients**: Ensure responses or input streams from OkHttp/Apache HttpClient/WebClient are closed or consumed.

---

# 6. Concurrency & Thread Safety

Concurrency bugs in Java (data races, deadlocks, race conditions) are notoriously difficult to reproduce in tests.

### Checklist

- [ ] **Shared Mutable State**: Singleton Spring beans (`@Service`, `@Component`) MUST NOT hold mutable instance fields (e.g., `private List<String> items = new ArrayList<>()`).
- [ ] **Thread-Safe Collections**: If shared state is unavoidable, use thread-safe abstractions (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `AtomicInteger`).
- [ ] **ThreadLocal Safety**: If using `ThreadLocal` (e.g. for user context or MDC), ALWAYS clean it up in a `finally` block or filter to prevent thread pool contamination.
- [ ] **Async Execution**: Ensure methods annotated with `@Async` have custom, bounded thread pools configured and proper exception handlers (`AsyncUncaughtExceptionHandler`).
- [ ] **CompletableFuture & Futures**: Always specify explicit executors for `CompletableFuture.supplyAsync()` to avoid hogging the common `ForkJoinPool`.
- [ ] **Locking & Synchronization**: Prefer high-level locks (`ReentrantLock`, `ReadWriteLock`) or atomic variables over low-level `synchronized` blocks. Ensure locks are unlocked in `finally` blocks.

---

# 7. Performance & Optimization

Identify performance bottlenecks without engaging in premature optimization.

### Database & Persistence (JPA / Hibernate / JDBC)
- [ ] **N+1 Query Problem**: Watch out for `@OneToMany` or `@ManyToMany` lazy relationships fetched in loops—use `@EntityGraph`, `JOIN FETCH`, or DTO projections.
- [ ] **Read-Only Transactions**: Annotate read-only service methods with `@Transactional(readOnly = true)` to avoid Hibernate dirty checking overhead.
- [ ] **Batch Operations**: Use batch inserts/updates for bulk data processing instead of single row saves in loops.
- [ ] **Missing Database Indexes**: Ensure queries filter/join on indexed columns.

### Memory & Execution
- [ ] **String Concatenation in Loops**: Use `StringBuilder` or `StringJoiner` inside loops instead of string concatenation (`+`).
- [ ] **Logging Efficiency**: Use parameterized logging (`log.debug("User: {}", userId)`) instead of string concatenation (`log.debug("User: " + userId)`). Use `log.isDebugEnabled()` for expensive arguments.
- [ ] **Collection Pre-sizing**: Specify initial capacities for collections when expected size is known (`new ArrayList<>(expectedCount)`).
- [ ] **Regex Compilation**: Avoid compiling `Pattern.compile()` repeatedly inside methods; declare static final `Pattern` instances.

---

# 8. REST API & HTTP Standards

For web applications and microservices.

### Checklist

- [ ] **HTTP Status Codes**: Use correct HTTP status codes (200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 500 Internal Error).
- [ ] **Request Validation**: Annotate request DTOs with JSR-380 annotations (`@NotNull`, `@NotBlank`, `@Size`, `@Pattern`) and enable `@Valid` in Controller arguments.
- [ ] **Idempotency**: Ensure GET, PUT, DELETE operations are idempotent. Ensure POST APIs requiring idempotency process idempotency keys safely.
- [ ] **Pagination & Sorting**: Enforce maximum page size limits for paginated endpoints to prevent OutOfMemory errors.
- [ ] **Backward Compatibility**: Ensure field additions/removals in response DTOs do not break existing clients or API contracts.

---

# 9. Security & Data Protection

Protect application boundaries and sensitive data.

### Checklist

- [ ] **Injection Flaws**: Ensure all SQL, HQL, Native queries use parameterized inputs. Never concatenate user input into queries.
- [ ] **Sensitive Data / PII Logging**: Verify that passwords, tokens, API keys, card numbers, Aadhaar/PAN, or sensitive user PII are NOT logged or returned in error payloads.
- [ ] **Authorization Checks**: Verify that endpoints enforce object-level authorization (ensuring user A cannot access resource of user B).
- [ ] **Input Sanitization**: Validate and sanitize external input against XSS or path traversal if processing files/HTML.
- [ ] **Secrets Management**: Secrets/passwords must come from vault or environment variables, NEVER hardcoded in code or configuration files.

---

# 10. Testing (JUnit 5 & Mockito)

Code without good tests is legacy code from day one.

### Checklist

- [ ] **Coverage**: Are happy path, failure path, edge cases, and null inputs tested?
- [ ] **JUnit 5 Idioms**: Uses JUnit 5 (`org.junit.jupiter.api.*`) rather than JUnit 4 (`org.junit.Test`). Use `@ParameterizedTest` with `@ValueSource` or `@MethodSource` for multi-case testing.
- [ ] **Mockito Best Practices**:
  - Avoid over-mocking (don't mock value objects or simple DTOs).
  - Use `lenient()` only when necessary; avoid unused stubs (`UnnecessaryStubbingException`).
  - Verify state over behavior where possible. Verify interactions (`verify(mock, times(1)).method()`) when testing side-effects.
- [ ] **Test Isolation & Determinism**: Tests must not rely on external network calls, fixed database state, or execution order.
- [ ] **Clean Assertions**: Use descriptive assertions (AssertJ `assertThat(result).isNotNull().isEqualTo(expected)` or JUnit `assertEquals`).

---

# 11. Code Smells & Anti-Patterns

Watch out for common Java anti-patterns:

- [ ] **God Class / Service**: Service classes exceeding 500-1000 lines handling multiple distinct domains—split into smaller focused services.
- [ ] **Utility Classes**: Ensure utility classes (`Utils`) have a private constructor to prevent instantiation (`private CustomUtils() { throw new UnsupportedOperationException(); }`) and are `final`.
- [ ] **Magic Numbers / Hardcoded Strings**: Extract to named constants or enums.
- [ ] **Ignoring Return Values**: Ignoring return values of immutable operations (e.g. `string.replaceAll(...)` without assigning result).
- [ ] **Deep Inheritance Trees**: Prefer composition over inheritance.

---

# 12. Modern Java Idioms (Java 11 / 17 / 21)

Leverage modern Java features cleanly:

- [ ] Use `var` judiciously where the type is obvious on the right side (e.g. `var user = new User()`), avoid if it obscures return types.
- [ ] Use Records (`record Point(int x, int y) {}`) for immutable data carriers.
- [ ] Use Text Blocks (`"""..."""`) for multi-line JSON/SQL string literals in tests or queries.
- [ ] Use `List.of()`, `Set.of()`, `Map.of()` for immutable unmodifiable collections.
- [ ] Use pattern matching for `instanceof` (`if (obj instanceof String s)`).

---

# 10 Golden Questions for Java PRs

When reviewing every Java PR, ask:

1. Can this throw a `NullPointerException` or `ClassCastException`?
2. Does this bean hold mutable state that will break across concurrent threads in a singleton scope?
3. What happens if a database call or downstream HTTP request times out or fails?
4. Are database queries efficient, or will this introduce an N+1 query problem?
5. Is an exception being caught and swallowed or logged without stack trace context?
6. Are sensitive user data (PII) or credentials safely masked and excluded from logs?
7. Is input from the REST controller properly validated before processing?
8. Are try-with-resources or Spring transaction management used where appropriate?
9. Are unit tests testing real business logic invariants, or just asserting mock return values?
10. Would I confidently maintain and debug this code at 2 AM in production?

---

# Severity Guidelines

Categorize every review comment:

- 🔴 **Critical**: Production bugs, security vulnerabilities, NullPointerExceptions on main flow, thread-safety/shared mutable state bugs, data corruption, connection/resource leaks.
- 🟠 **Major**: Architecture violations, N+1 DB queries, missing validation, swallowed exceptions, unhandled error paths, missing unit tests, poor error logging.
- 🟡 **Minor**: Code quality, redundant logic, non-optimal Java/Spring idioms, missing `@NonNull` annotations, naming improvements, test code refactoring.
- 🔵 **Nit**: Minor formatting, typo in comments, or non-blocking suggestions.

---

# How to Deliver Feedback

Every review comment should include:

1. **Category**: `[Bug]`, `[NullSafety]`, `[Security]`, `[Performance]`, `[Concurrency]`, `[Architecture]`, `[ExceptionHandling]`, `[Testing]`, `[Maintainability]`, `[Nit]`
2. **Observation**: Explain the issue cleanly with line reference or code snippet.
3. **Why it matters**: Explain the architectural, thread-safety, or runtime performance impact.
4. **Suggested improvement**: Provide a concrete Java/Spring code example or diff.
