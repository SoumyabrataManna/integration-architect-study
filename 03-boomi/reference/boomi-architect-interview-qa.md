# Dell Boomi: Architect-Level Interview Q&A

Generic Q&A for senior consultant / architect roles.

## 1. Architecture and design

### Q1: Explain the integration layers in Boomi and their significance.

Boomi integrations map to enterprise architecture concerns:

- **Connectivity layer:** connectors communicate with source and target systems (SAP, Salesforce, databases, REST/SOAP APIs).
- **Data transformation layer:** Map shapes, functions and cross-reference tables transform data between formats and schemas.
- **Process orchestration layer:** the core flow: routing, branching, error handling, batching and business logic with Decision, Branch and Flow Control shapes.
- **API management layer:** Boomi API Management exposes processes as managed APIs with throttling, security policies and developer portals.
- **Governance and monitoring layer:** the AtomSphere dashboard, process logs, alerts and audit trails.

Layering gives modularity, reusable sub-processes and independent scaling of concerns.

### Q2: How do you design a scalable integration architecture in Boomi?

- **Sub-process decomposition:** reusable sub-processes, each with a single responsibility (fetch, transform, publish).
- **Stateless design:** avoid keeping state in process properties across executions; use a database or cache.
- **Molecule or Atom Cloud for scale:** a multi-node Molecule gives parallel processing and load balancing.
- **Async patterns:** use queues (JMS, Boomi Message Queue) to decouple producers and consumers and prevent backpressure.
- **Idempotency:** handle duplicate messages safely with unique keys and upsert logic at the target.
- **Environment promotion:** Packaged Components and deployment pipelines across Dev, Test and Prod.

Scalability decisions are client-specific: 10K records a day needs a different design from 10M.

### Q3: How do you choose between Atom, Molecule and Atom Cloud?

- **Atom:** single-node runtime. Low volume, simple integrations, dev/test. No HA.
- **Molecule:** multi-node, single-tenant runtime. HA, load balancing, parallel processing for high-volume workloads.
- **Atom Cloud:** multi-tenant, hosted by Boomi. Suits smaller scenarios without on-premises needs; less control over the runtime.

Decision criteria: sensitive data needing on-premises processing means Atom or Molecule; HA means Molecule; zero infrastructure management means Atom Cloud. A client with on-premises SAP usually needs a Molecule in its own data center for latency, security and volume.

### Q4: Best practices for deploying Boomi in a high-availability environment?

- **Molecule:** 2+ nodes on different servers or VMs to avoid a single point of failure.
- **Shared file system:** Molecule nodes need a shared NFS/SAN mount for the working data directory.
- **Load balancer:** NGINX, F5 or similar in front of API listener nodes.
- **Health monitoring:** alerts for Atom offline events in AtomSphere.
- **Retries:** automatic retries with backoff for transient failures.
- **Separate listener nodes:** dedicate nodes to inbound API/listener traffic versus scheduled batch jobs.

### Q5: How would you integrate Boomi with legacy systems?

- **File-based:** CSV or fixed-width files via the Disk connector with FTP/SFTP polling.
- **Database:** JDBC connectors when no API exists.
- **MQ/JMS:** IBM MQ or JMS queues through the JMS connector.
- **SOAP/Web Services:** WSDL-based services through the Web Services connector.
- **Custom scripting:** Groovy in a Data Process shape for proprietary formats.

## 2. Core platform and runtime

### Q6: What is an Atom Worker and how does it support parallel processing?

An Atom Worker is an execution thread within an Atom or Molecule that runs integration process instances.

- **Parallel execution:** multiple workers run concurrently, enabling parallel processing of documents.
- **Resource management:** bounded by the Atom's JVM memory and thread pool settings.
- **Fault tolerance:** if one worker fails, others continue.
- **Task queue:** workers take tasks from an internal queue, which spreads load across a Molecule's nodes.
- **Tuning:** adjust Max General Execution Threads and Max Low Priority Execution Threads for throughput versus stability.

### Q7: How does Boomi support real-time integration?

- **API Service processes:** a process deployed as a REST or SOAP endpoint runs synchronously per request.
- **Listeners:** HTTP/HTTPS Server, JMS, AMQP and AS2 listeners keep a process waiting for inbound messages.
- **Real-time connectors:** Salesforce outbound messaging, Kafka and JMS enable near-real-time flows.
- **Low latency design:** minimize transformation overhead; cache lookups (cross-reference tables, cached sub-processes).

Real-time suits order-to-fulfillment or event notification; batch suits nightly reconciliation. Knowing when not to use real-time shows judgment.

### Q8: Explain process routing and conditional branching.

- **Decision:** binary branch (true/false) on a single condition.
- **Route:** multi-path routing like switch/case.
- **Branch:** splits the flow into parallel paths, each processing the same document independently.

Example: in an SAP order integration, a Route shape sends Create, Update and Delete operations to separate downstream API calls.

## 3. Connectors, APIs and integration patterns

### Q9: Explain Boomi's connector architecture.

Connectors are reusable adapters that abstract protocol and authentication for an endpoint.

- **Two components:** Operation (what to do: query, create, update, delete) and Connection (where to connect: URL, credentials).
- **Types:** technology connectors (HTTP, JMS, Database), application connectors (SAP, Salesforce, Workday) and the Universal connector (custom APIs via an OpenAPI spec).
- **Import:** most connectors introspect the target schema and generate request/response profiles.
- **Connection pooling:** connectors manage connection lifecycle and reuse connections within an execution.

### Q10: How does Boomi differ from traditional ETL tools?

- **Cloud-native vs. server-based:** classic ETL (PowerCenter, DataStage) needs dedicated servers; Boomi runs in the cloud with lightweight on-premises Atoms.
- **Low-code vs. code-heavy:** visual drag-and-drop cuts development time.
- **Real-time vs. batch-only:** Boomi supports real-time, event-driven and API-driven patterns natively.
- **API-first:** APIs are first-class, with built-in API management.
- **Connectivity breadth:** many pre-built connectors for modern SaaS apps.

Boomi is not a replacement for heavy data warehouse ETL; Informatica IICS or Talend suit complex data quality, lineage and large batch transformation better.

### Q11: What are the SAP connector's requirements and limitations?

- **Configuration:** SAP JCo library, connection parameters (client ID, system number, hostname, logon group) and an RFC destination on the SAP side.
- **Operations:** BAPI calls (function modules), IDoc send/receive, ABAP proxy web services.
- **Limitations:** no native streaming of large IDocs, so chunk them; RFC calls are synchronous and can bottleneck high-volume scenarios.
- **Error handling:** BAPI errors come back in the payload (the RETURN table), not as HTTP errors. Explicitly check for E/A message types and route accordingly, for example a Decision shape after the BAPI call that inspects the RETURN TYPE field.

### Q12: Boomi API Management vs. APIGEE: when to use which?

- **Boomi API Management:** best when APIs are built on Boomi processes. Gateway and integration are co-located, with lower latency and simpler governance.
- **APIGEE:** a dedicated gateway with analytics, developer portals, monetization, OAuth 2.0, JWT validation and traffic shaping. Better for external partners and developers.
- **Combined pattern:** Boomi for backend integration (SAP, Salesforce, databases); APIGEE as the northbound gateway for external consumers.

Boomi-centric shop with mostly internal integrations: Boomi API Management reduces footprint. Building an API economy with external partners: APIGEE is the better investment.

## 4. Error handling and monitoring

### Q13: How is error handling managed in Boomi?

- **Try/Catch:** wraps risky operations; failed documents go to the Catch path.
- **Process-level behavior:** set what happens on error (stop, continue, or route to an error notification sub-process).
- **Connector error handling:** route HTTP 4xx/5xx to alternate paths.
- **Notify shape:** email or custom notifications with the error message and document context.
- **Manage tab:** all executions with status (Complete, Error, Warning) and document-level logs.
- **Retry:** resubmit failed documents manually or automatically from Process Reporting.

Describe a dead-letter pattern: failed documents, then an error notification, then persistence to a retry queue or table, then alerting to ops.

### Q14: How do you implement a dead-letter queue and compensating transactions?

Boomi has no native dead-letter queue, so build the pattern:

1. Wrap the integration in Try/Catch and route errors to an error sub-process.
2. Persist the failed payload, error message and timestamp to an `error_log` table with a Database connector.
3. Alert operations with the Notify shape or the Mail connector.
4. Run a scheduled process that queries unresolved failures and resubmits them (retry with backoff).
5. Make the target idempotent with upserts or idempotency keys.

For compensating transactions: if step 3 of 5 fails, call compensating operations in reverse order in the Catch path, for example rolling back records created upstream.

### Q15: How do you monitor Boomi processes in production?

- **AtomSphere dashboard:** real-time executions, errors and Atom health.
- **Process logs:** "Add to Process Log" shapes at key steps for tracing. Disable in production for performance unless troubleshooting.
- **External monitoring:** forward Atom logs to Splunk or ELK via log file monitoring or an HTTP listener.
- **AtomSphere API:** pull execution records, error counts and Atom status into your own dashboard.
- **Alerts:** Atom offline, process error and connector failure alerts in Notifications settings.

## 5. Security and governance

### Q16: How does Boomi handle data security and compliance?

- **In transit:** TLS 1.2+ through AtomSphere; connectors use the target's protocol (HTTPS, SFTP).
- **At rest:** the cloud platform encrypts stored data. For on-premises Atoms, local file system encryption is the client's responsibility.
- **RBAC:** developers can build but not deploy to production; operations can monitor but not modify.
- **Environment segregation:** separate Atoms for Dev, Test and Prod.
- **PGP:** built-in PGP encrypt/decrypt for file-based integrations.
- **Certifications:** SOC 2 Type II, ISO 27001, HIPAA-eligible, GDPR-compliant at platform level (verify current status).

### Q17: How do you manage credentials and secrets across environments?

- **Environment extensions:** different connection values (URLs, usernames, passwords) per environment without changing the process.
- **Encrypted fields:** passwords in Connection components are encrypted and masked.
- **External secrets managers:** HashiCorp Vault, AWS Secrets Manager or Azure Key Vault via a pre-process step that fetches credentials at runtime and sets them as dynamic document properties.
- **No hardcoding:** never store credentials in process properties or scripts.

Show awareness of least privilege and secret rotation.

### Q18: How do you govern a large Boomi environment with multiple teams?

- **Naming conventions:** for example `[Client]_[System]_[Direction]_[Object]`, with folders organized by domain.
- **Component reuse library:** shared sub-processes, maps and connections.
- **Role-based access:** developers see their own domain folder; an architect reviews cross-domain changes.
- **Packaged Components:** versioned deployments for rollback and traceability.
- **Change management:** CI/CD using the AtomSphere API, triggered from tickets or Git commits.
- **Documentation:** process-level notes and shape notes for non-obvious logic.

## 6. Consulting and assessment

### Q19: How do you approach a Boomi assessment for a new client?

A five-pillar framework:

1. **Current state inventory:** catalog processes, connectors and Atom configurations; find redundant or poorly designed integrations.
2. **Architecture review:** Atom topology (single Atom vs. Molecule), deployment patterns, environment segregation maturity.
3. **Governance review:** naming, reuse, RBAC and change management.
4. **Performance and reliability:** error rates, execution times, retry configuration, monitoring coverage.
5. **Roadmap:** quick wins (naming, error handling) and strategic moves (Molecule, CI/CD). Tie each recommendation to business value.

### Q20: How do you advise on Boomi API Management vs. APIGEE?

- **Boomi API Management** when the API layer mainly exposes Boomi-built integration processes internally: simpler, lower cost, one governance plane.
- **APIGEE** for external API monetization, developer self-service portals, complex OAuth flows or multi-cloud API governance.
- **Both** when Boomi handles internal integration and APIGEE handles the external API layer.

## 7. Quick reference

- **Atom:** lightweight single-node runtime that executes Boomi processes.
- **Molecule:** multi-node, single-tenant clustered runtime for HA and load balancing.
- **Atom Cloud:** multi-tenant hosted runtime managed by Boomi.
- **AtomSphere:** the cloud platform used to design, deploy and monitor integrations.
- **Connector:** adapter with a Connection (where) and an Operation (what).
- **Shape:** a visual element in a process (Map, Decision, Branch, Notify, Data Process).
- **Profile:** schema definition for a data format (XML, JSON, CSV, EDI) used in Maps.
- **Packaged Component:** versioned, deployable bundle for environment promotion.
- **Cross-reference table:** lookup table for static value mapping (for example SAP cost center to GL account).
- **Trading Partner:** Boomi's B2B/EDI module for partner configuration and document exchange.
- **Environment Extension:** environment-specific override of connection values for multi-environment deployment.
