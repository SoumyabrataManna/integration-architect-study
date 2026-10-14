# Event-Driven Integration: Error Handling Framework

Kafka, AWS SQS, IBM MQ, JMS, Dell Boomi, Spring Boot. Version 1.0. Scope: all async flows.

## 1. Overview

Event-driven integration decouples producers and consumers through asynchronous brokers. That brings its own error challenges: errors are discovered late, retry ownership moves to the broker or consumer, and partial failures in fan-out can silently corrupt downstream systems.

> **Objective:** every event has a deterministic outcome: successful processing, a classified failure with full context preserved, or safe replay via a monitored Dead Letter Queue. Silent event loss is never acceptable.

### 1.1 Core differences vs. synchronous error handling

| Concern | Synchronous (REST/SOAP) | Event-driven (Kafka/SQS/JMS) |
|---|---|---|
| Error discovery | Immediate, in the same request/response cycle | Delayed: discovered on consume, possibly hours later |
| Retry ownership | Caller retries the request | Broker or consumer manages retries |
| Delivery guarantee | Success or failure of the single call | At-least-once: the consumer must be idempotent |
| Partial failure | Clear rollback point | Fan-out: some consumers succeed, others fail |
| Ordering | Inherent in request sequence | Must be preserved explicitly on retry |
| Poison messages | Not applicable | Can block an entire Kafka partition indefinitely |

## 2. Error classification

Misclassification leads to infinite retry loops, blocked partitions or silent data loss.

| Class | Description | Examples | Retry? | Route to |
|---|---|---|---|---|
| Transient | Temporary, self-healing | DB timeout, downstream 503, network blip | Yes | Retry queue |
| Poison pill | Always fails regardless of retries | Malformed JSON, schema violation, corrupt message | No | DLQ immediately |
| Business | Data or logic violation | Duplicate event, invalid reference, rule violation | No | DLQ + alert |
| Infrastructure | Broker or system failure | Kafka broker down, SQS unavailable | Broker handles | Ops escalation |
| Ordering | Out-of-sequence events | Update arrives before create | Hold and retry | Staging table |

## 3. Idempotency

> **Rule #1:** all event consumers must be idempotent. At-least-once delivery means the same event will arrive more than once, especially during retry, rebalance or replay.

### 3.1 Processed event table pattern

Record every successfully processed event ID in the same transaction as the side effects.

```sql
CREATE TABLE processed_events (
  event_id       VARCHAR(255) PRIMARY KEY,
  topic          VARCHAR(200) NOT NULL,
  processed_at   TIMESTAMP NOT NULL DEFAULT NOW(),
  consumer_group VARCHAR(200),
  result         VARCHAR(50)   -- SUCCESS | SKIPPED | FAILED
);

-- Consumer logic (pseudocode)
BEGIN TRANSACTION;
  IF EXISTS (SELECT 1 FROM processed_events WHERE event_id = :id)
    THEN ROLLBACK;   -- already processed, safe to skip
  END IF;
  processBusinessLogic(event);
  INSERT INTO processed_events (event_id, topic, consumer_group)
    VALUES (:id, :topic, :group);
COMMIT;
```

### 3.2 Upsert pattern for state events

```sql
-- Safe to replay: same result however many times it runs
INSERT INTO orders (id, status, updated_at, version)
VALUES (:id, :status, :now, :version)
ON CONFLICT (id) DO UPDATE
  SET status     = EXCLUDED.status,
      updated_at = EXCLUDED.updated_at
  WHERE orders.version < EXCLUDED.version;   -- only update if newer
```

## 4. Retry strategy

Only transient errors are retried. Retrying business or poison-pill errors wastes resources and blocks pipelines.

### 4.1 Retry topic chain (Kafka)

The retry-topic pattern simulates exponential backoff at the broker level without blocking the main topic partition.

```
orders.created                 (main topic, first attempt)
  | transient failure
orders.created.retry.1         (delay: 1 min, attempt 2)
  | still failing
orders.created.retry.2         (delay: 5 min, attempt 3)
  | still failing
orders.created.retry.3         (delay: 30 min, attempt 4)
  | exhausted
orders.created.DLQ             (dead letter: ops review required)
```

```java
// Spring Kafka: @RetryableTopic handles the retry chain automatically
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 60000, multiplier = 5, maxDelay = 1800000),
    dltStrategy = DltStrategy.FAIL_ON_ERROR,
    exclude = {
        SchemaValidationException.class,   // poison pill: straight to DLQ
        BusinessRuleException.class        // business error: straight to DLQ
    }
)
@KafkaListener(topics = "orders.created", groupId = "order-processor")
public void consume(OrderEvent event) {
    orderService.process(event);
}
```

### 4.2 SQS retry configuration

| SQS setting | Recommended value | Purpose |
|---|---|---|
| Visibility timeout | 6x processing time (min 30s) | Prevent duplicate delivery during processing |
| maxReceiveCount | 3 to 5 | Retry budget before DLQ routing |
| DLQ redrive policy | Required on every queue | Capture exhausted messages |
| Message retention | 14 days (max) | Allow a replay window after the fix |
| Delay queue | 60s for retry queues | Backoff simulation in SQS |

### 4.3 IBM MQ / JMS retry

- Configure `BackoutThreshold` on the queue. MQ auto-routes to the backout queue after N failures.
- Use `BackoutRequeueQName` to define the DLQ (called the backout queue in MQ terminology).
- Set `RedeliveryDelay` on the MQ connection factory for timed backoff.
- Always inspect the `JMSXDeliveryCount` header to detect redeliveries and classify accordingly.
- For transactional JMS, use `SESSION_TRANSACTED`. A rollback automatically returns the message to the queue for retry.

## 5. Dead letter queue framework

> **Mandatory:** every event-driven consumer must have a DLQ. A DLQ is not a graveyard. It is an active operational component that needs alerting, monitoring and a replay workflow.

### 5.1 DLQ message envelope

Every message entering a DLQ carries full error context. Never strip headers or metadata on DLQ routing.

```json
{
  "eventId": "uuid-abc-123-xyz",
  "originalTopic": "orders.created",
  "partitionKey": "CUST-4421",
  "eventTimestamp": "2026-03-28T08:00:00Z",

  "failureReason": "NullPointerException: customerId is null at OrderMapper:42",
  "failureCode": "MAPPING_ERROR",
  "errorClass": "POISON_PILL",
  "retryCount": 3,
  "firstFailureAt": "2026-03-28T08:01:00Z",
  "lastFailureAt": "2026-03-28T08:31:00Z",

  "consumerGroup": "order-processor",
  "consumerHost": "pod-order-svc-7b4d",
  "dlqRoutedAt": "2026-03-28T09:00:00Z",

  "originalPayload": { }
}
```

### 5.2 DLQ operational workflow

| # | Step | Action / owner |
|---|---|---|
| 1 | Detection | Automated alert when DLQ depth > 0 (P1 flows) or over a threshold (P2/P3) |
| 2 | Triage | Ops/dev reads the DLQ envelope, classifies the error and finds the root cause |
| 3 | Fix | Deploy a fix (code, config or data correction) |
| 4 | Validate | Test the fix against a DLQ sample message in staging |
| 5 | Replay | Re-publish DLQ messages to the original topic with a replay consumer |
| 6 | Verify | Confirm all replayed messages processed; the DLQ drains to 0 |
| 7 | Post-mortem | Document root cause, SLA breach and preventive action; update the runbook |

### 5.3 DLQ replay consumer

```java
@Service
public class DlqReplayService {

  @KafkaListener(topics = "orders.created.DLQ", groupId = "dlq-replay")
  public void replay(DlqEnvelope dlqMessage) {

    // Validate the root cause is fixed before replaying
    if (!isFixDeployed(dlqMessage.getFailureCode())) {
      log.warn("Fix not deployed, skipping replay for {}", dlqMessage.getEventId());
      return;
    }

    // Re-publish to the original topic with the original partition key
    kafkaTemplate.send(
        dlqMessage.getOriginalTopic(),
        dlqMessage.getPartitionKey(),
        dlqMessage.getOriginalPayload());
    log.info("Replayed event {} to {}", dlqMessage.getEventId(),
        dlqMessage.getOriginalTopic());
  }
}
```

## 6. Poison pill handling

> **Critical (Kafka):** an undetected poison pill blocks the entire partition. No later message on that partition is processed until it is resolved.

### 6.1 Detection

- Schema validation: validate against Avro/JSON Schema before processing and reject immediately on failure.
- Checksum / signature verification: detect corrupt or tampered messages before deserialization.
- Deserialization error handler: catch `DeserializationException` separately. These are always poison pills.
- Retry count threshold: after N failures with the same error signature, classify as a poison pill and fast-fail to the DLQ.

### 6.2 Spring Kafka configuration

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> template) {

  // DLQ publisher for exhausted messages
  DeadLetterPublishingRecoverer recoverer =
      new DeadLetterPublishingRecoverer(template,
          (record, ex) -> new TopicPartition(record.topic() + ".DLQ", record.partition()));

  // Backoff: 3 retries with 5s intervals
  FixedBackOff backOff = new FixedBackOff(5000L, 3L);

  DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

  // These exceptions are never retried: straight to the DLQ
  handler.addNotRetryableExceptions(
      JsonParseException.class,
      SchemaValidationException.class,
      DeserializationException.class,
      BusinessRuleViolationException.class);

  return handler;
}
```

## 7. Saga pattern: distributed transaction error handling

When an event triggers a multi-step flow across services, partial failures need explicit compensating actions. The saga pattern keeps consistency without distributed locks.

### 7.1 Choreography vs. orchestration

| Aspect | Choreography | Orchestration |
|---|---|---|
| Control | Each service reacts to events; no central coordinator | A central orchestrator drives each step |
| Coupling | Loose: services don't know each other | Moderate: the orchestrator knows all participants |
| Visibility | Low: hard to see full flow state | High: the orchestrator tracks full state |
| Failure handling | Each service publishes compensating events | The orchestrator issues compensating commands |
| Best for | Simple 2-3 step flows | Complex multi-step flows with conditional logic |

### 7.2 Compensating transaction flow

```
Order placement saga: failure and compensation

Step 1: OrderService     -> OrderPlaced event      [SUCCESS]
Step 2: InventoryService -> StockReserved event    [SUCCESS]
Step 3: PaymentService   -> PaymentCharged event   [SUCCESS]
Step 4: ShippingService  -> CreateShipment         [FAILED]

Compensation (reverse order):
  PaymentService   <- PaymentRefundRequested -> RefundProcessed
  InventoryService <- StockReleaseRequested  -> StockReleased
  OrderService     <- OrderCancelled         -> OrderVoided
```

- Compensate in reverse order of execution.
- Compensation events must also be idempotent.

### 7.3 Saga state persistence

- Persist saga state after every step. Never rely on in-memory state.
- Store: saga ID, current step, status (IN_PROGRESS / COMPLETED / COMPENSATING / FAILED), timestamps.
- On restart or rebalance, resume from the last persisted state. Do not replay completed steps.
- Use optimistic locking (a version field) on saga state to prevent concurrent execution.

## 8. Circuit breaker in event-driven flows

The circuit breaker stops a consumer from overwhelming a failing downstream service that is invoked during event processing.

| State | Behavior | Transition |
|---|---|---|
| CLOSED | Normal processing; events consumed and processed | Opens when the failure rate exceeds the threshold (for example 50% in 10 calls) |
| OPEN | Consumer pauses; events stay in the topic unprocessed | Moves to HALF-OPEN after a cooldown (for example 30s) |
| HALF-OPEN | A single test event is processed to probe recovery | Closes on success; re-opens on failure |

```java
@KafkaListener(topics = "orders.created")
public void consume(OrderEvent event) {
    circuitBreaker.executeRunnable(() -> {
        inventoryClient.reserveStock(event.getOrderId(), event.getItems());
    });
}
```

```yaml
resilience4j.circuitbreaker.instances.inventoryService:
  failureRateThreshold: 50
  minimumNumberOfCalls: 10
  waitDurationInOpenState: 30s
  permittedNumberOfCallsInHalfOpenState: 3
```

## 9. Broker-specific error handling

### 9.1 Apache Kafka

Key risk: unhandled errors block the partition. Always configure an error handler and a DLQ.

- Consumer group rebalancing: if a consumer crashes mid-processing, uncommitted offsets are reprocessed by the new consumer, so ensure idempotency.
- Offset commit: use `MANUAL_IMMEDIATE` acknowledgment and commit only after successful processing.
- `enable.auto.commit = false`. Auto-commit commits before processing completes.
- Lag monitoring: consumer lag above a threshold means the consumer is stuck or too slow.
- Schema registry: use Confluent Schema Registry with Avro/JSON Schema. Schema evolution errors are the top poison-pill source in Kafka.

### 9.2 AWS SQS / SNS

- Standard queue: at-least-once delivery, no ordering. Always check for duplicates.
- FIFO queue: exactly-once processing within a message group, ordered. Use for saga sequences.
- SNS fan-out + SQS: each subscriber has its own SQS queue and DLQ, so failures are isolated per consumer.
- Lambda DLQ: for Lambda consumers, configure both the SQS DLQ (message level) and a Lambda destination (invocation level).

### 9.3 IBM MQ / JMS

- Backout queue: the IBM MQ equivalent of a DLQ. Set `BackoutThreshold` on every queue definition.
- `JMSXDeliveryCount`: inspect it in every consumer to detect redeliveries.
- Transactional sessions: use `SESSION_TRANSACTED` to process the message and commit the DB atomically (XA).
- MQ Explorer: monitor queue depth, backout queue depth and channel status.

### 9.4 Dell Boomi

- Use the Boomi JMS or Kafka connector as the Start shape for event-driven processes.
- Enable "Acknowledge on process success" on the JMS connector so it does not auto-ack before processing completes.
- Wrap the whole processing flow in a Try/Catch shape.
- In the catch path, extract `com.boomi.process.error` properties and write them to an error log table using a separate connection.
- Use a Decision shape to classify: transient errors re-queue to the retry topic; poison pills go to the DLQ connector; business errors alert and go to the DLQ.
- For Kafka, use the Boomi Kafka connector with a manual offset commit ("Commit Offset" set to "On Success").
- Add a Notify shape on every unrecoverable error path. Never let failures pass silently.

## 10. Event ordering and sequencing errors

Out-of-order events cause incorrect state mutations that are very hard to detect later.

### 10.1 Ordering guarantees by broker

| Broker | Ordering guarantee | Key for ordering |
|---|---|---|
| Kafka | Ordered within a partition only | Use a consistent partition key (for example customerId) |
| SQS Standard | No ordering guarantee | Use a sequence number in the payload; handle reordering in the consumer |
| SQS FIFO | Ordered within a message group | Use `MessageGroupId` consistently |
| IBM MQ | Ordered within a queue | One queue per ordered stream; avoid parallel consumers |
| JMS | Ordered within session/queue | One single-threaded consumer per ordered stream |

### 10.2 Handling out-of-order events

- Sequence number validation: include a sequence number or version in every event; reject or hold if a gap is detected.
- Staging table hold: if event N+1 arrives before N, persist it in staging and process it when N fills the gap.
- Timestamp-based resolution: for state events, apply only if the event timestamp is newer than the current record's.
- Event sourcing: store all events as an immutable log and rebuild state by replaying in order.

## 11. Observability and alerting

| Metric | Applies to | Alert threshold | Root-cause indicator |
|---|---|---|---|
| Consumer lag | Kafka | Growing trend | Consumer stuck or too slow |
| DLQ depth | All brokers | > 0 for P1 flows | Unresolved processing failures |
| Retry exhaustion rate | All brokers | Spike above baseline | Persistent downstream failure |
| Circuit breaker OPEN | Consumers with CB | Any OPEN transition | Downstream service degraded |
| Idempotency hit rate | All consumers | Sustained spike | Producer publishing duplicates |
| Processing time P99 | All consumers | Exceeds SLA threshold | Downstream latency / DB pressure |
| Poison pill rate | Kafka, SQS | Any non-zero | Schema change / bad producer |
| Saga compensation rate | Saga flows | Spike above baseline | Downstream service instability |

## 12. Key principles

1. Idempotency is not optional. At-least-once delivery is the default in every broker.
2. Classify before retrying. Transient errors retry with backoff; poison pills and business errors go straight to the DLQ.
3. Every consumer needs a DLQ, and its depth must be monitored and alerted on.
4. Fast-fail poison pills. Configure not-retryable exceptions explicitly.
5. Kafka partition health depends on error handling. One unhandled exception can halt a partition.
6. Saga compensation must be idempotent too.
7. Offset commit means processing acknowledgment. Never auto-commit in integration.
8. A DLQ is an active operational component: alerting, a triage runbook, root-cause analysis and a validated replay workflow.
9. Preserve all context in the DLQ envelope: original payload, failure reason, error class, retry count, timestamps.
10. Event ordering requires intentional design: consistent partition keys in Kafka, FIFO queues in SQS, sequence-number validation in consumers.
