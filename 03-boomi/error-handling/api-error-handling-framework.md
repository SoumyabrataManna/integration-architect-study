# API Error Handling Framework

SOAP, REST, GraphQL, gRPC and async/event-driven APIs. Applies to Boomi, Spring and APIGEE. Version 1.0.

## 1. Overview

Standard error handling for all API types used in enterprise integration. It covers error classification, protocol-specific patterns, retry strategies, circuit breakers, error response standards and observability.

> **Objective:** every API interaction must produce a deterministic outcome: success, a classified error with actionable context, or a safe retry. Silent failures and unclassified errors are not acceptable.

### 1.1 API types in scope

| API type | Protocol | Common use cases in integration |
|---|---|---|
| REST | HTTP/HTTPS + JSON/XML | Microservices, SaaS connectors, public APIs |
| SOAP | HTTP + XML (WSDL) | SAP, legacy ERP, banking, insurance |
| GraphQL | HTTP + JSON (query language) | Flexible front-end APIs, unified data APIs |
| gRPC | HTTP/2 + Protobuf | Internal microservice-to-microservice, high throughput |
| Async API | Kafka / SQS / JMS / WebSocket | Event-driven integration, CDC, notifications |

## 2. Universal error classification

All errors fall into one of four classes. Classify before applying any handling strategy.

| Class | Description | Examples | Retry? | Alert? |
|---|---|---|---|---|
| Transient | Temporary, self-healing failures | Timeout, 503, network blip, 429 | Yes | Threshold |
| Client | Caller error: bad request or auth | 400, 401, 403, 404, SOAP fault | No | Log only |
| Server | Downstream system failure | 500, 502, 504, gRPC INTERNAL | Limited | Yes |
| Business | Data or logic violation | 409 Conflict, validation error, schema fail | No | DLQ route |

## 3. REST API error handling

Protocol: HTTP/HTTPS. Format: JSON/XML. Standard: RFC 7807 Problem Details.

### 3.1 HTTP status code handling matrix

| Code | Meaning | Category | Action | Retry? |
|---|---|---|---|---|
| 200 / 201 | Success / Created | Success | Process response | N/A |
| 400 | Bad Request | Client error | Fix request payload | No |
| 401 | Unauthorized | Auth error | Refresh token / re-auth | Once |
| 403 | Forbidden | Auth error | Alert: permission issue | No |
| 404 | Not Found | Client error | Log, skip, or escalate | No |
| 409 | Conflict / Duplicate | Business error | Upsert or skip | No |
| 422 | Unprocessable Entity | Validation error | Log payload, route to DLQ | No |
| 429 | Rate Limited | Transient | Retry after `Retry-After` header | Yes |
| 500 | Internal Server Error | Server error | Retry 1-2x, then DLQ + alert | Limited |
| 502 / 504 | Bad Gateway / Timeout | Transient | Retry with backoff | Yes |
| 503 | Service Unavailable | Transient | Circuit breaker + retry queue | Yes |

### 3.2 RFC 7807 standard error response

Always return structured error envelopes. Never expose stack traces or raw exception messages to API consumers.

```json
{
  "type": "https://api.company.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Field 'customerId' is required and was not provided.",
  "instance": "/orders/create",
  "traceId": "abc-123-xyz",
  "timestamp": "2026-03-28T10:00:00Z"
}
```

### 3.3 Retry with exponential backoff

```java
@Retryable(
    value = { HttpServerErrorException.class, ResourceAccessException.class },
    maxAttempts = 4,
    backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 16000)
)
public ResponseEntity<String> callApi(String url) {
    return restTemplate.postForEntity(url, request, String.class);
}
// Retry schedule: 1s -> 2s -> 4s -> 8s -> DLQ
```

### 3.4 Token and auth error handling

- On 401: attempt a token refresh once using the `refresh_token` grant.
- On a second 401 after refresh: fail, alert, and do not retry in a loop.
- On 403: do not retry. It is a permissions issue, not a token issue.
- Cache tokens with a TTL slightly below expiry (for example 5 minutes before `exp`).
- Use OAuth2 `client_credentials` for service-to-service calls; avoid hardcoded API keys.

## 4. SOAP API error handling

Protocol: HTTP + XML. Format: WSDL / SOAP envelope. Common in SAP, ERP and banking.

### 4.1 SOAP fault structure

> **Critical:** a SOAP fault can come back with HTTP 200. Never rely on HTTP status alone. Always inspect the response envelope for `<soap:Fault>`.

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <soap:Fault>
      <faultcode>soap:Client</faultcode>
      <faultstring>Invalid order ID format</faultstring>
      <detail>
        <errorCode>ORD-4001</errorCode>
        <message>Order ID must be 10 digits</message>
      </detail>
    </soap:Fault>
  </soap:Body>
</soap:Envelope>
```

### 4.2 SOAP fault code classification

| Fault code | Meaning | Class | Action |
|---|---|---|---|
| soap:Client | Malformed message, invalid input from caller | Client error | Fix request |
| soap:Server | Server-side processing failure | Server error | Retry / alert |
| soap:VersionMismatch | SOAP version incompatibility | Fatal | Fix WSDL binding |
| soap:MustUnderstand | Required header not understood by server | Fatal | Fix header config |

### 4.3 SOAP error handling in Boomi

- Use an XML profile to parse the SOAP response envelope.
- Add a Decision shape that checks for the `<soap:Fault>` element.
- Route the fault path: classify faultcode, then retry queue (`soap:Server`) or DLQ (`soap:Client`).
- Extract `faultstring` and `detail/errorCode` as dynamic document properties for logging.
- For SAP SOAP, also check for a `<SAP:Error>` extension inside `<detail>`. SAP adds its own error codes.

## 5. GraphQL API error handling

Protocol: HTTP + JSON. Usually HTTP 200. Errors can coexist with partial data.

### 5.1 Error characteristics

> **Critical:** GraphQL typically returns HTTP 200. Errors appear in the `errors` array of the response body, even when partial data was returned. Always check the `errors` field.

```json
{
  "data": {
    "order": null,
    "customer": { "id": "C001", "name": "Acme Corp" }
  },
  "errors": [
    {
      "message": "Order not found",
      "locations": [{ "line": 3, "column": 5 }],
      "path": ["order"],
      "extensions": {
        "code": "RESOURCE_NOT_FOUND",
        "classification": "DataFetchingException"
      }
    }
  ]
}
```

### 5.2 Handling strategy

| Extension code | Classification | Action |
|---|---|---|
| UNAUTHENTICATED | Auth error | Refresh token, retry once |
| FORBIDDEN | Auth error | Alert: scope/permission issue |
| BAD_USER_INPUT | Client error | Log and route to DLQ |
| RESOURCE_NOT_FOUND | Business error | Skip or escalate based on SLA |
| INTERNAL_SERVER_ERROR | Server error | Retry with backoff, alert on repeat failures |

### 5.3 Partial response handling

- If `errors` exists and `data` is null, treat it as a full failure.
- If `errors` exists and `data` is partially populated, process the data and route the failed paths to the DLQ.
- Never silently accept a response with errors. Always log and classify.

## 6. gRPC error handling

Protocol: HTTP/2 + Protobuf. Binary transport. Status codes from `google.rpc.Code`.

### 6.1 Status code matrix

| Code | Name | Meaning | REST equivalent | Action |
|---|---|---|---|---|
| 0 | OK | Success | 200 | Process |
| 1 | CANCELLED | Client cancelled | 499 | Log only |
| 2 | UNKNOWN | Unclassified server error | 500 | Retry + alert |
| 3 | INVALID_ARGUMENT | Bad request | 400 | Fix, no retry |
| 4 | DEADLINE_EXCEEDED | Timeout | 504 | Retry with backoff |
| 5 | NOT_FOUND | Resource not found | 404 | Skip or escalate |
| 7 | PERMISSION_DENIED | Authorization failure | 403 | Alert, no retry |
| 8 | RESOURCE_EXHAUSTED | Rate limit / quota | 429 | Retry with delay |
| 14 | UNAVAILABLE | Server temporarily down | 503 | Circuit breaker + retry |

### 6.2 Rich error detail

```java
Status status = Status.newBuilder()
    .setCode(Code.INVALID_ARGUMENT.getNumber())
    .setMessage("Customer ID must be a valid UUID")
    .addDetails(Any.pack(BadRequest.newBuilder()
        .addFieldViolations(FieldViolation.newBuilder()
            .setField("customerId")
            .setDescription("Must be UUID format")
            .build())
        .build()))
    .build();
responseObserver.onError(StatusProto.toStatusRuntimeException(status));
```

### 6.3 Retry configuration

- Retry-eligible codes: `UNAVAILABLE`, `DEADLINE_EXCEEDED`, `RESOURCE_EXHAUSTED`.
- Never retry: `INVALID_ARGUMENT`, `NOT_FOUND`, `PERMISSION_DENIED`, `ALREADY_EXISTS`.
- Use the gRPC built-in retry policy in the service config, in preference to client-side retry loops.
- Always set a deadline on every call. Never rely on a server-side timeout alone.

## 7. Async / event-driven API error handling

Protocols: Kafka, AWS SQS, JMS, WebSocket, webhook.

### 7.1 Core differences from synchronous APIs

| Concern | Sync (REST/SOAP/gRPC) | Async (Kafka/SQS/JMS) |
|---|---|---|
| Error visibility | Immediate, synchronous | Delayed, discovered on consume |
| Retry ownership | Caller retries the request | Broker or consumer retries |
| Delivery guarantee | Once: success or fail | At-least-once: must be idempotent |
| Poison messages | Not applicable | Can block an entire partition (Kafka) |

### 7.2 Dead letter queue pattern

> **Mandatory:** every async consumer must have a DLQ. Messages that exhaust retries must never be silently discarded. Route them to the DLQ with the full error context preserved.

```json
{
  "originalTopic": "orders.created",
  "eventId": "uuid-abc-123",
  "source": "OrderService",
  "failureReason": "NullPointerException at OrderMapper.java:42",
  "failureCode": "MAPPING_ERROR",
  "retryCount": 3,
  "failureTimestamp": "2026-03-28T10:00:00Z",
  "originalPayload": { }
}
```

### 7.3 Kafka retry topic chain

```
orders.created            (original topic)
 -> orders.retry.1        (delay: 1 min, transient errors)
 -> orders.retry.2        (delay: 5 min, 2nd attempt)
 -> orders.retry.3        (delay: 30 min, final attempt)
 -> orders.DLQ            (exhausted: ops review required)
```

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 60000, multiplier = 5),
    exclude = { ValidationException.class, SchemaException.class }
)
@KafkaListener(topics = "orders.created")
public void consume(OrderEvent event) { ... }
```

### 7.4 Webhook error handling

- Return HTTP 2xx immediately on receipt and process asynchronously.
- Never perform business logic synchronously in the webhook handler.
- If downstream processing fails after the 2xx, use an internal retry queue.
- Validate the webhook signature (HMAC) before processing. Reject invalid payloads with 401.
- Respond within 5 seconds. Most webhook providers retry on timeout.

## 8. Circuit breaker pattern

A circuit breaker prevents cascade failures by stopping calls to a failing downstream API after repeated errors, so the system can recover without being flooded.

| State | Behavior | Transition |
|---|---|---|
| CLOSED | Normal operation, all calls pass through | Opens when the failure rate exceeds the threshold (for example 50%) |
| OPEN | All calls fail immediately, no downstream call made | Moves to HALF-OPEN after a cooldown period |
| HALF-OPEN | A single probe call is allowed to test recovery | Closes on success; re-opens on failure |

```java
// Resilience4j circuit breaker (Spring Boot)
@CircuitBreaker(name = "orderApi", fallbackMethod = "fallback")
public OrderResponse callOrderApi(OrderRequest req) {
    return restTemplate.postForObject(orderApiUrl, req, OrderResponse.class);
}

public OrderResponse fallback(OrderRequest req, Exception e) {
    // Return a cached response or queue for later retry
    return OrderResponse.queued(req.getId(), "Service temporarily unavailable");
}
```

```yaml
# application.yml
resilience4j.circuitbreaker.instances.orderApi:
  failureRateThreshold: 50
  waitDurationInOpenState: 30s
  slidingWindowSize: 10
```

## 9. APIGEE API gateway error handling

APIGEE acts as the entry and exit point for managed APIs. Gateway-level error handling protects backends and keeps error responses consistent.

### 9.1 FaultRule configuration

```xml
<FaultRules>
  <FaultRule name="BackendTimeout">
    <Condition>(fault.name = "timeout")</Condition>
    <Step><Name>RF-503-ServiceUnavailable</Name></Step>
  </FaultRule>
  <FaultRule name="AuthFailure">
    <Condition>(fault.name = "InvalidAccessToken")</Condition>
    <Step><Name>RF-401-Unauthorized</Name></Step>
  </FaultRule>
</FaultRules>
<DefaultFaultRule>
  <Step><Name>RF-500-InternalError</Name></Step>
</DefaultFaultRule>
```

### 9.2 Checklist

- Define FaultRules for every known fault condition. Never rely on the DefaultFaultRule alone.
- Use RaiseFault policies to normalize error responses to RFC 7807.
- Strip internal error details (stack traces, server hostnames) before returning to the consumer.
- Add a correlation/trace ID to every error response.
- Implement a rate limit (Quota policy) with a 429 response and a `Retry-After` header.
- Log all faults to Cloud Logging or Splunk with flow name, fault name and client ID.

## 10. Boomi API error handling

Boomi integrations calling external APIs need error handling at both connector and process level.

### 10.1 HTTP client

- Enable "Return HTTP Error Response" on the HTTP Client connector. Otherwise 4xx/5xx bypass the error path.
- Wrap the connector call in a Try/Catch shape.
- In the catch path, extract `com.boomi.process.error.statuscode` as a document property.
- Use a Decision shape to route by status code: 429/5xx to the retry queue, 4xx to the DLQ, 401 to the re-auth flow.
- Never leave the error path empty. Silent failures create ghost transactions.

### 10.2 SOAP connector

- After every SOAP connector call, add a Map shape to check for `<soap:Fault>` in the response.
- Route the fault path through a Decision shape on the faultcode value.
- `soap:Server` faults go to the retry queue; `soap:Client` faults go to the DLQ with the payload logged.
- For SAP SOAP, inspect the `<SAP:Error>` extension block separately.

### 10.3 Retry process pattern

```
Start -> HTTP Connector -> [Success] -> continue processing
              | [Try/Catch]
        Extract error properties
              |
        Decision: status code
     +--------+---------+
   429/5xx    4xx       401
     |         |         |
 RetryCounter DLQ write Re-auth
     |                   |
 Wait + loop          Retry once
 (max 3x)                |
     |              [still 401]
 DLQ + notify -> Alert
```

## 11. Observability and alerting

| Metric | Applies to | Alert condition | Indicates |
|---|---|---|---|
| 4xx error rate | REST, GraphQL, gRPC | > 5% of calls | Client contract violation |
| 5xx error rate | REST, SOAP, gRPC | > 1% sustained | Downstream instability |
| Circuit breaker OPEN | All sync APIs | Any transition | Cascade risk: immediate review |
| DLQ depth | Async APIs, Boomi | > 0 for P1 flows | Ops review required |
| Retry exhaustion rate | All API types | Spike above baseline | Persistent downstream failure |
| Auth failure rate | REST, SOAP, gRPC | > N per hour | Token/credential issue |
| P99 API latency | All sync APIs | Exceeds SLA threshold | Performance degradation risk |

## 12. Key principles

1. HTTP 200 does not mean success. Always inspect SOAP and GraphQL response bodies for errors.
2. Classify before acting. Transient errors retry; client errors get fixed; business errors route to the DLQ.
3. Every async consumer needs a DLQ. Messages that exhaust retries must never be silently dropped.
4. Auth errors have two causes. 401 means an expired or invalid token (retry once after refresh); 403 means wrong permissions (alert, no retry).
5. Circuit breakers prevent cascades. Open on threshold, probe on cooldown, never flood a degraded system.
6. Standardize error responses: RFC 7807 for REST, structured faults for SOAP, extensions for GraphQL.
7. Never expose internals. Strip stack traces, server hostnames and raw DB errors from API responses.
8. Traceability is mandatory. Every error response carries a correlation/trace ID end to end.
