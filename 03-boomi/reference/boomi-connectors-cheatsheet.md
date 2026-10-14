# Dell Boomi Connectors: Quick Reference

Functionality and use cases for the commonly used connectors.

## SAP connectors

| Connector | Functionality | When to use |
|---|---|---|
| SAP (RFC/BAPI) | Invokes SAP RFCs and BAPIs using JCo libraries. Synchronous request-response calls to SAP ECC/S4HANA. | Real-time transactional calls: posting sales orders (`BAPI_SALESORDER_CREATEFROMDAT2`), material lookups, customer master reads. Requires a JCo JAR upload. |
| SAP OData | Connects to SAP Gateway / S/4HANA OData v2/v4 services over HTTP/HTTPS. CRUD + query with `$filter`, `$expand`. | SAP Fiori-exposed APIs, S/4HANA Cloud integrations, SuccessFactors APIs. Preferred for cloud-to-cloud SAP connectivity. |
| SAP IDoc | Sends and receives IDocs asynchronously via JCo. Supports all standard IDoc types. | Async document exchange: MATMAS (material master), DEBMAS (customer master), ORDERS05 (purchase orders). Batch and event-driven flows. |

## Application connectors

| Connector | Functionality | When to use |
|---|---|---|
| Salesforce | Native CRM connector. CRUD, bulk operations, SOQL queries, real-time event listening. | CRM sync, lead/opportunity management, account master data. Bulk connector for large volumes. |
| NetSuite | Oracle NetSuite ERP via SuiteTalk SOAP or REST. Saved searches and record operations. | ERP data sync: orders, invoices, inventory, GL journal entries. |
| Workday | Workday HCM/Finance via Workday Web Services. Get, Put and custom reports. | HR data integration: onboarding, payroll sync, benefits enrollment, position management. |
| ServiceNow | ServiceNow REST Table API. CRUD on any table (incidents, CIs, change requests). | ITSM integration: incident creation from monitoring tools, CMDB sync, ticket workflows. |
| Microsoft Dynamics 365 | Dynamics 365 CE/FO via OData/REST. Entity operations and batch requests. | CRM/ERP integration in the Microsoft ecosystem: customer, pipeline and financial data. |

## Database connectors

| Connector | Functionality | When to use |
|---|---|---|
| Database | JDBC connector for Oracle, SQL Server, MySQL, PostgreSQL, SAP HANA, DB2 and more. SQL queries, stored procedures, batch operations. | Direct DB reads/writes, ETL jobs, data migration, stored procedure execution. JDBC driver upload for non-standard DBs. |
| Disk (flat file) | Reads/writes files on the local Atom server disk. Any format (CSV, XML, JSON, fixed-width). | Legacy file-based integrations, processing drop files from SFTP, local batch file processing. |

## Communication protocol connectors

| Connector | Functionality | When to use |
|---|---|---|
| HTTP Client | Generic HTTP/HTTPS for REST. GET, POST, PUT, PATCH, DELETE with custom headers and auth. | Any REST API without a native connector; webhooks, third-party SaaS APIs, microservice calls. |
| Web Services (SOAP) | Consumes SOAP/WSDL services. WSDL import generates request/response profiles. | Legacy SOAP integrations, SAP PI/PO services, banking/insurance SOAP APIs, government/B2B services. |
| SFTP | Secure file transfer over SSH. Upload, download, list, delete, move. | B2B file exchange, EDI file drops, batch transfers, partner files. |
| FTP | Standard FTP, active/passive mode. | Legacy file transfer. Use SFTP when possible. |
| AS2 | Secure B2B document exchange with MDN receipts and encryption. | EDI/B2B trading partner integration: retail supply chain (EDI 850, 856, 810), healthcare (X12 837/835). |
| Mail (SMTP/POP3/IMAP) | Send and receive email with attachments, HTML body, mailbox polling. | Notifications on integration events, inbound email attachments, alerting. |
| JMS | Java Message Service for queues and topics. | Enterprise messaging: IBM MQ, ActiveMQ, TIBCO EMS. Async message-driven architectures. |

## Cloud and middleware connectors

| Connector | Functionality | When to use |
|---|---|---|
| Amazon S3 | CRUD on S3 buckets and objects. | Cloud file storage, data lake ingestion, archiving processed documents. |
| Amazon SQS | Send/receive on SQS queues, standard and FIFO. | Event-driven architectures on AWS, decoupling microservices, async triggers. |
| Amazon SNS | Publish to SNS topics for fan-out. | Broadcast events to multiple subscribers, push and multi-channel alerts. |
| Amazon Redshift | Redshift via JDBC for SQL queries and bulk loads. | Analytics loading, warehouse ETL, reporting sync. |
| Amazon EventBridge | Publish and consume on the EventBridge bus. | AWS event-driven integration, cross-account routing, SaaS events. |
| Azure Service Bus | Send/receive on Service Bus queues and topics. | Azure messaging, hybrid cloud patterns, ESB replacement. |
| Azure Blob Storage | CRUD on Blob containers and blobs. | Azure file storage, document pipelines, data lake integration. |
| Google Cloud Storage | CRUD on GCP Storage buckets. | GCP file storage, cross-cloud data movement. |
| Google BigQuery | Run queries and load data. | GCP analytics, warehouse loading, large-scale analysis. |
| Kafka | Produce and consume Kafka topics. Avro, JSON and string serialization. | Real-time streaming, event sourcing, CDC pipelines, high-throughput messaging. |
| MQ Series (IBM MQ) | Native IBM MQ connector. Queue get/put and topic pub/sub. | Mainframe and legacy middleware, banking/insurance queues. |
| RabbitMQ | AMQP messaging for exchanges, queues and routing. | Microservice communication, task queuing, lightweight messaging. |

## API management and event connectors

| Connector | Functionality | When to use |
|---|---|---|
| API (Web Server) | Exposes Boomi processes as REST/SOAP APIs, with authentication, rate limiting and custom endpoints. | Publishing integration services as APIs, webhook receivers, API facades over legacy systems. |
| Event Streams | Boomi's native pub/sub within the Boomi ecosystem. | Intra-Boomi event-driven flows, process-to-process async communication. |
| Flow Services | Connects Boomi Integration with Boomi Flow for workflow and human tasks. | Approval workflows, human-in-the-loop processes. |

## Data format and EDI connectors

| Connector | Functionality | When to use |
|---|---|---|
| EDI (X12/EDIFACT) | Processes X12 (US) and EDIFACT (international) with translation, validation and acknowledgments. | B2B EDI: purchase orders (850), invoices (810), ASN (856), healthcare claims (837). |
| HL7 | HL7 v2.x over MLLP. | Healthcare integration: ADT, ORM, ORU between EHR/EMR, lab systems and hospital middleware. |
| FHIR | FHIR REST APIs for modern healthcare data exchange. | Patient records, clinical data, payer integrations, SMART on FHIR apps. |

## Productivity and collaboration

| Connector | Functionality | When to use |
|---|---|---|
| Google Workspace | Google Sheets, Drive, Calendar and Contacts APIs. | Syncing CRM data to Sheets, report distribution, calendar-triggered workflows. |
| Microsoft 365 / SharePoint | SharePoint Online, OneDrive, Office 365. | Document management, SharePoint list sync, Teams notifications, user provisioning. |
| Slack | Messages and interaction with Slack channels and users. | Integration alerts, notifications, ChatOps triggers. |

## Master data and governance

| Connector | Functionality | When to use |
|---|---|---|
| MDM (Master Data Hub) | Boomi MDM golden record management: match, merge, quarantine, CRUD. | Customer, product and vendor mastering; golden record distribution. |
| Data Catalog & Prep | Boomi DataHub data quality profiling and preparation. | Quality checks before integration, profiling source data, cleansing. |

## Trading partner and B2B

| Connector | Functionality | When to use |
|---|---|---|
| Trading Partner | Manages partner profiles, communication channels (AS2, SFTP, HTTP) and document routing. | Multi-partner B2B, EDI partner onboarding, dynamic routing on partner config. |
| Managed File Transfer (MFT) | Enterprise file transfer with auditing, scheduling, encryption and retry logic. | Regulated transfers, SOX-compliant exchange, large batch files with guaranteed delivery. |

## Connector selection tips

- Prefer native connectors (Salesforce, NetSuite, Workday) over generic HTTP when available. They handle pagination, auth and error mapping.
- For SAP: RFC/BAPI for real time, IDoc for async/batch, OData for S/4HANA Cloud.
- Use the Database connector with JDBC drivers for direct DB access (Oracle, SQL Server, HANA, Postgres, MySQL).
- For event-driven patterns, prefer Kafka or SQS/SNS over polling for scalability and near-real-time processing.
- Expose integration processes as APIs with the API (Web Server) connector, paired with API Management for rate limiting, security and analytics.
- For B2B/EDI, combine Trading Partner + AS2 + EDI connectors for a complete onboarding and exchange solution.
