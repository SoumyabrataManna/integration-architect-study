# Database Error Handling Framework

For enterprise integration systems. Applies to Boomi, Spring Boot and APIGEE. Version 1.0.

## 1. Overview

Standard framework for handling database errors in enterprise integration systems: classification, retry policies, transaction management, idempotency and observability.

> **Goal:** no database error causes silent data loss, partial commits or system instability. Every error is classified, handled deterministically and observable.

## 2. Error classification

Classification drives retry eligibility, routing and alerting.

| Class | Examples | Handling strategy | Retry? |
|---|---|---|---|
| Transient | Connection timeout, deadlock, lock wait, pool exhaustion | Retry with exponential backoff | Yes (max 3-5) |
| Fatal | Schema mismatch, auth failure, tablespace full | Fail fast, alert ops immediately | No |
| Constraint | Unique key violation, FK failure, NOT NULL | Upsert or skip; route to DLQ | No |
| Business | No rows found, orphan record, duplicate detection | Route to business exception handler / DLQ | No |

## 3. Connection-level error handling

### 3.1 Connection pool configuration

Pool exhaustion is the most common integration-layer DB failure. Configure pool parameters explicitly. Never rely on framework defaults.

```properties
# HikariCP recommended settings
hikari.maximumPoolSize=20
hikari.minimumIdle=5
hikari.connectionTimeout=30000     # 30s max wait for a connection
hikari.idleTimeout=600000          # 10 min idle before release
hikari.maxLifetime=1800000         # 30 min max connection age
hikari.connectionTestQuery=SELECT 1   # validate on borrow
```

### 3.2 Stale connection prevention

- Enable connection validation before use (`testOnBorrow=true` or a connection test query).
- Set `maxLifetime` below the DB-side timeout so stale connections do not fail silently.
- Monitor pool wait time. Spikes indicate saturation or long-running transactions.

## 4. Transaction error handling

### 4.1 Core principle

> **Rule:** never allow a partial write to commit. All steps in a business transaction succeed together or roll back entirely.

```java
@Transactional(rollbackFor = Exception.class)
public void processOrder(Order order) {
    orderRepo.save(order);        // Step 1
    inventoryRepo.deduct(order);  // Step 2: failure rolls back Step 1
    auditRepo.log(order);         // Step 3
}
```

### 4.2 Transaction propagation modes

The wrong propagation mode is a leading cause of silent data loss and `UnexpectedRollbackException` in integration flows.

| Mode | Behavior | Use case |
|---|---|---|
| REQUIRED | Joins the existing transaction or creates a new one | Default for most business operations |
| REQUIRES_NEW | Suspends the outer transaction, creates an independent one | Audit logs, error recording |
| NESTED | Savepoint within the outer transaction; partial rollback possible | Optional steps inside a larger flow |
| NOT_SUPPORTED | Runs outside any transaction | Read-only reporting queries |

### 4.3 Error logging must survive rollback

If the error logger uses the same transaction, a rollback erases the error record too.

```java
// WRONG: the error log is rolled back with the outer transaction
@Transactional
public void logError(ErrorRecord error) {
    errorRepo.save(error);
}

// CORRECT: REQUIRES_NEW commits independently
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logError(ErrorRecord error) {
    errorRepo.save(error);   // survives outer transaction rollback
}
```

## 5. Constraint and integrity violation handling

### 5.1 Duplicate key / unique constraint

Constraint violations are never retried, because they will not self-heal. Detect, then upsert or skip.

```java
try {
    repo.save(entity);
} catch (DataIntegrityViolationException e) {
    if (isDuplicateKey(e)) {
        repo.update(entity);   // upsert: safe to proceed
    } else {
        throw e;               // re-throw other constraint types
    }
}

private boolean isDuplicateKey(DataIntegrityViolationException e) {
    return e.getCause() instanceof ConstraintViolationException
        && e.getMessage().contains("unique constraint");
}
```

### 5.2 Foreign key / referential integrity

- Cause: a child record arrives before its parent is committed (a sequencing issue).
- Fix: reorder the pipeline so the parent is created first.
- Alternative: stage child records in a holding table until the parent exists.
- Never retry FK violations blindly. They need a data or sequence correction.

## 6. Deadlock handling

Deadlocks are transient: the database rolls back one transaction, and that transaction is safe to retry.

```java
@Retryable(
    value = { DeadlockLoserDataAccessException.class },
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2)   // 500ms, 1s, 2s
)
@Transactional
public void updateRecord(Record r) {
    repo.update(r);
}
```

### 6.1 Prevention

- Acquire locks in the same order across all transactions.
- Keep transactions short. Never call external APIs inside a transaction.
- Use `SELECT FOR UPDATE SKIP LOCKED` for queue-table fan-out patterns.
- Avoid row-level locks held across multiple round trips.

## 7. Optimistic vs. pessimistic locking

| | Optimistic | Pessimistic |
|---|---|---|
| Mechanism | `@Version` field checked on update | `SELECT FOR UPDATE`: row locked on read |
| Failure mode | `OptimisticLockException` on stale read | Deadlock or timeout under high contention |
| Recovery | Reload the entity and retry the business logic | Handle the timeout, reduce lock scope |
| Best for | Low-contention integration flows | High-contention critical sections |

```java
// Optimistic: use for most integration flows
@Version
private Long version;   // JPA checks this on update

// Pessimistic: use sparingly for critical sections
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT e FROM Order e WHERE e.id = :id")
Order findByIdForUpdate(@Param("id") Long id);
```

## 8. Idempotency at the database layer

Integration flows replay messages during failures. Every write must be safe to execute more than once.

### 8.1 Upsert pattern

```sql
INSERT INTO orders (id, status, updated_at)
VALUES (:id, :status, :now)
ON CONFLICT (id) DO UPDATE
SET status = EXCLUDED.status,
    updated_at = EXCLUDED.updated_at;
-- Safe to run N times; same result every time
```

### 8.2 Processed event table

For event-driven flows, track event IDs to prevent reprocessing.

```sql
CREATE TABLE processed_events (
  event_id     VARCHAR(255) PRIMARY KEY,
  source       VARCHAR(100) NOT NULL,
  processed_at TIMESTAMP NOT NULL,
  process_name VARCHAR(200)
);

-- Before processing: check
SELECT 1 FROM processed_events WHERE event_id = :eventId;

-- After successful processing: mark
INSERT INTO processed_events (event_id, source, processed_at)
VALUES (:eventId, :source, NOW());
```

## 9. Middleware-specific implementation

### 9.1 Dell Boomi

- Wrap DB connector operations in a Try/Catch shape.
- In the catch path, extract `com.boomi.process.error.message` as a document property.
- Write to the error log table using a separate DB connector (an independent connection, so it is not rolled back).
- Use a Decision shape to classify the error: transient errors to the retry queue, constraint errors to the DLQ.
- Use a Notify shape or the Mail connector for immediate alerting on fatal errors.

### 9.2 Spring Boot microservices

Use a global exception handler to map DB errors to business error codes. Never expose raw SQL errors to callers.

```java
@ControllerAdvice
public class DatabaseExceptionHandler {

  @ExceptionHandler(DataIntegrityViolationException.class)
  public ResponseEntity<ErrorResponse> handleConstraint(DataIntegrityViolationException e) {
    return ResponseEntity.status(409)
        .body(new ErrorResponse("DUPLICATE_RECORD", e.getMessage()));
  }

  @ExceptionHandler(TransientDataAccessException.class)
  public ResponseEntity<ErrorResponse> handleTransient(TransientDataAccessException e) {
    return ResponseEntity.status(503)
        .body(new ErrorResponse("DB_TRANSIENT_ERROR", "Retry after delay"));
  }
}
```

## 10. Observability and alerting

| Metric | Alert threshold | Indicates |
|---|---|---|
| Connection pool wait time | > 5 seconds | Pool exhaustion risk |
| Deadlock frequency | > 10 / hour | Lock ordering issue |
| Transaction rollback rate | Spike above baseline | Systematic data quality problem |
| Slow query count | > defined threshold | Missing index or schema issue |
| DLQ depth (DB errors) | > 0 for P1 flows | Immediate ops review required |
| Constraint violation rate | Any sustained increase | Upstream idempotency gap |

## 11. Key principles

1. Classify before acting. Transient errors retry; constraint errors upsert or skip; fatal errors alert immediately.
2. Error logging must survive rollback. Use `REQUIRES_NEW` for audit and error records.
3. All writes must be idempotent. Replay safety is non-negotiable in integration systems.
4. Never expose raw DB errors upstream. Map them to business error codes with an appropriate HTTP status.
5. Deadlocks are normal. Handle them with retry and prevent them with consistent lock ordering.
6. Pool health is DB health. Monitor wait time and size pools explicitly for every environment.
