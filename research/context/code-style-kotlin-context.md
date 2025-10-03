# Code Style context (Kotlin)

Below is a ready-to-use **AI agent prompt context** you can feed into an AI assistant (or embed in an agent) so it can reliably generate a production-quality Kotlin + Spring Boot backend focused on **core service design**: APIs, persistence (JdbcTemplate, SQL-first), messaging (Kafka, async events), Redis cache, and optional Spring Batch jobs. It also generates full Kotlin classes (controllers, services, configs) and tests (JUnit5 + Mockito + integration tests).

Use this as a universal template — reuseable for any domain model.

## Guidelines

### 1. Purpose (short)

Generate a complete, consistent, and testable Kotlin + Spring Boot microservice codebase (or individual files) following **SQL-first** principles, using:

* JDBC (JdbcTemplate / NamedParameterJdbcTemplate) with MS SQL
* Kafka for asynchronous domain events (produce & consume)
* Redis as a cache layer (caching SQL query results / session tokens)
* Optional Spring Batch job(s)
* Generate unit and integration tests (JUnit 5 + Mockito; integration tests use Testcontainers where appropriate)

---

### 2. High-level constraints & assumptions

* Language: **Kotlin** (use idiomatic Kotlin style for Spring Boot).
* Framework: **Spring Boot** (non-reactive, blocking).
* Database access: **JdbcTemplate / NamedParameterJdbcTemplate**; explicit SQL queries stored separately (resources/sql/\*.sql) — **no JPA**.
* Migrations: use **Flyway** (generate migration placeholders).
* Kafka: asynchronous, event-driven (publish domain events on changes). Consumers may update read models or trigger actions.
* Redis: used as cache layer via Spring Cache abstraction (annotations or manual).
* Spring Batch: include jobs only when requested; jobs should be modular and testable.
* Tests: produce **unit tests** (Mockito + JUnit5) and **integration tests** (Testcontainers for MSSQL, Kafka, Redis if possible).
* Security, infra, observability are out-of-scope except minimal logging and configurable properties.
* Follow single-responsibility + layered architecture: controller → service → repository → SQL.

---

### 3. Coding & architecture rules (agent must enforce)

1. **Layering**

    * Controller: validation, request/response mapping (DTOs).
    * Service: business logic, transactions, orchestration (annotated `@Service`).
    * Repository: direct SQL operations using `NamedParameterJdbcTemplate` (annotated `@Repository`).
    * EventPublisher: separate component to publish domain events to Kafka.
    * Cache: abstract via interfaces; use `@Cacheable`, `@CacheEvict` where appropriate.

2. **SQL-first**

    * All SQL lives in `resources/sql/<entity>.sql` or `resources/sql/<feature>/`.
    * SQL files are named and versioned; repository methods reference SQL by filename or load SQL via helper util.
    * Prefer explicit column lists (avoid `SELECT *`).

3. **Transactions**

    * `@Transactional` applied to service methods that require DB atomicity.
    * For methods that publish Kafka events after DB write, follow “transactional outbox” pattern OR publish events only after successful commit (simple approach: publish after save within same `@Transactional` method with `TransactionSynchronizationManager` callbacks, or use Kafka transactions if configured).

4. **Error handling**

    * Use domain-specific exceptions (e.g., `EntityNotFoundException`, `ValidationException`) and a global `@ControllerAdvice` to map to HTTP statuses and structured error body.
    * Log errors with context (request id if available).

5. **Naming conventions**

    * Packages: `com.example.<service>` → `controller`, `service`, `repository`, `config`, `dto`, `event`, `batch`.
    * Class names: `XController`, `XService`, `XRepository`, `XDto`, `XEvent`.

6. **Dependencies & DI**

    * Prefer constructor injection.
    * Keep beans single-purpose and small.

7. **Testing**

    * Unit tests: mock repositories and event publishers; test service logic and controllers (using `MockMvc`).
    * Repository tests: use `JdbcTest` or integration tests with Testcontainers MSSQL.
    * Integration tests: spin up MSSQL, Kafka, Redis via Testcontainers; seed SQL via Flyway or direct scripts; verify end-to-end flows including published events.

8. **SQL helpers**

    * Provide a small util to load SQL from `resources/sql/` and return as String (to keep queries externalized).

9. **DTO mapping**

    * Use manual mapping functions (or MapStruct if requested, but default: manual Kotlin extension functions).

10. **Configuration**

    * Use `application.yml` with profiles (`dev`, `test`, `prod`).
    * Externalize all sensitive values via environment variables.

---

### 4. Files & folder structure the agent should produce

```
src/main/kotlin/com/example/service/
  config/
    KafkaConfig.kt
    DataSourceConfig.kt
    RedisConfig.kt
    CacheConfig.kt
    BatchConfig.kt
  domain/
    <Entity>Service.kt
    EventPublisher.kt
  data/
    <Entity>Repository.kt
    SqlLoader.kt
  gateway/
    <Entity>Controller.kt
  model/
    batch/
      <JobName>JobConfig.kt
    dto/
      <Entity>CreateRequest.kt
      <Entity>Response.kt
    event/
      <Entity>CreatedEvent.kt
    exception/
      ApiException.kt
      NotFoundException.kt
      GlobalExceptionHandler.kt
resources/
  application.yml
  sql/
    entity/
      insert_<entity>.sql
      select_<entity>_by_id.sql
      update_<entity>.sql
      delete_<entity>.sql
  db/migration/ (flyway migrations)
test/
  unit/
  integration/
```

---

### 5. Request/Response format for using the agent

When asking the agent to generate code, provide:

* Service name (artifactId or base package).
* Entity name + fields (with SQL types) — minimal: `id: UUID`, plus other fields.
* Required endpoints and behavior (create, read, update, delete, search, pagination).
* Domain events to publish (e.g., `UserCreated` with what payload).
* Cache rules (which endpoints/results to cache, TTL).
* Any Spring Batch jobs (job name, input source, output).
* Whether to include integration tests with Testcontainers (yes/no).

**Example prompt the AI agent must follow:**

```
Generate a Kotlin Spring Boot microservice named "order-service" with base package com.example.order:
Entity: Order (id: UUID, customerId: UUID, amount: DECIMAL(10,2), status: VARCHAR(30), created_at: DATETIME)
Endpoints: POST /orders (create), GET /orders/{id}, PUT /orders/{id} (update status), GET /orders?customerId=...
Publish event: OrderCreated (id, customerId, amount, createdAt) to topic "order.events".
Cache: cache GET /orders/{id} for 60s.
Create repository with JdbcTemplate, SQL files in resources/sql/order/.
Generate unit tests for service and controller; integration tests using Testcontainers (MSSQL, Kafka, Redis).
Also create a simple Spring Batch job named OrderExportJob (reads orders, writes CSV).
```

---

### 6. Sample code templates (short excerpts the agent must follow)

**Controller skeleton**

```kotlin
@RestController
@RequestMapping("/orders")
class OrderController(
  private val orderService: OrderService
) {

  @PostMapping
  fun create(@RequestBody req: CreateOrderRequest): ResponseEntity<OrderResponse> {
    val created = orderService.create(req)
    return ResponseEntity.status(HttpStatus.CREATED).body(created)
  }

  @GetMapping("/{id}")
  fun getById(@PathVariable id: UUID): ResponseEntity<OrderResponse> {
    val dto = orderService.getById(id)
    return ResponseEntity.ok(dto)
  }
}
```

**Service skeleton (transactional)**

```kotlin
@Service
class OrderService(
  private val orderRepository: OrderRepository,
  private val eventPublisher: EventPublisher,
  private val transactionTemplate: TransactionTemplate // optional
) {

  @Transactional
  fun create(req: CreateOrderRequest): OrderResponse {
    val order = orderRepository.insert(req.toEntity())
    // After successful insert, publish domain event
    eventPublisher.publish(OrderCreatedEvent(order.id, order.customerId, order.amount, order.createdAt))
    return order.toResponse()
  }
}
```

**Repository skeleton (JdbcTemplate, SQL loader)**

```kotlin
@Repository
class OrderRepository(
  private val jdbc: NamedParameterJdbcTemplate,
  private val sqlLoader: SqlLoader
) {
  fun insert(order: Order): Order {
    val sql = sqlLoader.load("order/insert_order.sql")
    val params = MapSqlParameterSource().addValue("id", order.id).addValue("customer_id", order.customerId)...
    jdbc.update(sql, params)
    return order
  }

  fun findById(id: UUID): Order? {
    val sql = sqlLoader.load("order/select_order_by_id.sql")
    return jdbc.query(sql, mapOf("id" to id)) { rs, _ ->
        // map row to Order
    }.firstOrNull()
  }
}
```

**SQL loader utility**

```kotlin
@Component
class SqlLoader(
  private val resourceLoader: ResourceLoader
) {
  fun load(path: String): String { /* load from classpath: /sql/$path */ }
}
```

**Event publisher**

```kotlin
@Component
class KafkaEventPublisher(
  private val kafkaTemplate: KafkaTemplate<String, Any>
) : EventPublisher {
  override fun publish(event: Any, topic: String) {
    kafkaTemplate.send(topic, event)
  }
}
```

**Cache usage**

```kotlin
@Cacheable(value = ["orders"], key = "#id", unless = "#result == null")
fun getById(id: UUID): OrderResponse { ... }

@CacheEvict(value = ["orders"], key = "#order.id")
fun update(order: Order) { ... }
```

---

### 7. Testing rules & examples

* **Unit tests**: Use `@ExtendWith(MockitoExtension::class)`, mock repositories and event publishers. Verify interactions and logic.
* **Controller tests**: Use `@WebMvcTest` + `MockMvc` for serialization/validation checks.
* **Integration tests**:

    * Use Testcontainers for MSSQL, Kafka, Redis.
    * Start the Spring context with test profile; apply Flyway migrations before tests.
    * Verify DB writes, event publication (consume from Testcontainers Kafka), and cache behavior (Redis).
* **Test artifacts**: Provide sample `pom.xml` test dependencies (JUnit 5, Mockito, Testcontainers, Spring Boot Test, Flyway).

---

### 8. Spring Batch guidance (if requested)

* Provide a `Job` + `Step` config with:

    * `ItemReader` reading from DB using `JdbcCursorItemReader` (SQL defined in resources/sql/batch/).
    * `ItemProcessor` mapping entities to export DTO.
    * `ItemWriter` writing CSV files or calling services.
* Make jobs restartable and idempotent; persist job metadata (default Spring Batch tables via provided scripts).

---

### 9. Security & validation

* Use `javax.validation` annotations on DTOs and `@Validated` on controllers.
* Security (authn/authz) is *out-of-scope*, but create pluggable filters if requested later.

---

### 10. Formatting & style requirements for generated Kotlin code

* Kotlin coding style: use data classes for DTOs, `internal` for package-internal types, explicit nullability.
* Avoid long functions; keep ≤ 40 LOC per method where possible.
* Provide KDoc comments for public classes and methods.
* Add `@Profile` or conditional beans for test/dev separation.

---

### 11. Example agent response structure (when asked to *generate* a service or files)

Agent must return a structured JSON (or clear file list) containing:

* `metadata`: {serviceName, package, features}
* `files`: array of {path, language, content}
* `tests`: list of test files (path + content)
* `instructions`: how to run locally (build/run/test), and any extra manual steps (e.g., create Kafka topic).
* `notes`: decisions, trade-offs, and places to customize (e.g., transactional outbox suggestion).

**Example output snippet:**

```json
{
  "metadata": { "service": "order-service", "package": "com.example.order" },
  "files": [
    { "path": "src/main/kotlin/com/example/order/controller/OrderController.kt", "content": "..."},
    ...
  ],
  "tests": [
    { "path": "src/test/kotlin/com/example/order/service/OrderServiceTest.kt", "content": "..." }
  ],
  "instructions": "Run `./mvnw spring-boot:run`. For integration tests run `./mvnw test` with Docker running..."
}
```

---

### 12. Example prompt templates to give to the agent (copy-paste)

* **Create entire service skeleton**

  > `Generate a new Kotlin Spring Boot service named <service-name> with base package <base-package>. Entities: [list]. For each entity generate controllers, services, repositories (JdbcTemplate), SQL files, domain events to Kafka, Redis cache rules, unit tests, and one integration test using Testcontainers.`

* **Create files for a specific entity**

  > `Add an Order entity (id: UUID, amount: DECIMAL, status: VARCHAR) and generate OrderRepository, OrderService, OrderController, SQL files, OrderCreated event, caching rules (cache getById 60s), and unit/integration tests.`

---

### 13. Additional best-practice suggestions (agent should proactively include)

* Prefer `NamedParameterJdbcTemplate` for readability.
* Use Flyway for migrations and keep schema in version control.
* Consider an **outbox table** if event delivery consistency is crucial.
* Keep SQL well-documented and test each query with representative data.
* Provide sample Docker Compose for local dev with MSSQL, Kafka, and Redis (optional).

---

### 14. Deliverable checklist (agent must verify)

* [ ] `pom.xml` with dependencies for Spring Boot, JdbcTemplate, Kafka, Redis, Flyway, Spring Batch, Testcontainers (test scope).
* [ ] `application.yml` with profiles.
* [ ] SQL files in `resources/sql/`.
* [ ] Repository using `NamedParameterJdbcTemplate`.
* [ ] Controller/Service/EventPublisher classes.
* [ ] Unit tests for controller, service, repository (mocks).
* [ ] Integration test(s) using Testcontainers (MSSQL, Kafka, Redis).
* [ ] Example Spring Batch job config (if requested).
* [ ] README with run/test instructions and notes about transaction/event ordering.

---

### 15. If the agent is uncertain about something, how it must behave

* Make a best-effort choice and document the assumption in `notes`.
* Do not halt or ask for clarification when minor details are missing; instead, generate a sensible default (e.g., TTL=60s, use UUID for id) and list it in `notes`
