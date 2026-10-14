# Dell Boomi Process Shapes: Comprehensive Comparison Guide

Every shape, functional categories and head-to-head comparisons.

## 1. Master shape reference: all 23 shapes

Boomi's process canvas organizes shapes into four categories: Start, Execute, Logic and Connector.

### 1.1 Start shape

| Shape | What it does | When to use |
|---|---|---|
| Start | Entry point of every process. Configures the trigger: schedule, listener (real-time) or no data (manual/API). | Every process begins with a Start shape. Configure the data source connector and schedule here. |

### 1.2 Execute shapes (data manipulation)

| Shape | What it does | When to use |
|---|---|---|
| Map | Transforms data from one profile/format to another using field mapping, functions, lookups and custom scripts. | Any integration moving data between systems with different schemas. The most frequently used shape. |
| Set Properties | Sets Dynamic Process Properties, Dynamic Document Properties or connector attributes (for example filename, email subject). | Store runtime values, pass context between shapes, set outbound connector parameters. |
| Message | Generates a free-form text document from static and dynamic input parameters. Replaces the current document. | Custom payloads, email bodies, log messages, dynamic API request bodies. |
| Notify | Writes custom messages to the process execution log and/or sends email/RSS alert notifications. | Debugging, audit logging, process monitoring. |
| Program Command | Executes OS-level commands or scripts (shell, batch, PowerShell) on the Atom's host. | External scripts, file operations, legacy system commands. Use sparingly: it creates an OS dependency. |
| Process Call | Invokes another Boomi process (subprocess) synchronously or asynchronously. | Reusable logic, modular design, error-handling isolation. The "function call" of Boomi. |
| Process Route | Routes the current execution to a different deployed process based on a configurable component. | Dynamic process selection at runtime; multi-tenant or configuration-driven architectures. |
| Find Changes | Tracks document changes (add/update/delete) by comparing current documents against a stored baseline. | CDC scenarios where you must detect new, modified or deleted records. |
| Add to Cache | Stores documents in a named Document Cache for later retrieval in the same or a sub-process execution. | Lookup/reference data caching, cross-reference between parent and sub-processes, join-like operations. |
| Retrieve From Cache | Retrieves documents from a previously populated Document Cache. | Access cached reference data during mapping; correlate documents from different sources. |
| Remove From Cache | Removes documents from a Document Cache. | Memory management, cache cleanup, prevent stale data. |
| Cleanse | Validates and standardizes data using data quality rules (format, pattern, lookup-based). | Data quality enforcement, address standardization, format validation before sending to the target. |
| Data Process | Multi-purpose shape for splitting, combining, sorting, custom scripting (Groovy/JS), zipping/unzipping documents. | Split batches into individual documents, combine documents, custom transformations, archive/compress files. |

### 1.3 Logic shapes (flow control and routing)

| Shape | What it does | When to use |
|---|---|---|
| Branch | Sends copies of each document down all paths. Every path executes unconditionally. | The same document must reach multiple destinations (for example CRM, data warehouse and logging). |
| Route | Routes documents down the first matching path based on evaluated conditions, with a default path. | Multi-path conditional logic (switch/case): route by document type, region, status or any field. |
| Decision | Binary true/false routing based on comparing two values. | Simple if/else checks: does the record exist? Is the amount above a threshold? Is a field empty? |
| Business Rules | Evaluates documents against multiple business rules to accept or reject them. | Multi-condition validation: 5+ rules checked together. |
| Exception | Terminates document processing and throws an error. Marks the document as failed. | Explicitly fail a document on an unrecoverable condition (for example a missing mandatory field). |
| Stop | Terminates the data flow in a process path without generating an error. | End a path when no further action is needed. |
| Return Documents | Returns documents to the calling source: a parent process (Process Call) or a web service client. | Subprocess return values, API response generation. Must be the last shape in a subprocess or listener path. |
| Flow Control | Configures how documents are processed: individually, in batches, in parallel, or throttled. | Control parallelism; throttle to stay under API rate limits. |
| Try/Catch | Captures process-level or document-level errors and routes failed documents to a Catch path. | Error handling, retry logic (0-5 retries), logging failures without killing the whole process. |

### 1.4 Connector shapes

| Shape | What it does | When to use |
|---|---|---|
| Connector | Gets data from or sends data to external systems using a Connection + Operation. | Every data input/output point: database queries, API calls, file reads/writes, SAP BAPI calls, Salesforce CRUD. |
| Trading Partner | Specialized connector for B2B/EDI exchange with trading partners over AS2, FTP or other protocols. | EDI: send/receive X12 and EDIFACT documents using B2B management profiles. |

## 2. Head-to-head comparisons

### 2.1 Branch vs. Route (the most common interview question)

| Aspect | Branch | Route | Takeaway |
|---|---|---|---|
| Logic type | AND: all paths execute | OR: first match wins | Branch = parallel fan-out; Route = conditional switch |
| Conditions | None, unconditional | Conditions per path + default | Branch has no conditions at all |
| Document flow | Copies the document to every path | Sends the document to one path only | A document hits all Branch paths but only one Route path |
| Path execution | Sequential top to bottom, each to completion | First matching condition wins | Branch executes all; Route evaluates and picks one |
| Default path | No concept of default | Yes: unmatched documents go here | Route always needs a default for safety |
| Use case | Send an order to SAP + CRM + DW | Route by region: EMEA vs. APAC vs. default | Branch = fan-out; Route = decision tree |
| Analogy | Photocopier: copies to all | Traffic signal: directs to one lane | Copies vs. directions |

### 2.2 Decision vs. Route

| Aspect | Decision | Route | Takeaway |
|---|---|---|---|
| Paths | Exactly 2: true and false | Multiple paths + default | Decision = if/else; Route = switch/case |
| Conditions | Single comparison of 2 values | Multiple conditions, one per path | Decision is simpler; Route handles complexity |
| Best for | Simple yes/no checks | Multi-way branching (3+ paths) | Decision for binary; Route for multi-path |
| Example | Is the order amount > $10K? | Route by order type: Standard / Rush / Return / Other | One question vs. categorization |
| Performance | Lightweight single comparison | Evaluates conditions sequentially | Decision is faster for simple checks |

### 2.3 Decision vs. Business Rules

| Aspect | Decision | Business Rules | Takeaway |
|---|---|---|---|
| Complexity | Single condition | Multiple rules evaluated together | Decision = one check; Business Rules = validation suite |
| Output | True / False paths | Accept / Reject paths | Similar binary outcome, different framing |
| Rules | One comparison | Multiple rules with AND/OR logic | Business Rules replaces chains of Decisions |
| Profile aware | Compares any two values | Works with the document profile structure | Business Rules understands your schema |
| Best for | Does the customer exist? | amount > 0 AND status = Active AND region in (EMEA, APAC) AND date not null | Multi-field validation engine |
| Edition | All editions | Advanced Workflow (Pro/Enterprise) | Higher license tier |

### 2.4 Map vs. Message

| Aspect | Map | Message | Takeaway |
|---|---|---|---|
| Purpose | Transform between structured profiles | Generate free-form text | Schema-to-schema vs. template output |
| Input/output | Source profile to target profile | Parameters to a free-form text document | Map preserves structure; Message creates new text |
| Profiles | Requires source and target profiles | None needed | Message is freeform |
| Functions | Built-in functions + custom scripting | Static text + parameter substitution | Map has rich transformation |
| Use case | SAP IDoc to a JSON API | Build an email body or dynamic API URL | Data transformation vs. text generation |
| Document effect | Replaces the document with the transformed version | Replaces the document with the generated message | Both replace the current document |

### 2.5 Set Properties vs. Map (for setting values)

| Aspect | Set Properties | Map | Takeaway |
|---|---|---|---|
| Primary purpose | Store metadata / context values | Transform document data | Variables vs. data transformation |
| Where values go | Process/document properties | Target profile fields | Properties are metadata; Map output is the document |
| Document impact | Does not modify the document content | Replaces the document with transformed output | Set Properties leaves the document unchanged |
| Use case | Store a filename, batch ID, counter | Convert an XML customer to a JSON customer | Context vs. transformation |
| Persistence | Properties persist through the process | The transformed document flows forward | Properties are reusable |

### 2.6 Notify vs. Message

| Aspect | Notify | Message | Takeaway |
|---|---|---|---|
| Purpose | Write to the process log / send alerts | Generate a new document from parameters | Logging/alerting vs. document creation |
| Document impact | Does not change the document | Replaces the current document | Notify passes through; Message overwrites |
| Output | Log entry + optional email/RSS alert | New text document in the flow | Side effect vs. transformation |
| Use case | Log "Step 3 complete, 100 records processed" | Build a custom JSON payload for an API call | Observability vs. payload construction |
| Debugging | Primary debugging tool in Boomi | Not typically used for debugging | Use Notify generously in dev |

### 2.7 Process Call vs. Process Route

| Aspect | Process Call | Process Route | Takeaway |
|---|---|---|---|
| Mechanism | Calls a specific named subprocess | Routes execution to a dynamically selected process | Static call vs. dynamic dispatch |
| Target | Fixed at design time | Determined at runtime by configuration | Process Call knows the target; Process Route resolves it |
| Use case | Call a shared error-handling subprocess | Multi-tenant: route to a tenant-specific process | Reusable modules vs. dynamic routing |
| Return | Can return documents to the parent | Execution transfers: no return | Process Call can be synchronous; Process Route is a hand-off |
| Edition | All editions | Advanced Workflow (Pro/Enterprise) | Higher license tier |

### 2.8 Try/Catch vs. Exception

| Aspect | Try/Catch | Exception | Takeaway |
|---|---|---|---|
| Purpose | Capture and handle errors gracefully | Deliberately throw an error | Catches errors vs. creates them |
| Error flow | Failed documents go to the Catch path for recovery | Immediately terminates document processing | Safety net vs. kill switch |
| Retry | Built-in retry (0-5 attempts) | No retry: immediate failure | Automatic retry vs. final |
| Use case | Wrap API calls to handle timeouts | Reject a document that fails mandatory validation | Recoverable vs. fatal |
| Scope | Process-level or document-level errors | Only the current document | Try/Catch is broader |
| Combined use | The Catch path can contain Exception shapes | An Exception inside a Try triggers the Catch path | Exception triggers; Try/Catch handles |

### 2.9 Stop vs. Exception

| Aspect | Stop | Exception | Takeaway |
|---|---|---|---|
| Behavior | Silently ends the processing path | Throws an error and marks the document failed | Graceful end vs. error termination |
| Process status | Process completes successfully | Process may show errors/warnings | Stop doesn't affect success status |
| Logging | No error generated | Error logged in process execution | Stop is invisible; Exception is visible |
| Use case | End a path after routing (nothing more to do) | Reject invalid data and alert operations | Expected dead-ends vs. real failures |

### 2.10 Flow Control vs. Branch (parallel processing)

| Aspect | Flow Control | Branch | Takeaway |
|---|---|---|---|
| Parallelism type | Same documents processed in parallel threads | Document copies sent to different paths | Thread parallelism vs. path parallelism |
| Purpose | Control how documents are processed | Execute multiple processing pipelines | Execution mode vs. pipeline fan-out |
| Use case | Process 1000 records with 10 threads | Send the same order to SAP + Salesforce + audit log | Throughput vs. multi-destination |
| Options | Run individually, batch count, parallel threads | Add as many paths as needed | Execution modes vs. paths |

## 3. Shapes grouped by purpose

- **Data transformation:** Map, Message, Data Process (custom script), Cleanse. Map for structured profile-to-profile; Message for free-form text; Data Process scripts (Groovy/JavaScript) beyond Map's reach; Cleanse for data quality.
- **Routing and branching:** Branch, Route, Decision, Business Rules, Process Route. Decision for binary checks, Route for multi-path, Business Rules for multi-condition validation, Branch for unconditional fan-out, Process Route for dynamic dispatch.
- **Error handling:** Try/Catch, Exception, Stop. Every production process should have at least one Try/Catch.
- **Process flow and execution:** Flow Control, Process Call, Return Documents, Start.
- **Caching:** Add to Cache, Retrieve From Cache, Remove From Cache. They share a Document Cache component.
- **Monitoring and logging:** Notify, Set Properties. Set Properties does not appear in logs unless you also use Notify.
- **External interaction:** Connector, Trading Partner, Program Command.
- **Change tracking:** Find Changes compares current documents with a stored baseline and routes them to Add, Update or Delete paths.

## 4. Scenario quick-fire: which shape?

| Scenario | Shape and why |
|---|---|
| Send the same document to 3 systems at once | Branch: unconditional parallel fan-out |
| A document may be an order, return or credit memo, each processed differently | Route: conditional routing on document type |
| Check whether a customer exists before inserting | Decision: binary true/false |
| An order must pass 5 validation rules | Business Rules: multi-condition validation with Accept/Reject paths |
| Transform SAP IDoc XML into REST JSON | Map |
| Build a dynamic email body with order details | Message |
| Log "Step 3 complete: 150 records processed" | Notify: logs without modifying the document |
| Store the current batch ID for later use | Set Properties: Dynamic Process Property |
| An API call may time out; retry 3 times | Try/Catch with retry count 3 and a Catch path |
| A document is missing a mandatory field and should be rejected | Exception |
| Process 2000 records with 10 parallel threads | Flow Control |
| A reusable error-handling routine called from 5 processes | Process Call |
| Detect records added, updated or deleted since the last sync | Find Changes |
| Reference data needed during mapping without repeated DB calls | Add to Cache, then Retrieve From Cache |
| A subprocess returns data to the parent | Return Documents |
| Split a 500-record batch file into individual documents | Data Process (Split Documents) |
| After routing valid documents, the invalid path has nothing to do | Stop |
| Run a shell script on the Atom server to trigger a legacy system | Program Command |
| Send an EDI 850 purchase order to a trading partner via AS2 | Trading Partner |
| Pick the target process at runtime from configuration | Process Route |

## 5. Best practices for enterprise projects

1. Every production process has at least one Try/Catch around the critical logic. Never fail silently.
2. Use Decision for single binary checks; move to Business Rules when you chain 3+ Decisions.
3. Use Branch sparingly. Each path gets a full copy of every document, which raises memory use. For large payloads consider sequential processing with Process Call.
4. Give every shape a display name. Unnamed shapes show as generic types in Process Reporting.
5. Consolidate Set Properties shapes. Five properties in one shape beat five shapes.
6. Don't use Exception and Notify together redundantly. An Exception is already logged.
7. Use Flow Control's parallel processing judiciously. Too many threads can overwhelm downstream systems or exceed API rate limits.
8. Use Process Call for any logic that appears in more than one process.
9. Use Data Process for operations that fit no other shape: splitting, combining, sorting, zipping, Groovy/JavaScript scripting.
10. Cache reference data with Add to Cache at the start of a process rather than repeating connector lookups during mapping.
11. Route shapes always have a default path. Unmatched documents are logged and handled, never dropped.
12. In Map, prefer built-in functions over custom Groovy scripts. Reserve scripting for logic the visual mapper cannot express.
