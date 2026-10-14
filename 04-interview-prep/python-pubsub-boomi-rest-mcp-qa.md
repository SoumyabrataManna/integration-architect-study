# Technical Round Q&A: Python, Pub/Sub, Boomi, REST API and MCP

Fifty practical interview questions across five topics (Python, GCP Pub/Sub, Boomi, REST API design, MCP), each with a structured answer outline. Company-neutral. Code samples from the original source were flattened and are omitted here; the outlines describe what to code.


## Python


### Q1. What are the main Python data types and when would you use each in an integration context?

*Level: Basic*

- **String (str):** API field values, JSON keys, error messages. Most data from external APIs arrives as strings.
- **Integer / Float:** numeric values — billing amounts, record counts, confidence scores. Always use Decimal for financial amounts, never float (0.1 + 0.2 is not 0.3 in float).
- **List:** ordered collection of records from an API response. Iterate, filter, sort. Example: list of account records from Salesforce.
- **Dictionary (dict):** key-value pairs — a single API record, a JSON object. Most API responses are dicts or lists of dicts.
- **Boolean:** status flags, validation results (is_valid, needs_review). Also used in conditional logic.
- **None:** represents missing or null values from APIs. Always check 'if value is not None' before processing — never assume a field is populated.

### Q2. How do you read a JSON file and a CSV file in Python? Show a simple example of each.

*Level: Basic*

- **JSON files:** use the built-in json module. json.load() reads from a file object. json.loads() parses a JSON string. Most API responses and configuration files are JSON.
- **CSV files:** use the built-in csv module or pandas. csv.DictReader() reads each row as a dictionary — field names become keys. Good for data files without heavy transformation.
- **Error handling:** always wrap file operations in try/except. FileNotFoundError if the file does not exist. json.JSONDecodeError if the JSON is malformed. These are the two most common failures.
- **For large files:** do not load everything into memory at once. Use a generator or process line by line. A 1GB CSV should be streamed, not loaded into a list.
- **In an integration context:** you read JSON for API configs, historical mapping documents, and API responses stored as files. You read CSV for data exports from legacy systems.

### Q3. How do you call a REST API in Python and handle the response? Show authentication with a Bearer token.

*Level: Basic*

- Use the requests library — the standard for HTTP calls in Python. pip install requests.
- Always set a timeout — never make an API call without a timeout parameter. Without it, a hanging server will hang your application forever.
- Check the status code before processing the response — do not assume success. response.raise_for_status() raises an exception for 4xx and 5xx responses.
- Parse JSON response with response.json() — returns a Python dict. Handle json.JSONDecodeError if the response is not valid JSON.
- **Headers:** pass Authorization: Bearer token for OAuth2 APIs. Content-Type: application/json for POST requests with JSON bodies.

### Q4. Write a Python function that filters a list of records and transforms them. For example: filter active clients and format their names.

*Level: Basic*

- This tests basic list operations — filtering with a condition and transforming each element.
- **Two approaches:** (1) for loop with append — most readable for beginners. (2) list comprehension — more Pythonic, preferred for simple one-line transformations.
- Always validate the input — what if the list is empty? What if a record is missing a field? Use .get() with a default value to handle missing fields safely.
- Separation of concerns — the filter logic and the transform logic should be clearly distinct. In production code, they might even be separate functions.
- Type hints — add input and output types to the function signature. Makes the code self-documenting and helps IDEs catch errors.

### Q5. How do you handle exceptions in Python? What is the difference between except Exception and except specific errors?

*Level: Basic*

- Try/except is Python's error handling mechanism. Code in the try block runs normally. If an error occurs, execution jumps to the matching except block.
- **Catch specific exceptions, not bare except:** — bare except catches everything including KeyboardInterrupt and SystemExit, which should not be caught in application code
- Except Exception as e: catches all normal application errors. Better than bare except but still broad. Use for a final fallback after specific handlers.
- **Exception hierarchy:** FileNotFoundError is a subclass of OSError which is a subclass of Exception. Catch the most specific exception first
- Finally block: runs regardless of whether an exception occurred. Use for cleanup — closing file handles, releasing database connections, logging completion.
- **Re-raising:** if you catch an exception to log it but cannot handle it, re-raise it with 'raise' to let the caller handle it. Do not silently swallow exceptions.

### Q6. What is a Python decorator and when would you use one in an integration pipeline?

*Level: Basic*

- A decorator is a function that wraps another function to add behaviour without modifying its code. Uses the @decorator_name syntax above a function definition.
- **Common built-in decorators:** @staticmethod, @classmethod, @property. Popular library decorators: @lru_cache (caching), @retry (retry logic), @app.route (Flask routing).
- **Integration use cases:** @retry decorator to automatically retry failed API calls. @timer decorator to log how long each pipeline step takes. @lru_cache to cache API responses for the same input within a session.
- The @wraps decorator from functools.wraps should always be used inside a custom decorator — it preserves the wrapped function's name and docstring for debugging.
- **When not to use:** if the decorator logic is complex and makes the code harder to understand. Explicit is better than implicit in Python (PEP 20).

### Q7. LIVE CODING: Write a Python function that reads a list of mapping records, groups them by source system, and returns a summary count.

*Level: Basic*

- **Approach:** iterate through the list, use a dictionary to accumulate counts by source_system key. Python's collections.defaultdict makes this cleaner.
- **Think about edge cases:** empty list, records missing the source_system field, None values.
- After writing — test it mentally with a small example. Say what the output would be.

### Q8. LIVE CODING: Write a simple Python script that calls the Claude API and returns a mapping suggestion for a given field name.

*Level: Basic*

- **Structure:** import the anthropic library, create a client, build a simple prompt, call client.messages.create(), return the text response.
- Keep the prompt simple in a live coding context — you do not need the full RAG context. A basic prompt showing the pattern is enough.
- **Handle the response correctly:** response.content[0].text is the text output. Wrap in try/except for API errors.
- If asked to extend it — suggest adding historical context via RAG, confidence scoring, or JSON output format. Show you know what production looks like.

### Q9. What is the difference between synchronous and asynchronous Python? When would async record for an integration pipeline?

*Level: Core*

- **Synchronous (default Python):** one operation completes before the next starts. Simple, predictable. For a pipeline with 3 sequential API calls — total time = sum of all 3 call times.
- **Asynchronous (asyncio, async/await):** multiple operations can be in progress simultaneously. While waiting for API call 1 to return, Python starts API call 2. Total time approaches max of all call times instead of sum.
- **When async helps:** I/O-bound tasks — waiting for API responses, database queries, file reads. Multiple simultaneous API calls where you can start them all and wait for all to complete.
- **When async does NOT help:** CPU-bound tasks (computation, data transformation). For those use multiprocessing instead.
- **Practical integration example:** fetching data from Salesforce AND deal CRM simultaneously instead of sequentially. asyncio.gather() runs both calls in parallel — halves the total fetch time.
- **Caveat:** async code is more complex to write, test, and debug. Only add it when the performance gain justifies the complexity.

### Q10. What Python libraries are most important for integration and data pipeline work?

*Level: Core*

- **requests:** HTTP API calls. The standard for every REST call. Know get(), post(), Session(), timeout, raise_for_status().
- **anthropic:** Claude API SDK. Know client.messages.create(), model names, response.content[0].text.
- **json:** built-in serialisation and deserialisation: json.load(), json.loads(), json.dump(), json.dumps().
- **boto3:** AWS SDK. S3 uploads/downloads, SQS messages, Secrets Manager.
- **pandas:** DataFrame operations for data transformation: read_csv(), read_json(), merge(), groupby(), to_sql(). For medium-scale data work.
- **python-dotenv:** load environment variables from .env files for local development; call load_dotenv() at the start of the script.
- **logging:** built-in structured logging instead of print(); getLogger() and log levels (DEBUG, INFO, WARNING, ERROR).
- **dataclasses / pydantic:** type-safe data models. dataclass for simple structs, pydantic for validation with type enforcement, which is great for validating API response schemas.

## GCP Pub/Sub


### Q11. What is Pub/Sub and how does it work? Explain the publisher, subscriber, and topic model.

*Level: Core*

- Pub/Sub (Publish/Subscribe) is a messaging pattern where publishers send messages to a named channel called a topic, and subscribers receive messages from topics they are registered to.
- **The key characteristic:** publishers and subscribers are decoupled. The publisher does not know who receives the message. The subscriber does not know who sent it. They are connected only through the topic.
- **Topic:** a named channel for a specific type of event. Example: 'record-status-changed', 'new-client-created', 'billing-entry-added'. One topic can have many publishers and many subscribers.
- **Publisher:** The service that produces the event. When a record status changes in the platform — the record service publishes a message to the 'record-status-changed' topic.
- **Subscriber:** a service that receives messages from a topic. Multiple independent services can each subscribe to the same topic and process the event independently — fan-out pattern.
- **GCP Pub/Sub specifically:** managed service, at-least-once delivery, push or pull subscriptions, message acknowledgement required, messages retained for 7 days by default.

### Q12. What is the difference between GCP Pub/Sub and Apache Kafka? When would you choose one over the other?

*Level: Core*

- **GCP Pub/Sub:** fully managed, serverless. No cluster to manage. Automatic scaling. Google handles infrastructure, replication, and durability. Pay per message. Best for: GCP-native architectures, variable message volumes, teams that do not want to manage Kafka clusters.
- **Apache Kafka:** self-managed (or Confluent managed). You configure brokers, partitions, replication factor. Much more control and lower per-message cost at high volume. Best for: very high throughput (millions of messages/second), long retention (Kafka can retain messages for weeks/months), stream processing with Kafka Streams or Flink.
- **Key technical differences:** Kafka uses consumer groups and offsets — consumers track their own position in the log. Pub/Sub uses subscriptions — GCP manages delivery state. Kafka messages are ordered within a partition. Pub/Sub does not guarantee strict ordering by default.
- **Retention:** Kafka retains messages for a configurable period (days/weeks/months) — great for event sourcing and replay. Pub/Sub retains for 7 days maximum.

### Q13. How would you design an event-driven integration between a SaaS platform and a customer's practice-management system using Pub/Sub?

*Level: Advanced*

- **Event identification first:** which events need to flow? record.created, record.status_changed, record.closed, client.created, billing_entry.submitted. These are the topics.
- **Publisher design:** The integration service publishes events when tenant data changes. The tenant's practice management system publishes events when their data changes. Each system publishes to its own topics.
- **Subscriber design:** multiple services can subscribe to the same topic independently. When record.status_changed fires: (1) MDM service updates the golden record. (2) the AI assistant MCP server invalidates its cache. (3) Reporting service updates the dashboard. (4) The tenant's system receives the change back for reconciliation.
- **Message schema:** each message should have a consistent envelope — event_type, event_id (UUID for deduplication), timestamp, tenant_id (tenant isolation), payload (the actual changed data). Version the schema.
- **Idempotency:** subscribers must handle receiving the same message twice. Use the event_id to check if this message was already processed. At-least-once delivery means duplicates will occur.
- **Error handling:** failed processing should not acknowledge the message — Pub/Sub will redeliver. After N redeliveries, the message goes to a dead letter topic for investigation.

### Q14. What is at-least-once delivery in Pub/Sub and why is idempotency essential for subscribers?

*Level: Advanced*

- **At-least-once delivery:** Pub/Sub guarantees a message will be delivered at least once but may deliver it more than once. Duplicates can occur due to network issues, subscriber restarts, or acknowledgement timeouts.
- **Why duplicates happen:** subscriber receives the message, begins processing, but crashes before acknowledging. Pub/Sub redelivers after the acknowledgement timeout (configurable, default 10 seconds to 10 minutes).
- **Idempotent subscriber:** a subscriber that produces the same result regardless of how many times it processes the same message. Processing the same billing entry twice should not create two billing records.
- **Implementation approaches:** (1) Database unique constraint on event_id — INSERT fails on duplicate, subscriber handles the constraint violation gracefully. (2) Redis SET NX — set event_id in Redis only if not exists, skip if already set. (3) Lookup table — check processed_events table before processing.
- **Exactly-once delivery:** Pub/Sub does not guarantee this natively. If you need exactly-once — implement idempotent subscribers (simpler) or use Pub/Sub with Dataflow which provides exactly-once semantics.
- **Acknowledgement deadline:** set it long enough that your processing completes before Pub/Sub redelivers. If processing takes 30 seconds, set deadline to 60 seconds minimum.

### Q15. How would you use Pub/Sub to feed an AI assistant layer with real-time data updates from enterprise systems?

*Level: Advanced*

- **The problem:** The AI assistant's MCP server serves data to the LLM. If that data is stale (from yesterday's batch run), the AI gives outdated answers. The solution: Pub/Sub events trigger cache invalidation when data changes.
- **Architecture:** source system change -> Pub/Sub topic -> Cache Invalidation Service subscribes -> invalidates the relevant MCP server cache entry -> the next AI assistant query fetches fresh data.
- Event-driven cache invalidation is more efficient than TTL-based expiry: data is refreshed exactly when it changes, not on a fixed schedule. High-frequency data (record status) benefits most.
- **MCP server integration:** The MCP server's get_record_context tool checks a Redis cache first. When the Cache Invalidation Service receives a record-status-changed event, it deletes the Redis cache key for that record. Next tool call fetches fresh data from the source.
- The AI assistant-specific topics to subscribe to: record.status_changed (invalidate record context cache), client.updated (invalidate client profile cache), billing_entry.submitted (invalidate billing summary cache).
- **Latency benefit:** with event-driven invalidation, the AI assistant's data is fresh within seconds of a change in the source system, versus up to 24 hours with daily batch processing.

### Q16. What is a dead letter topic in Pub/Sub and when do you need one?

*Level: Core*

- **Dead letter topic:** a separate Pub/Sub topic where messages are automatically moved after failing to be acknowledged a maximum number of times (configurable, e.g. 5 attempts).
- When a message ends up in the dead letter topic: the subscriber consistently fails to process it — exception in processing code, downstream system unavailable, message is malformed, business rule violation.
- **Why you need one:** without a dead letter topic, a failing message blocks the subscription or gets lost. With a dead letter topic, the bad message is safely moved aside so other messages can continue processing. Alerts can be set on the dead letter topic.
- **What to do with dead letter messages:** (1) Monitor — alert on-call when dead letter count > 0. (2) Investigate — what caused the failure? Fix the root cause. (3) Replay — once fixed, re-publish the messages from dead letter back to the original topic.
- **Configuration in GCP:** set max_delivery_attempts on the subscription. Specify the dead_letter_topic. Ensure the Pub/Sub service account has publish permission on the dead letter topic.
- **In a platform context:** billing entry fails to process (record does not exist in MDM yet) -> goes to dead letter -> alert fires -> data team investigates -> MDM is updated -> message replayed successfully.

### Q17. How do you ensure message ordering in GCP Pub/Sub? When is ordering important?

*Level: Core*

- **Default Pub/Sub:** no ordering guarantee. Messages may be delivered out of order across different subscribers and even to the same subscriber.
- **When ordering matters:** record status transitions — if 'Active -> Under Review -> Closed' messages arrive out of order, the final state in your MDM may be wrong. Sequential financial transactions where order affects balances.
- **Enabling ordering:** set an ordering_key on published messages. All messages with the same ordering_key are delivered in order to a single subscriber. Example: use record_id as the ordering key — all events for the same record arrive in order.
- **Limitation:** ordering is guaranteed only within a single subscriber for the same ordering_key. If you have multiple subscriber instances, ordering is maintained per-instance per-key, not across all instances.
- **Subscriber must also have ordering enabled:** set enable_message_ordering=True on the subscription.
- **Alternative for strict ordering at scale:** use Kafka (guaranteed order within a partition) or design your system to be order-independent by including the full state in the message, not just the delta.

### Q18. How does Pub/Sub fit into an integration architecture compared to direct API calls? When would you choose Pub/Sub over calling an API directly?

*Level: Core*

- **Direct API call (synchronous):** Service A calls Service B's API and waits for a response. Tightly coupled — if B is slow or down, A is affected. Simple, immediate feedback, easy to implement.
- **Pub/Sub (asynchronous):** Service A publishes an event and continues immediately. Service B processes the event independently. Loosely coupled — B's availability does not affect A's operation.
- **Choose direct API when:** you need an immediate response to continue processing (create a record and need the record_id back). The operation must succeed before the next step. Low latency is critical.
- **Choose Pub/Sub when:** The producer does not need the result immediately. Multiple services need to react to the same event (fan-out). You want to buffer load spikes — events queue up during peak, processed at steady rate. Services should be independently deployable.
- **For the platform's integration layer:** source system -> integration service (direct API for synchronous operations like compliance checking). The platform -> downstream data pipeline -> the AI assistant cache (Pub/Sub for async event processing).
- **Hybrid approach (most production systems):** synchronous API for the core operation, Pub/Sub for side effects. Create a record via API (immediate), publish record.created event for MDM update, reporting refresh, the AI assistant cache warm-up (async).

### Q19. How do you monitor a Pub/Sub pipeline in production? What metrics matter?

*Level: Core*

- Subscription backlog (num_undelivered_messages): how many messages are waiting to be processed. If growing — your subscriber is too slow for the incoming rate. Alert if backlog > threshold for more than 5 minutes.
- **Oldest unacked message age:** how long the oldest unprocessed message has been waiting. If this is growing — a stuck message is blocking the queue. Alert if age > your SLA (e.g., > 5 minutes for real-time pipelines).
- **Message delivery latency:** time from publish to subscriber receiving. Should be seconds for normal operation. High latency = subscriber overloaded or network issue.
- **Dead letter topic message count:** any message in dead letter = a processing failure. Alert immediately on dead letter message count > 0.
- **Acknowledgement rate vs publish rate:** should be balanced. If ack rate < publish rate sustained over time — backlog is growing and subscriber needs to scale.
- **In GCP:** Cloud Monitoring dashboards for Pub/Sub metrics. Set alerting policies on the above metrics with notifications to PagerDuty or Slack.

### Q20. How would you use Pub/Sub in an event-driven Boomi integration? Can Boomi listen to Pub/Sub topics?

*Level: Advanced*

- Yes — Boomi has a GCP Pub/Sub connector. Boomi can both publish messages to Pub/Sub topics and subscribe to topics as an event-driven trigger for an integration process.
- **Boomi as subscriber:** configure a Boomi process with a GCP Pub/Sub listen operation as the start shape. The process fires automatically when a message arrives on the subscribed topic. No polling needed — truly event-driven.
- **Boomi as publisher:** at the end of a Boomi integration process, publish an event to Pub/Sub to notify downstream systems that the integration completed. Example: after syncing billing system data to MDM, publish 'mdm.sync.completed' event.
- **Use case for the platform:** Salesforce publishes a client.updated event to Pub/Sub. Boomi process listens to that topic and automatically triggers the Salesforce-to-MDM sync process. No scheduled polling — runs only when data actually changes.
- **Architecture benefit:** Boomi focuses on the integration transformation logic. Pub/Sub handles the event routing and buffering. Clean separation of concerns.
- **Alternative:** use Boomi's built-in API Service to expose a webhook endpoint, and configure Salesforce to call that webhook. Simpler for point-to-point; Pub/Sub is better for fan-out to multiple consumers.

## Boomi


### Q21. Walk through how you would design a Boomi integration process from Salesforce to an MDM platform. What shapes would you use?

*Level: Core*

- **Start shape:** either a Schedule (run every 15 minutes) or a GCP Pub/Sub Listen shape (event-driven — preferred for real-time). The choice depends on the latency requirement.
- **Connector shape (Salesforce):** Query operation using Salesforce connector. SOQL query to fetch recently modified records: SELECT Id, Name, Email FROM Account WHERE LastModifiedDate >= LAST_N_MINUTES:15.
- **Map shape:** transform the Salesforce data structure to the platform's canonical MDM schema. Field mappings: Salesforce.Id -> target_client.salesforce_id. Salesforce.Name -> target_client.client_name. Apply functions: TRIM(), UPPER(), date format conversions.
- **Decision shape:** route records based on conditions. If record exists in MDM (lookup) -> Update path. If new record -> Insert path. If validation fails -> Error handling path.
- **Data Process shape:** for operations on lists — splitting large result sets into individual records for processing, combining records, or sorting.
- **Connector shape (MDM API):** HTTP Client connector with PATCH operation (upsert). Pass the mapped record as JSON body. Set Authorization header with OAuth token from Connection.
- **Exception shape and Notify shape:** on error — log the failed record with details, send alert notification, continue processing remaining records (do not fail the entire batch for one bad record).

### Q22. How do you handle errors in a Boomi process? What is the difference between the Try/Catch shape and the Exception shape?

*Level: Core*

- **Exception shape:** placed at the process level. When any unhandled error occurs anywhere in the process, execution jumps to the Exception shape. Good for: process-level error logging, sending alert notifications, writing failed documents to a disk or queue for manual review.
- **Try/Catch shape:** wraps a specific section of the process. Errors in the Try block jump to the Catch block. After the Catch block, the process continues. Good for: handling errors in one step without failing the entire process. Example: wrapping a single API call that might fail for some records.
- **For batch processing — Try/Catch per record:** wrap the processing of each individual record in a Try/Catch. If one record fails, log it, send it to a dead letter queue, and continue with the next record. Never fail the entire batch for one bad record.
- **Error document handling:** in the Exception/Catch shape, the failed document is available. Log: the document content, the error message, the shape that failed, a correlation_id. Then decide: retry, dead letter, or alert.
- **Notify shape:** sends email or notification when called. Connect to the Exception shape to alert the operations team when a process fails.
- **Process reporting:** all errors appear in Boomi's Manage tab under Process Reporting. Filter by status=Error, drill into a specific execution to see the error shape, the error message, and the document that failed.

### Q23. What is a Boomi Map and how do you handle complex transformations — concatenation, splitting, lookups, and conditional mapping?

*Level: Core*

- **Boomi Map:** The visual field mapping tool. Drag connections from source fields to target fields. For complex transformations — use Map Functions applied to the connection.
- **Concatenation:** use the Concatenate function. Connect FirstName and LastName to a Concatenate function, then to the FullName target field. Set the separator (space, comma, etc.).
- **Splitting:** use the Split function. Connect a FullAddress field to a Split function with separator comma, and route to AddressLine1, City, State outputs.
- **Cross-reference lookup:** The most powerful Map function. Example: Salesforce sends a 2-letter state code ('CA'), the platform needs the full state name ('California'). Create a cross-reference table in Boomi with the mappings. The lookup function replaces the code with the value at runtime.
- **Conditional mapping (if/else):** use the IfElse function. Example: if BillingType = 'Hourly', use HourlyRate field; if BillingType = 'Fixed', use FixedFee field. Routes to one target based on a condition.
- **Date format conversion:** use the Date/Time function. Convert Salesforce date format (2026-06-01T00:00:00.000Z) to the platform format (01/06/2026). Specify input and output formats.
- **Scripting:** for transformations too complex for built-in functions — use the Scripting function with Groovy or JavaScript. Last resort — prefer built-in functions for maintainability.

### Q24. What is a Boomi Connector and how do you build or configure one for a new system with its own REST API?

*Level: Core*

- **Boomi Connector:** a pre-built integration component for a specific system or protocol. Boomi provides 200+ connectors — Salesforce, SAP, ServiceNow, HTTP, Database, FTP, and more. Connectors encapsulate the authentication, connection, and operation logic.
- **HTTP Client connector:** The most flexible connector. Use for any REST API that does not have a dedicated Boomi connector. Configure: URL, authentication type (OAuth2, Basic, API Key), request/response format (JSON, XML).
- **Configuring for the platform's API:** create an HTTP Client connection with: Base URL = https://api.example.com, Auth Type = OAuth2 Client Credentials, Client ID and Secret from provider developer console (stored in encrypted Boomi Connection field).
- **Operation:** define the HTTP method, relative URL path, request profile (JSON structure), response profile (JSON structure). Boomi will auto-generate profiles from a sample JSON response.
- **Custom connector:** for proprietary systems without an HTTP API — Boomi provides the Connector SDK (Java) to build a fully custom connector. Rare — usually HTTP Client connector handles most cases.

### Q25. What standards, governance and reusable frameworks would you establish in a Boomi Center of Excellence?

*Level: Expert*

- **Naming conventions:** standardised process names (SYSTEM-SYSTEM-ENTITY-OPERATION format. Example: SF-MDM-CLIENT-SYNC), component names, folder structures. Ensures any team member can understand what a process does from its name alone.
- **Reusable process components:** build shared sub-processes for common operations — OAuth token refresh, error logging, notification sending, cross-reference lookups. Teams use these instead of reinventing per project.
- **Connection governance:** approved list of connectors and connection configurations. Teams cannot create new production connections without COE review. Prevents credential sprawl and unlicensed connector usage.
- **Error handling framework:** standardised Exception shape configuration used by all processes. Includes: logging format (structured JSON with correlation_id), DLQ destination, notification recipients. Applied consistently across all teams' work.
- **Development standards:** process design reviews before production deployment. Peer review checklist — error handling, idempotency, retry logic, logging, test cases. Documented in Boomi's Notes field on every process.
- **Performance standards:** process design guidelines — avoid loading entire datasets into memory, use Boomi cache for repeated lookups, set appropriate batch sizes for database operations.

### Q26. How does Boomi API Management work and how does it compare to APIGEE?

*Level: Advanced*

- **Boomi API Management:** an add-on module to the Boomi AtomSphere platform. Allows you to expose Boomi integration processes as managed APIs, add security policies (OAuth2, API key), rate limiting, and analytics — all within the Boomi platform.
- **How it works:** create an API Service component in Boomi that wraps a Boomi process. Boomi generates an API endpoint URL. Incoming requests hit the Boomi API gateway, which authenticates, rate-limits, and routes to the underlying process.
- **Boomi API vs APIGEE comparison:** Boomi API Management is tightly integrated with Boomi processes — simple to set up when the API logic is implemented in Boomi. APIGEE is a dedicated enterprise API gateway — more powerful for: complex policy enforcement, multi-backend routing, enterprise-grade analytics, non-Boomi backends.
- **When to use Boomi API Management:** exposing Boomi integration processes to external consumers. Internal APIs where Boomi is already the integration platform. Quick, simple API exposure without a separate API gateway.
- **When to use APIGEE (or Kong):** enterprise-wide API gateway managing APIs from multiple backends (Boomi, microservices, databases). Advanced policies — JWT validation, quota management per developer tier, monetisation. Centralised API portal with developer onboarding.

### Q27. How do you migrate a large number of integrations from one platform to Boomi? What is your approach?

*Level: Advanced*

- **Discovery phase:** inventory all existing integrations — source system, target system, data entity, frequency, volume, business criticality. Boomi-based migration is most tractable with a complete inventory. Tools: Boomi Assure, custom discovery scripts, interviews with teams.
- **Classification:** not all integrations are equal. Tier 1 (business-critical, high-volume) — migrate last, most risky. Tier 3 (low-volume, low-criticality) — migrate first, validate the process. Start with Tier 3 to build confidence and refine the migration process.
- **Migration approach per integration:** analyse existing process -> map to Boomi shapes -> build in Boomi -> test with production-equivalent data in TEST environment -> parallel run (old and new both active) -> validate outputs match -> cutover -> decommission old.
- **Parallel run is critical:** run old and new integration simultaneously, compare outputs. This catches subtle data transformation differences that testing alone misses. Duration: 1–2 weeks for low-risk, 4+ weeks for high-risk integrations.
- **Rollback plan:** never decommission the old integration until the Boomi version has been stable in production for 2+ weeks. Keep the old system runnable for at least 30 days post-cutover.
- **COE standardisation during migration:** migration is the opportunity to apply COE standards to everything. Every migrated integration gets: standardised error handling, correlation IDs, structured logging, performance benchmarking.

### Q28. How do you monitor and troubleshoot a failing Boomi integration in production? Walk through your diagnostic approach.

*Level: Core*

- **Boomi Manage -> Process Reporting:** first stop. Filter by Status=Error, Date=today. Click on a failed execution to see: which shape failed, the error message, the document that was being processed at the time of failure.
- **Document viewer:** in Process Reporting, view the actual data at the point of failure. Boomi stores document snapshots at each shape (if configured). This shows exactly what data the failed shape received.
- **Error message analysis:** most Boomi errors fall into categories. Connection refused / timeout: downstream system unavailable. JSON parse error: unexpected response format. Map error: source field value incompatible with target field type. Authentication error: expired token or wrong credentials.
- **Root cause categories:** bad data in source system (fix at source). Downstream system unavailable (wait for recovery or trigger retry). Transformation error (fix the map or add data validation). Configuration error (wrong URL, expired credentials). Schema change in source or target system.
- **Hotfix approach:** if a process is failing consistently due to a known bad data record — use Boomi's Replay feature to re-run specific failed documents after fixing the underlying issue, without rerunning the entire process.

### Q29. What is Boomi's Master Data Hub and how would you use it for an MDM requirement?

*Level: Advanced*

- **Boomi Master Data Hub (MDH):** a cloud-native MDM capability within the Boomi platform. It allows you to create Golden Record definitions, configure matching and survivorship rules, and manage master data stewardship — all within Boomi.
- **How it works:** connect source systems to Boomi MDH via Boomi integration processes. MDH receives records, runs matching (deterministic on key fields + probabilistic on name/address), applies survivorship rules to create a Golden Record, and publishes the Golden Record back to subscribed target systems.
- **For the platform's MDM:** configure MDH with a Client Universe (the master domain for client records). Connect Salesforce, deal CRM, and billing system as contributing sources. Define matching rules: exact match on TaxID (high confidence), fuzzy match on ClientName + Jurisdiction (lower confidence, requires stewardship review). Define survivorship: ClientName from document management system (authoritative for legal names), Email from CRM (most frequently updated).
- **Stewardship workflow:** MDH has a built-in stewardship portal. Low-confidence matches (below threshold) are queued for a data steward to review and approve the merge decision. High-confidence matches are auto-merged.
- **Alternative to standalone MDH:** if the platform already has its own MDM layer (which it does — the platform is the MDM system for tenant data), Boomi MDH may not be needed. Boomi is the integration orchestrator feeding the platform's MDM, not a replacement for it.

### Q30. How does Boomi handle security — OAuth2, credential storage, and SSL certificates?

*Level: Core*

- **Encrypted connections:** all credentials (passwords, API keys, OAuth client secrets) stored in Boomi are encrypted at rest using AES-256. They are never visible in plaintext after being saved, even to administrators.
- **OAuth2 support:** Boomi HTTP Client connector natively supports OAuth2 Client Credentials and Authorization Code flows. Configure once in the Connection component — Boomi handles token acquisition and automatic refresh when the token expires.
- **Environment-specific credentials:** Boomi Environments (Test, Staging, Production) can have different connection configurations. A process promoted from Test to Production automatically uses Production credentials — no manual change needed. Enforces separation of environments.
- **SSL certificates:** Boomi atoms can be configured with custom SSL certificates for mutual TLS (mTLS) connections. Import the certificate and private key into Boomi's certificate store. Referenced in the Connection component. Boomi handles the TLS handshake.
- **Atom security:** The Boomi Atom (the runtime process execution engine) runs on-premises or in cloud infrastructure you control. All data processed stays within your network boundary — Boomi Cloud only sees metadata, not the actual data payloads.
- **Audit logging:** all connection configuration changes, process deployments, and execution history are logged in Boomi's audit log. SOC 2 compliant. Important for enterprise customers with strict audit requirements.

## REST API design


### Q31. What are the core REST principles and HTTP methods? How do you decide which method to use?

*Level: Core*

- **REST principles:** stateless (each request contains all information needed), client-server separation, uniform interface (consistent URL and method conventions), resource-based (URLs identify resources, not actions).
- **GET:** retrieve a resource. Safe (no side effects) and idempotent. Never use GET for operations that change data. GET /records (list), GET /records/123 (single).
- **POST:** create a new resource. Not idempotent — calling twice creates two records. Use Idempotency-Key header to make POST safe to retry. POST /records (create new record).
- **PUT:** replace an entire resource. Idempotent. If you PUT the same data twice, the result is the same. PUT /records/123 (replace entire record).
- **PATCH:** partial update. Only send the fields you want to change. More efficient than PUT for small updates. PATCH /records/123 {status: 'Closed'}.
- **DELETE:** remove a resource. Idempotent — deleting an already-deleted resource returns 404 (or 204 on some APIs). DELETE /records/123.
- **For the platform's API:** GET for the AI assistant queries (read record data). POST for creating clients and records. PATCH for status updates (record closed, client updated). Never use GET with a body (some proxies strip it).

### Q32. What HTTP status codes should you use for: resource not found, validation error, rate limit, created successfully, async accepted, access denied by policy?

*Level: Core*

- 200 OK: standard success for GET and PATCH. The request succeeded and here is the data.
- 201 Created: new resource created via POST. Include Location header pointing to the new resource URL.
- 202 Accepted: request accepted for async processing (report generation, bulk import). Include a status_url for polling.
- 400 Bad Request: client sent invalid data — missing required field, wrong format, invalid value. Include specific error details explaining what was wrong.
- 403 Forbidden: authenticated but not authorised. For information barrier: use 403 not 404. Returning 404 for an information barrier breach reveals that the record exists — a confidentiality violation.
- 404 Not Found: resource genuinely does not exist. Record ID that was never created.
- 409 Conflict: duplicate resource — client tried to create a record that already exists.
- 422 Unprocessable Entity: request is well-formed but semantically invalid — billing rate is negative, closed date before open date.
- 429 Too Many Requests: rate limit exceeded. Always include Retry-After header.
- 500 Internal Server Error: server-side error. Never return 200 with an error in the body — it breaks monitoring and retry logic.

### Q33. How do you design pagination for an API that returns potentially thousands of records?

*Level: Advanced*

- Never return all records without pagination — a customer with 50,000 active records would crash the client and the server if all records were returned in one response.
- **Cursor-based pagination (preferred):** instead of offset/page numbers, return a cursor (an opaque token) that encodes the position in the result set. Client passes the cursor in the next request to get the next page. More efficient than offset for large datasets.
- **Offset pagination (simpler):** GET /records?offset=0&limit=20, then ?offset=20&limit=20. Simple but gets slower on large offsets — the database must scan all previous rows. Also prone to duplicates if records are inserted mid-pagination.
- **Response envelope for paginated lists:** always include: data (the array of records), pagination object (next_cursor or next_page, has_more boolean, total_count optionally), metadata (page_size, returned_count).
- **Limit:** enforce a maximum page size (e.g., 100 records). If client requests limit=1000, return limit=100. Document the maximum in the API spec.
- **Consistent ordering:** pagination only works correctly with a stable sort order. Always add ORDER BY to the underlying query. For records: default sort by opened_date DESC or by record_id for cursor pagination.

### Q34. How do you implement OAuth2 authentication for a REST API? Explain the token flow from a Boomi integration perspective.

*Level: Advanced*

- **OAuth2 Client Credentials flow:** designed for machine-to-machine (M2M) authentication — exactly what a Boomi integration process uses. No user is involved. Boomi presents its client_id and client_secret to get an access token.
- **Token request:** POST to the token endpoint with grant_type=client_credentials, client_id, client_secret, and requested scope. Response includes access_token (JWT), token_type=Bearer, expires_in (seconds).
- **Using the token:** include in every API request header as Authorization: Bearer {access_token}. The platform API validates the token signature using the public key from the JWKS endpoint.
- **Token expiry:** access tokens are short-lived (typically 15 minutes to 1 hour). Boomi HTTP Client connector automatically refreshes when the token expires — configured in the OAuth2 connection settings.
- **JWT validation on the server side:** The platform's API gateway (APIGEE or internal) validates: signature (RSA public key), expiry (exp claim), audience (aud claim must match the API), issuer (iss claim).
- **Scopes:** The token includes the scopes granted at token request time. The API checks scope on every endpoint — record:read to read records, billing:write to submit billing entries. Boomi's connection requests only the minimum scopes needed.

### Q35. How do you design a REST API that supports both real-time queries from an AI assistant and batch data exports for reporting?

*Level: Core*

- Two different use cases with very different requirements: the AI assistant queries are interactive (sub-second response, single entity lookup). Batch exports are offline (minutes, thousands of records, file format).
- **Real-time API:** GET /v1/records/{id} for single entity. GET /v1/records with filtering and pagination for small lists. Response: JSON, immediate, maximum 100 records per call. Optimised with caching (Redis) and efficient database indexes.
- **Batch export API:** async pattern — POST /v1/exports to request a batch export. Returns a job_id immediately (202 Accepted). Client polls GET /v1/exports/{job_id}/status. When complete, GET /v1/exports/{job_id}/download returns a file URL (S3 pre-signed URL for large files).
- **File format for batch:** CSV or JSON Lines (one JSON object per line). JSON Lines is preferable — streaming parseable, does not require loading entire file into memory.
- **Rate limiting differentiation:** real-time queries: 1000 req/minute. Batch export requests: 10 per hour (expensive operations). Different limits enforced at the API gateway.
- **Caching strategy:** real-time GET endpoints can be cached (Redis, 60-second TTL for record data). Batch export jobs should not be cached — they are point-in-time snapshots.

### Q36. What is API versioning and how would you handle breaking changes in a REST API?

*Level: Advanced*

- **Breaking change:** any change that causes existing client code to fail. Removing a field, changing a field type, making an optional field required, changing a URL, changing HTTP method.
- **Non-breaking change:** adding new optional fields to responses (clients ignore unknown fields), adding new optional parameters, adding new endpoints.
- **URL versioning (recommended):** /v1/records, /v2/records. Clear, visible in logs and browser. Easy to route at the API gateway. Most common approach. Used by: Twitter, YouTube, Stripe (though Stripe uses header versioning).
- **Deprecation process:** (1) Announce deprecation 12 months in advance. (2) Add Deprecation and Sunset headers to v1 responses. (3) Track usage metrics per API key — which clients still use v1? (4) Reach out to tenants still using v1. (5) Return 410 Gone after the sunset date with a migration guide URL.
- Enterprise customers move slowly — 12-month deprecation minimum. For large enterprise customers, consider 18–24 months. Legal consequences of breaking a billing integration mid-year could be significant.
- **Compatibility layer:** during the transition, consider translating v1 requests to v2 internally at the gateway — clients can migrate at their own pace, infrastructure is running v2.

### Q37. What is idempotency in REST APIs and why is it critical for billing and payment APIs?

*Level: Core*

- **Idempotency:** making the same request multiple times has the same effect as making it once. GET is naturally idempotent (reading does not change state). PUT and DELETE are designed to be idempotent. POST is not — calling POST /billing-entries twice creates two billing entries.
- **Why critical for billing:** a network timeout causes the Boomi process to retry a POST billing-entry submission. Without idempotency, the user gets billed twice. This is a serious error — billing dispute, loss of client trust, potential legal liability.
- **Why critical for compliance checks:** if a compliance check request is submitted twice (network retry), it should only be checked once. Running the same compliance check twice and getting different results (due to data changes in between) could cause confusion.
- **Idempotency-Key header:** client generates a UUID before the first attempt. Sends it with every retry attempt. Server checks if this key was already processed — if yes, returns the original response without reprocessing.
- **Server-side storage:** store the idempotency key in Redis (24-hour TTL for fast lookup) and in the database (for durability). Redis for speed, database for persistence if Redis is flushed.
- **For Boomi integrations:** Boomi should generate an idempotency key (UUID) at the start of each integration run for POST operations. Store it in a Boomi document property. Include in every retry of the same operation.

### Q38. How do you document a REST API? What should a good API specification include?

*Level: Core*

- **OpenAPI Specification (OAS 3.0):** The industry standard for REST API documentation. Machine-readable YAML or JSON that describes all endpoints, parameters, request bodies, response schemas, authentication methods, and error codes. From this spec, tools auto-generate: client SDKs, server stubs, interactive documentation (Swagger UI, Redoc).
- **What every endpoint needs:** HTTP method and URL path. Path/query/header parameters with types, descriptions, required/optional. Request body schema (for POST/PATCH/PUT) with field descriptions and examples. Response schemas for each status code. Authentication requirement.
- **Error responses:** document every possible error code, not just the success response. A good API contract tells clients exactly what errors to expect and how to handle them.
- **Examples:** include realistic example request and response bodies. Not foo/bar examples — actual domain-relevant values (real record field names, realistic billing amounts).
- **Boomi-specific:** when building Boomi integrations, you use the OpenAPI spec to auto-generate the JSON profile for the HTTP Client connector. The spec is the source of truth for the Boomi connection configuration.

### Q39. How do you handle rate limiting in a Boomi integration that calls an API with strict rate limits (e.g., Salesforce: 100,000 calls/day)?

*Level: Advanced*

- **Understand the rate limit:** Salesforce Enterprise org typically has 100,000 API calls per 24-hour rolling window. All apps and integrations share this pool. Your Boomi integration is one of potentially many consumers.
- **Track consumption:** check X-Sforce-Limit-Info response header from Salesforce API calls. It returns current usage and limit. Log this on every call — alert when above 80% of daily limit.
- **Optimise queries:** reduce API calls per operation. Use SOQL queries that return multiple fields in one call rather than multiple single-field queries. Use bulk API for large data sets (Bulk API has separate rate limits — 10,000 batches/day, not counted against REST API limit).
- **Scheduled vs real-time:** batch processes should run during off-peak hours (2am–6am EST). Real-time processes should be reserved for truly time-sensitive operations. Avoid polling on short intervals — use Salesforce Change Data Capture (CDC) or Platform Events to eliminate polling entirely.
- **Boomi rate limit handling:** configure Boomi HTTP Client to respect 429 (Too Many Requests) responses. Implement retry with exponential backoff in the Exception shape — wait 60 seconds, retry. Do not hammer a rate-limited API.

### Q40. How would you design the REST API layer between Boomi and an AI platform? What endpoints does the AI layer need?

*Level: Advanced*

- The AI assistant queries tenant data via MCP tools. Those MCP tools call the platform's internal REST APIs. The API layer between MCP and the underlying data is what you are designing here.
- **Entity APIs the AI assistant needs:** GET /v1/records/{id} (get full record context for AI query). GET /v1/clients/{id} (client profile for relationship intelligence). GET /v1/compliance/check (compliance check for new engagement). GET /v1/billing/records/{id}/summary (billing overview for a record).
- **Design for LLM consumption:** API responses for the AI assistant should be designed for AI readability — include context-rich descriptions, not just raw IDs. Return: record_name (not just record_id), user_name (not just user_id), human-readable status descriptions.
- **Semantic fields:** add summary fields to responses. Example: add a billing_status_description field: 'Record has 15 unpaid invoices totalling $45,000' — this is more useful to an LLM than a raw list of invoice records.
- **Pagination and size:** The AI assistant's context window is limited. API responses should be bounded — never return unbounded lists. Default limit=10, maximum limit=50. MCP tools handle pagination if more records needed.
- **Caching:** The AI assistant queries the same data repeatedly during a conversation. Cache API responses at the MCP layer (Redis, 60-second TTL). Include cache-control headers in the API response to guide caching behaviour.

## Model Context Protocol (MCP)


### Q41. What is MCP, why was it created, and how does it differ from traditional API integration?

*Level: Core*

- **MCP (Model Context Protocol):** an open standard created by Anthropic in November 2024. It defines a standardised way for AI models to connect to external tools, data sources, and services.
- **Why it was created:** before MCP, every AI integration required custom code to connect a model to an external system. MCP standardises the interface — any MCP-compatible client (Claude, Cursor, other AI tools) can use any MCP server without custom integration code.
- **Traditional API integration:** developer writes code to call an API, parse the response, and pass it to the LLM as text in the prompt. Custom per integration. Tight coupling between the AI model logic and the data retrieval logic.
- **MCP:** The MCP server defines what data and tools are available (as Resources and Tools). The AI client discovers them automatically. The LLM decides when to call which tool — the developer does not hardcode which data to fetch.
- **The key shift:** from 'developer decides what context to give the AI' to 'AI decides what context it needs and retrieves it'. More flexible, more powerful, but requires careful design of what to expose.
- Platform context: the AI assistant uses MCP to connect to tenant data. The MCP server exposes record data, client data, compliance check results as Tools and Resources. The AI assistant autonomously retrieves the context it needs to answer user queries.

### Q42. Explain MCP Tools, Resources, and Prompts. Give a concrete example of each for an enterprise AI assistant platform.

*Level: Core*

- **Tools:** callable functions the AI invokes to perform actions or retrieve dynamic data. The AI model decides when to call them based on the conversation context. Tools are for: querying live data, performing calculations, triggering actions.
- **Resources:** read-only data sources identified by URI. Like files or database views. The model reads them for reference information that does not change frequently. Resources are for: schemas, policies, documentation, configuration.
- **Prompts:** reusable prompt templates with parameters. Surfaced to the user (not invoked autonomously by the model). The user selects a prompt template, fills in parameters, and the template pre-populates context for a common task.
- **For the AI assistant:** Tools = live data queries (get_record_status, check_compliance, search_billing). Resources = tenant data schema, AI usage policy document, integration mapping definitions. Prompts = 'draft engagement letter', 'summarise record for client update', 'prepare billing summary'.
- **Key distinction — who initiates:** Tools are invoked by the AI model autonomously. Resources are read by the model when referenced. Prompts are selected by the user as starting points.
- **Design principle:** expose Tools for data that changes (record status, billing entries). Expose Resources for data that is relatively stable (schema definitions, tenant policies). Use Prompts for recurring multi-step tasks users do repeatedly.

### Q43. How would you build an MCP server in Python that exposes business record data? Walk through the implementation.

*Level: Advanced*

- **MCP Python SDK:** Anthropic provides the 'mcp' Python library. pip install mcp. The SDK handles the MCP protocol communication — you implement the tool and resource handlers.
- **Server structure:** create an MCP server instance, register tool handlers with @server.call_tool() decorator, register resource handlers with @server.read_resource() decorator. Run with stdio transport for local, SSE transport for remote.
- **Tool handler:** receives the tool name and arguments, executes the business logic (database query, API call), returns a structured response. Must handle errors gracefully — return an error response, not raise an exception that crashes the server.
- **Information barrier enforcement:** every tool handler that returns tenant data must check RecordAccess before returning anything. This is not optional — it is the security boundary for user-client privilege.
- **Schema definition:** use Pydantic models or JSON Schema to define the input parameters for each tool. The MCP client uses this schema to know what arguments to pass.
- **Testing:** test each tool handler as a regular Python function before wrapping in the MCP server. Unit test the business logic. Integration test the full MCP server with a test client.

### Q44. How do you secure an MCP server deployed for enterprise use? What authentication and authorisation controls are needed?

*Level: Advanced*

- **Authentication:** The MCP host (Claude Desktop or the AI assistant) establishes the MCP connection. Enterprise deployment uses SSE transport over HTTPS. The MCP server validates the session using a JWT or API key passed in the connection headers.
- **Tenant isolation:** every MCP connection must be scoped to a specific tenant. The tenant_id is extracted from the authenticated session and passed to every tool call. No tool handler should ever query data without the tenant_id as a mandatory filter.
- **Per-user authorisation:** even within a tenant, different users have different access. A junior user should not see a restricted client relationship. The MCP server passes the user identity to the tool handler, which checks the user's permissions against the data being requested.
- **Information barrier enforcement:** The most critical security control for customer deployments. Before any record data is returned, check the RecordAccess table for the requesting user. If an information barrier applies — return a restriction message, never the data.
- **Audit logging:** every tool call must be logged with: user_id, tenant_id, tool_name, parameters (PII-masked), timestamp, access_granted (true/false), data_summary. Immutable audit log. Required for customer regulatory compliance.
- **Secret management:** The MCP server's credentials (database connection string, API keys) stored in AWS Secrets Manager or similar. Never hardcoded. Rotate regularly.

### Q45. How do you handle context window management in an MCP server? When does too much context hurt AI performance?

*Level: Advanced*

- **Context window:** LLMs have a maximum token limit for the combined input (conversation history + retrieved context + system prompt). Claude Sonnet 4.6: 200K tokens. Sounds large but large Salesforce data sets fill it quickly.
- **Too much context causes:** The model focuses on the most recent/prominent content, ignoring earlier context. Relevant information gets buried. Model response quality decreases. Latency increases (more tokens to process). Cost increases.
- **MCP tool response design:** every tool should return the minimum useful context, not everything available. get_record_context should return a concise summary (300-500 tokens), not the full record with all 200 billing entries.
- **Relevance filtering:** when returning a list (search results, billing entries), return the most relevant 5–10 items, not all 500. Include a count: 'Showing 10 of 247 billing entries. Use filters to narrow results.'
- **Pagination in tools:** design tools to accept a limit and cursor parameter. The AI can request more if needed. Never return unbounded results from an MCP tool.
- **Context hierarchy:** system prompt (most important) -> retrieved tool results (second) -> conversation history (third). Design your tool responses to be dense with relevant information, not padded with irrelevant fields.

### Q46. How do you test an MCP server before deploying it to production?

*Level: Core*

- **Unit test tool handlers:** test each tool handler function independently as a regular Python function. Do not need the full MCP server running. Test with realistic inputs including edge cases (missing fields, invalid IDs, empty results, information barrier scenarios).
- **MCP Inspector:** Anthropic provides a CLI tool (npx @modelcontextprotocol/inspector) that connects to an MCP server and lets you invoke tools and read resources interactively. Use this for manual testing before integrating with Claude.
- **Integration test:** use Claude Desktop with the MCP server configured. Test realistic conversation scenarios. Ask questions that should trigger specific tools. Verify: the right tool is called, the response is accurate, information barrier blocks work.
- **Information barrier test suite:** create a set of canonical test cases where specific users MUST NOT have access to specific records. Run automatically before every deployment. A deployment that breaks information barrier enforcement should never reach production.
- **Load test:** for a multi-tenant enterprise deployment — simulate concurrent MCP connections from multiple tenants. Verify tenant isolation holds under load. Check that one tenant's slow query does not block another tenant's connection.
- **Monitoring in production:** log every tool call. Alert on error rate > 5% for any tool. Alert on P99 latency > 3 seconds. Alert immediately on any information barrier bypass attempt.

### Q47. How would you design MCP tools that connect Boomi integration data to an AI assistant? What tools would expose integration status and pipeline health?

*Level: Advanced*

- **Tool:** get_integration_status(integration_name) — returns the current health, last run time, last run result, and error count for a specific Boomi integration. The AI assistant can answer: 'Is the Salesforce sync running? When did it last succeed?'
- **Tool:** get_pipeline_errors(tenant_id, from_date) — returns recent integration failures for a tenant. The AI assistant surfaces: 'The billing system billing sync has failed 3 times in the last hour — here is the error detail.'
- **Tool:** trigger_integration_retry(integration_name, failed_execution_id) — allows the AI assistant to suggest and potentially trigger a retry of a failed integration. Human approval required before execution (agentic workflow with human-in-the-loop).
- **Resource:** platform://integrations/catalog — exposes the full list of active integrations for a tenant, their schedules, and their SLAs. The AI assistant can reference this without a tool call.
- This is the foundation of an incident management agent — the AI assistant monitors integration health and proactively surfaces issues to the right person.

### Q48. What is context engineering and how does it improve MCP server performance?

*Level: Core*

- **Context engineering:** The discipline of designing what information an AI model receives — what to include, what to exclude, in what order, and in what format. Goes beyond writing prompts to managing the entire information environment.
- **Applied to MCP:** context engineering determines which tools and resources to expose, how to structure tool responses, what to include in the system prompt, and how to prioritise information within the context window.
- **Tool response design (context engineering):** a poorly engineered tool response dumps raw JSON with 50 fields. A well-engineered response returns a natural language summary with the 5 most relevant pieces of information. The difference in AI output quality is significant.
- **System prompt context engineering:** The MCP server's system prompt sets the model's role and constraints. For the AI assistant: 'You are an AI assistant for [Tenant Name] with access to their record management system. You must respect information barriers...' This context shapes every response.
- **Project context files:** structured files that define project context, coding standards and data models. Applied to MCP: a structured context document for each tenant that defines domain terminology and AI usage policies, loaded as a Resource.
- **Impact of good context engineering:** The same AI model and tools, but with well-engineered context, produce significantly better answers.

### Q49. How does MCP compare to LangChain tool use? When would you choose one over the other for an enterprise AI platform?

*Level: Advanced*

- **LangChain tool use:** Python-based framework. Tools defined as Python functions with @tool decorator. Tools live in the application code — tightly coupled to the Python application that uses them. Works with any LLM that supports function calling.
- **MCP:** a transport-level protocol. Tools defined in a separate server process (any language). The AI host discovers tools at connection time. Decoupled from application code. Standardised — works across multiple AI clients (Claude, VS Code Copilot, etc.).
- **Key difference:** coupling. LangChain tools are in-process (same Python app). MCP tools are out-of-process (separate server, communicates over stdio or SSE). MCP is like a microservice for AI context.
- **Choose LangChain when:** building a standalone Python AI application with a fixed set of tools. Need complex multi-step orchestration (LangGraph). The tools are tightly coupled to the application logic and do not need to be shared.
- Practical choice for an enterprise AI platform: MCP for the AI assistant platform layer (shared, multi-tenant, multi-team). LangChain or direct SDK for one-off AI automation scripts within a specific Python pipeline.

### Q50. CAPSTONE: How would you design a complete MCP architecture for an enterprise AI platform, covering Python, Pub/Sub, Boomi, REST API and MCP?

*Level: Expert*

- This is the question that ties everything together. Python + Pub/Sub + Boomi + REST API + MCP. Walk through the full architecture end-to-end.
- **Data layer (Boomi + REST API):** Boomi integration processes sync data from source systems (billing system, document management system, Salesforce, deal CRM) into the platform's canonical data model via REST APIs. Event-driven using GCP Pub/Sub — Boomi publishes events when syncs complete. Data quality validation at every sync step.
- **Storage layer:** processed tenant data stored in Redshift (analytical queries, Tableau). Redis cache layer for frequently accessed entities (record status, client profiles, compliance status). S3 for documents and audit logs.
- **MCP server layer (Python):** Python-based MCP servers expose tenant data as Tools and Resources to the AI assistant. One MCP server instance per tenant or shared with tenant isolation. Tools: get_record_context, check_compliance, search_records, get_billing_summary. Resources: tenant schema, AI policy.
- **Event-driven cache invalidation (Pub/Sub):** when Boomi sync completes, Pub/Sub event triggers cache invalidation in Redis. The AI assistant's next query fetches fresh data. Latency: data in the AI assistant within seconds of a source system change.
- **Security throughout:** OAuth2 at the Boomi-to-REST-API layer. JWT authentication at the MCP connection layer. information barrier check in every MCP tool handler. Tenant isolation at every layer. Immutable audit log of all AI data access.
- **Monitoring:** Boomi process health in Splunk. Pub/Sub backlog and dead letter alerts in GCP Monitoring. MCP tool call latency and error rate in CloudWatch. The AI assistant AI quality metrics (acceptance rate, feedback scores).
