# Repository Summary: PartsUnlimitedMRP
---
## Overview
Repository Overview
- Purpose: PartsUnlimitedMRP is a Manufacturing Resource Planning (MRP) solution that runs the core order-to-fulfillment workflow for parts-based manufacturing: maintaining a product catalog and inventory view, generating quotes, converting quotes into orders, tracking orders through shipment, and synchronizing with external systems and channels.
- Business problem solved: It provides a single, reliable system of record and integration hub for parts inventory, quoting, ordering, and shipment tracking. It keeps catalogs current, processes customer orders efficiently, coordinates with suppliers/partners, and exposes operational status—reducing manual effort, latency, and errors across supply chain processes.

Architecture
- Two runtime components:
  - Ordering service (smpl.ordering.*): A Spring Boot microservice (WAR-capable) exposing REST APIs for catalog, dealers, quotes, orders, shipments, and delivery confirmations. It persists to MongoDB (production) or an in-memory store (tests/dev), and instruments requests with Application Insights.
  - Headless integration worker (integration.*): A Spring-based daemon running scheduled jobs. It pushes product/inventory updates to Azure Storage Queues and pulls customer orders from queues into the ordering service via REST.
- Integration fabric (integration.services.*):
  - REST adapter (MrpConnectService) for synchronous calls to the MRP/ordering APIs.
  - Queue layer (QueueFactory, QueueService) for typed, durable messaging with Azure Storage Queues, including JSON serialization, visibility timeouts, and poison-message cleanup.
- Data and domain layers:
  - Domain DTOs for ordering (smpl.ordering.models.*) and persistence mappers for MongoDB (smpl.ordering.repositories.mongodb.models.*).
  - Website-facing DTOs (integration.models.website.*) to decouple public payloads from internal domain types.
- Persistence abstraction (smpl.ordering.repositories.*):
  - Repository interfaces for catalog, dealers, quotes, orders, shipments, and a RepositoryFactory to switch backends (mock vs. MongoDB).
  - Concrete adapters for MongoDB (smpl.ordering.repositories.mongodb.*) and in-memory mocks (smpl.ordering.repositories.mock.*).
- Cross-cutting infrastructure:
  - Configuration for databases, queues, endpoints (smpl.ordering configuration classes; integration.infrastructure.*).
  - Web filters for telemetry and CORS.
  - Tests for backend conformance and integration flows (mock.test and mongodb.test).

Key Functionalities
- Catalog and inventory master data:
  - CRUD/upsert for parts with SKU indexing, pricing, inventory, lead-time logic (effective lead time when inventory is available).
- Dealer (partner) management:
  - CRUD/upsert and queries to maintain dealer master data used by quotes and orders.
- Quote lifecycle:
  - Create, read, update, delete; aggregate pricing context, dealer/customer/address; line items; validation; ID generation.
- Order lifecycle:
  - Create from quote (prevent quote reuse), status management via events, queries by dealer, status, and quote link; audit trail via OrderEventInfo.
- Shipment lifecycle and delivery:
  - One shipment per order rule; create, update, append events, query by order status; aggregate Delivery view (quote + order + shipment) and confirmed deliveries.
- Integration/synchronization:
  - Scheduled product feed: periodically fetch catalog updates and publish ProductMessage batches to Azure queues.
  - Scheduled order intake: read OrderMessage items from Azure queues, create orders in the MRP service via REST, and delete processed messages (at-least-once semantics).
  - Typed queue operations with JSON serialization, visibility timeouts, poison-message handling, and create-if-not-exists queues.
- Observability and operations:
  - Application Insights telemetry for HTTP requests and Mongo operations; structured logging across services.
  - Health/status endpoints, build metadata, and permissive CORS for multi-origin access.
- Configurability and portability:
  - Externalized settings for MongoDB, Azure Storage, queue names/timeouts, and service endpoints; dev/test/prod flexibility via properties.

Domain Alignment
- Manufacturing and supply chain focus:
  - End-to-end flow from catalog availability through quoting to orders and shipments mirrors real manufacturing operations.
  - Inventory/lead-time projection in catalog items supports planning and customer expectations.
  - Order and shipment event histories provide auditability crucial for fulfillment and logistics tracking.
  - Supplier/dealer management underpins B2B scenarios typical in manufacturing distribution networks.
- Integration patterns tuned for operations:
  - Queue-based decoupling manages spikes and supports resilience across supplier portals and web channels.
  - Scheduled polling with at-least-once delivery and idempotent endpoints matches the reliability needs of distributed supply chains.
  - DTO/anti-corruption layers stabilize APIs for the website and external partners without leaking internal models.

Package Interactions
- Ordering service request path:
  - HTTP requests hit smpl.ordering.controllers.* which validate, orchestrate, and log telemetry, then delegate to repositories via RepositoryFactory.
  - Repositories persist and query through Mongo adapters (smpl.ordering.repositories.mongodb.*) that map between domain DTOs (smpl.ordering.models.*) and Mongo document models (smpl.ordering.repositories.mongodb.models.*).
- Integration worker path:
  - Scheduled tasks (integration.scheduled.*) use MrpConnectService to call the ordering service REST APIs and QueueService to interact with Azure queues.
  - QueueService relies on QueueFactory for durable queue clients and on integration.models.* for typed message handling (QueueResponse and website DTOs).
  - Configuration comes from integration.infrastructure.* (for queues and endpoints), ensuring environment-specific wiring.
- Data and model boundaries:
  - Website-facing payloads (integration.models.website.*) encapsulate product availability and order submission schemas for external channels.
  - Domain DTOs (smpl.ordering.models.*) remain internal to the ordering service for consistent persistence, validation, and event recording.
- Testing and parity:
  - The same repository contracts are exercised against mock and Mongo backends (smpl.ordering.repositories.mock.test.* and smpl.ordering.repositories.mongodb.test.*), ensuring behavioral consistency and reducing integration risk.

Executive Summary
PartsUnlimitedMRP is a pragmatic, production-ready MRP building block that combines an API-first ordering service with a headless integration engine. It manages core supply chain data (catalog, dealers), executes the quote→order→shipment lifecycle with auditability, and keeps external channels synchronized via REST and Azure Storage Queues. The architecture uses clear separations—controllers, domain models, repositories, persistence mappers, and integration services—backed by robust configuration and observability. It aligns tightly with manufacturing workflows, favoring resilience (at-least-once, scheduled polling), portability (pluggable backends), and maintainability (DTOs, mapping layers, contract tests) to support reliable operations across procurement, inventory, order fulfillment, and shipment coordination.
## Statistics
- **Total Packages**: 15
- **Total Files**: 99

---
## Package Summaries
### 1. Package: `integration`
**Files**: 2

Package-level summary:

1) Overall purpose and role
- The integration package provides the headless runtime that continuously connects the Parts Unlimited MRP system with external systems (e.g., supplier portals, catalogs, or order/shipment status providers). It runs as a lightweight Spring-based worker that schedules and executes background jobs to keep product data synchronized and to process orders automatically, enabling reliable inventory control, order fulfillment, and supplier coordination without manual triggers or cron scripts.

2) How the files work together
- Main.java boots a Spring application context and activates the schedulers for the integration tasks (e.g., CreateOrderProcessTask and UpdateProductProcessTask). It also keeps the JVM alive so these background processes run continuously.
- Constants.java exposes a single, shared polling interval (30 seconds) that the scheduled tasks use to run at a consistent cadence. By centralizing this value, all integration jobs stay aligned on timing, simplifying tuning and ensuring predictable synchronization.
- Together, Main acts as the orchestrator/daemon, while Constants enforces uniform scheduling behavior across tasks. The tasks themselves (referenced by Main) handle the domain-specific integrations such as pushing new orders to suppliers and syncing product data inbound/outbound.

3) Key functionalities provided
- Bootstrapping of a long-running integration service via Spring, with automatic startup of scheduled jobs.
- Centralized, reusable scheduling interval for all periodic integration activities.
- Continuous execution of core integration workflows:
  - Order creation/processing against external systems.
  - Product data synchronization (updates, refreshes).
- Reliability primitives for integrations through repeated, predictable polling rather than one-off or manual runs.

4) Notable patterns and architectural decisions
- Headless worker/daemon pattern: a dedicated process specialized for background integration tasks.
- Spring-based scheduling and dependency injection: tasks are registered and managed in the application context, enabling clean lifecycle control and loose coupling.
- Separation of concerns: Main.java handles process lifecycle and scheduler activation; task classes encapsulate integration logic; Constants.java supplies shared configuration.
- Utility-constants pattern: a non-instantiable constants class prevents magic numbers and enforces consistent timing across jobs.
- Polling integration style: favors simple, resilient synchronization cycles at a fixed cadence, supporting predictable operations in manufacturing and supply chain flows.

In short, the integration package is the always-on synchronization engine of the repository, ensuring MRP data stays in lockstep with external partners and systems through scheduled, centrally timed background jobs.

### 2. Package: `integration.services`
**Files**: 3

Package-level summary:

1) Overall purpose and role
- The integration.services package is the integration fabric for Parts Unlimited MRP. It provides both synchronous (REST) and asynchronous (Azure Storage Queue) connectivity so the web front end, back-end MRP services, and external partners can exchange data reliably.
- It enables end-to-end order flows (quote → order → shipment) while also decoupling longer-running or cross-system activities (procurement events, inventory updates, supplier notifications, shipment updates) via typed, durable message queues.
- The package improves scalability, resilience, and observability in manufacturing and supply chain workflows by centralizing communication concerns and hiding infrastructure details behind simple, type-safe services.

2) How the files work together
- QueueFactory creates and caches Azure CloudQueue clients. It ensures a queue exists and exposes a thread-safe, reusable handle for subsequent operations.
- QueueService depends on QueueFactory (and configuration) to:
  - Put typed messages on queues (serialize to JSON).
  - Retrieve messages with visibility timeouts, deserialize them into target types, and return a combined raw/typed response.
  - Delete messages, including automatic cleanup of poison messages when deserialization fails, with structured logging.
- MrpConnectService acts as a REST adapter using Spring’s RestTemplate to orchestrate synchronous interactions with MRP services:
  - Creates quotes, converts them to orders, and triggers shipments for website-originated requests.
  - Retrieves catalog data needed by ordering flows.
- Together, these services let the application choose the right integration mode for each use case:
  - Synchronous REST for immediate user-facing operations (MrpConnectService).
  - Asynchronous queues for downstream or cross-system processes that benefit from decoupling and retry semantics (QueueService + QueueFactory).

3) Key functionalities provided
- Typed, reliable messaging over Azure Storage Queues:
  - Enqueue/dequeue/delete operations with JSON serialization/deserialization.
  - Configurable visibility timeouts.
  - Poison-message detection and cleanup to prevent stuck workflows.
  - Centralized queue provisioning and client reuse with create-if-not-exists.
- REST-based orchestration of core MRP workflows:
  - Quote creation, order creation, shipment initiation.
  - Product catalog retrieval.
  - Centralized HTTP handling and logging for operational traceability.
- Configuration-driven connectivity and structured SLF4J logging across services for consistent observability.

4) Notable patterns and architectural decisions
- Adapter/Facade pattern: MrpConnectService encapsulates multiple REST endpoints behind a simple API for the web tier, reducing coupling to MRP service contracts.
- Factory + Registry pattern: QueueFactory lazily creates and caches CloudQueue instances in a thread-safe manner, optimizing resource usage and startup time.
- Message-driven integration: QueueService provides a type-safe, JSON-based conduit for asynchronous workflows, enabling elastic scaling and decoupled processing.
- Separation of concerns:
  - QueueFactory handles queue discovery and lifecycle.
  - QueueService handles message serialization, retrieval, and error handling.
  - MrpConnectService focuses on synchronous orchestration and business flow.
- Reliability and operability choices:
  - Create-if-not-exists for queues to avoid provisioning drift.
  - Visibility timeouts to handle competing consumers and retries.
  - Poison-message deletion with structured logging to prevent repeated failures and aid diagnostics.
- Configuration-first design: Externalizing queue names, connection details, and timeouts allows environment-specific tuning without code changes.

In summary, integration.services unifies REST and queue-based integrations into a cohesive layer that enables robust, scalable manufacturing and supply chain operations, balancing immediate user-driven actions with decoupled, asynchronous processing.

### 3. Package: `integration.models`
**Files**: 1

Package-level summary:

1) Overall purpose and role
- The integration.models package defines lightweight, type-safe model objects that sit at the boundary between infrastructure (e.g., Azure Storage Queues) and business logic for Parts Unlimited’s manufacturing and supply chain integrations. Its models capture the data and metadata required to reliably move work through the integration pipeline without coupling business processing to transport-specific concerns.
- By centralizing these integration-layer models, the package promotes consistent handling of queue-driven events (order updates, shipment notifications, supplier feeds) across services and workflows, improving robustness, clarity, and operability.

2) How the files work together
- Currently represented by QueueResponse, the package provides a single, well-scoped handoff object used across the pipeline:
  - Ingest/deserialization component reads a CloudQueueMessage from Azure Storage Queues, parses its payload into a strongly typed object T, and constructs a QueueResponse<T>.
  - Business processors/handlers receive QueueResponse<T>, operate primarily on the typed payload, and still retain access to the original queue message for operational decisions (acknowledge, retry, update visibility).
  - Completion/retry logic (often in a queue client or orchestration layer) uses the CloudQueueMessage reference carried by QueueResponse to perform delete, extend-visibility, or re-enqueue actions consistent with processing outcomes.
- This pattern keeps transport concerns available where needed without leaking them throughout business code, enabling clear separation of responsibilities across the integration flow.

3) Key functionalities provided
- Typed payload binding to infrastructure messages: couples a CloudQueueMessage with a generic payload T so downstream code works with domain-level data while preserving message metadata.
- Reliable message lifecycle control: maintains the original queue message reference to support completion, retry, visibility timeout updates, and failure handling after processing.
- Immutability and simplicity: exposes minimal getters over an immutable container, reducing side effects and making it safe for asynchronous and concurrent processing.
- Consistent handoff semantics: standardizes how queue messages and parsed content are passed between ingestion, processing, and acknowledgment components across multiple MRP workflows.

4) Notable patterns and architectural decisions
- Value-object wrapper with generics: QueueResponse<T> is a generic, immutable value object that cleanly bridges infrastructure and domain payloads without sacrificing type safety.
- Separation of concerns: business logic focuses on the typed payload, while message lifecycle operations remain explicit and localized via the preserved CloudQueueMessage.
- Message-driven integration: the package supports an event/queue-based architecture, where reliability is achieved through explicit acknowledgment, retries, and visibility management.
- Pipeline handoff pattern: models act as stable handoff contracts between stages (deserialize → process → ack/retry), enabling composability, testability, and clearer failure handling paths.

In sum, integration.models establishes the model layer for queue-centric integrations, with QueueResponse providing the core abstraction that keeps message metadata and lifecycle control co-located with the parsed, strongly typed payload used by business services. This improves reliability and maintainability across supply chain integration workflows.

### 4. Package: `integration.infrastructure`
**Files**: 2

Package-level summary: integration.infrastructure

1) Overall purpose and role
- This package provides the configuration backbone for the integration layer of the Parts Unlimited MRP system. It centralizes how the application discovers and reads environment-specific settings required to connect to external services (e.g., Azure Storage queues and the MRP service endpoint).
- By externalizing and standardizing access to these settings, it enables reliable deployments across dev/test/prod, and ensures integration workflows (order processing, inventory updates, shipment coordination) can consistently locate endpoints, credentials, and timeouts.

2) How the files work together
- ConfigurationHelpers is the low-level utility that loads key-value pairs from classpath properties files into a shared Properties store. It offers simple getters (string/int with defaults) and exposes the underlying Properties object.
- ConfigurationManager sits on top as a narrow, typed façade that defines and exposes the specific configuration values the integration layer needs (Azure Storage connection string, queue names, queue timeouts, MRP endpoint). It delegates all retrieval to ConfigurationHelpers.
- Other integration components depend only on ConfigurationManager for configuration, keeping business logic decoupled from file formats and property key names. ConfigurationHelpers remains the single place that knows how settings are loaded and parsed.

3) Key functionalities provided
- Centralized properties loading:
  - Reads classpath-based properties files into a shared store for the integration layer.
- Typed configuration access:
  - Static getters for critical settings: Azure Storage connection string, order and inventory queue names, queue message/visibility timeout, and the MRP service endpoint.
- Defaulting and tolerance:
  - String and integer retrieval with fallback defaults to reduce hard failures at runtime.
- Unified configuration surface:
  - One place to manage property keys and environment-specific settings, reducing duplication and drift across the codebase.

4) Notable patterns and architectural decisions
- Façade over configuration:
  - ConfigurationManager provides a small, intention-revealing API over generic Properties, preventing key-name scattering and improving readability.
- Separation of concerns:
  - ConfigurationHelpers handles IO/parsing; ConfigurationManager defines domain-specific configuration needs for integration workflows.
- Static, globally accessible configuration:
  - Simple to use and low ceremony, but introduces global state characteristics that can impact testability and flexibility compared to dependency injection.
- Pragmatic error handling with trade-offs:
  - Defaults and swallowed exceptions favor resilience but risk masking misconfigurations; lack of stream closing may cause resource leaks.
- Environment-driven design:
  - Properties-based externalization supports deploying the same code to multiple environments with different settings, crucial for integration with external systems.

In sum, integration.infrastructure acts as the integration layer’s configuration hub, offering a clean, stable API (ConfigurationManager) atop a simple properties loading mechanism (ConfigurationHelpers) to reliably wire the MRP system to Azure queues and the MRP service across environments.

### 5. Package: `integration.scheduled`
**Files**: 2

Package-level summary: integration.scheduled

1) Overall purpose and role
- This package hosts scheduled integration jobs that keep the Parts Unlimited MRP ecosystem in sync with external systems via Azure queues and the MRP REST API. It automates two critical, opposing data flows:
  - Outbound catalog/inventory updates from the MRP service into the system’s messaging layer
  - Inbound customer orders from the messaging layer into the MRP backend
- By running on a schedule and using queues for decoupling, it supports timely, resilient, and scalable cross-system synchronization without tight coupling between services.

2) How the files work together
- UpdateProductProcessTask periodically calls the MRP service (via MrpConnectService) to fetch CatalogItem updates, batches them into a ProductMessage, and publishes to the Azure inventory queue (via QueueService). This populates the messaging layer with fresh product/inventory state for downstream consumers (web, order/quote processing, etc.).
- CreateOrderProcessTask periodically reads OrderMessage instances from the Azure order queue (via QueueService), then creates corresponding orders in the MRP system (via MrpConnectService.createNewOrder). After successful creation, it deletes the message.
- Together, they form a bidirectional bridge:
  - MRP → Azure queue (catalog updates)
  - Azure queue → MRP (orders)
- Both tasks share common integration services (MrpConnectService, QueueService, ConfigurationManager) and rely on Spring scheduling and SLF4J logging, but operate independently so that catalog updates and order intake do not block each other.

3) Key functionalities provided
- Scheduled polling of external systems and queues
  - Fetches CatalogItem updates from MRP
  - Drains OrderMessage items from Azure queues
- Message production and consumption
  - Batches catalog updates into a single ProductMessage for efficient publication
  - Iteratively processes order messages to drive order creation
- Reliability and operational behavior
  - UpdateProductProcessTask logs and swallows exceptions to avoid disrupting the scheduler and accepts temporary staleness when failures occur
  - CreateOrderProcessTask aborts the current run on error to maintain at-least-once delivery semantics (potential duplicates if failure occurs after order creation but before message deletion)
- Stateless, per-run execution
  - Constructs required services on each invocation and performs no-ops when there is nothing to process
- Config-driven integration
  - Uses ConfigurationManager to resolve endpoints and queue names for portability across environments
- Observability
  - SLF4J logging around each step for traceability and operational troubleshooting

4) Notable patterns and architectural decisions
- Asynchronous, queue-based integration
  - Azure queues decouple producers and consumers, smoothing spikes, enabling backpressure, and improving resilience
- Scheduled polling adapters
  - Each class acts as a scheduled adapter: one pushes domain updates to the queue; the other pulls commands from the queue to the MRP API
- At-least-once processing for commands
  - The order flow intentionally favors reliability over exactly-once semantics; downstream idempotency is implied/required to handle duplicates
- Best-effort, eventually consistent updates
  - Product updates are pushed in batches and failures are tolerated per run, leading to eventual consistency without blocking other operations
- Separation of concerns
  - Clear split between catalog (read/update propagation) and orders (command processing), akin to CQRS tendencies
- Fault isolation
  - Independent tasks and exception strategies ensure one integration path’s issues don’t cascade to the other
- Simplicity of lifecycle
  - Stateless tasks and per-run service instantiation reduce lifecycle complexity and shared state risks

In summary, integration.scheduled provides the time-driven, decoupled connectors that keep catalog data flowing out of the MRP system and orders flowing into it. It does so via Azure queues, a small set of shared integration services, and deliberate reliability choices that favor resilience and operational safety in a distributed manufacturing and supply chain environment.

### 6. Package: `integration.models.website`
**Files**: 4

Package-level summary: integration.models.website

1) Overall purpose and role
- This package defines the website-facing data contracts (DTOs) used to move product and order information between the web tier and the backend MRP domain/services. It acts as an anti-corruption layer that shields internal domain models from external exposure, stabilizes REST payloads, and standardizes how the website sends orders and receives catalog availability from the MRP system.

2) How the files work together
- Product flow (MRP -> Website):
  - Core MRP code supplies CatalogItem objects.
  - ProductItem maps CatalogItem fields (skuNumber, inventory, leadTime) into a lightweight, transport-safe DTO.
  - ProductMessage wraps a list of ProductItem to form a response envelope for catalog listings, searches, and availability feeds exposed to the website or external clients.
- Order flow (Website -> MRP):
  - The website submits an OrderMessage as the request body for order-related endpoints.
  - OrderMessage contains customer/dealer/contact/shipping details, order date, financials, and a list of OrderItem entries.
  - Each OrderItem ties the request back to the MRP catalog via SKU and price, enabling inventory checks, quote/totals calculation, fulfillment, and shipment coordination in downstream services.

3) Key functionalities provided
- Stable JSON payloads for REST:
  - JavaBean-style getters/setters and no-arg constructors ensure Jackson-friendly serialization/deserialization.
- Product availability projection:
  - ProductItem exposes SKU, inventory, and lead time, giving the website the minimum data needed for display and lead-time estimation.
  - ProductMessage provides a simple envelope for returning collections of ProductItem.
- Order intake payload:
  - OrderMessage aggregates all customer/order details with a list of OrderItem line items, forming a single, consistent contract for order submission from the web.
- Mapping from domain to web model:
  - ProductItem includes a mapper-style constructor from CatalogItem, simplifying transformation from internal domain objects to web DTOs.

4) Notable patterns and architectural decisions
- DTO/Message models: All classes are anemic, transport-focused POJOs intentionally free of business logic, optimized for inter-service and UI boundary crossing.
- Anti-corruption layer: The package isolates external/web schemas from core MRP domain models (e.g., CatalogItem), reducing coupling and allowing independent evolution/versioning.
- Envelope pattern: ProductMessage serves as a response wrapper for lists of product projections, simplifying REST responses and enabling future metadata additions without breaking clients.
- Composition over inheritance: OrderMessage contains OrderItem list; ProductMessage contains ProductItem list. This keeps models small and composable.
- JavaBean/Jackson conventions: No-arg constructors and setters favor framework compatibility in controllers/adapters.
- Channel-specific models: The website namespace implies these contracts are tailored to the web channel, allowing other integration channels (e.g., EDI, partner APIs) to define their own variants without leaking internal types.
- Trade-offs:
  - Mutability and lack of validation/concurrency guarantees favor flexibility and ease of serialization but rely on upstream validation and downstream domain logic to enforce invariants.
  - String-based order date implies a need for agreed formatting (typically ISO-8601) at the API boundary.

In summary, integration.models.website provides the thin, stable, JSON-friendly contracts that connect the website to the MRP system for both product discovery/availability and order submission, using simple DTOs, mapping from domain models, and message envelopes to keep the integration clean, decoupled, and maintainable.

### 7. Package: `smpl.ordering.models`
**Files**: 19

Package-level summary: smpl.ordering.models

1) Overall purpose and role
- This package defines the core domain models/DTOs for the ordering area of the Parts Unlimited MRP system. It provides the canonical data shapes used end-to-end across the web/API tier, services, and persistence (e.g., MongoDB) to represent quotes, orders, shipments, and related contact/location information.
- It anchors the fulfillment lifecycle: quote creation and pricing, conversion to an order with status progression and audit events, and shipment coordination with delivery details and tracking events. The models enable consistent serialization, validation, comparison, and transport between components and external integrations.

2) How the files work together
- Quote lifecycle
  - Quote aggregates line items (QuoteItemInfo), basic customer/dealer and address details, and pricing totals/discounts. QuoteItemInfo standardizes SKU-based items with ordering via Comparable and reliable deduplication via equals/hashCode.
  - DealerInfo provides structured dealer contact details referenced by the quote context.
- Order lifecycle
  - Order links back to its quote via quoteId, carries its OrderStatus (from OrderStatus enum), and maintains an audit trail of OrderEventInfo entries. OrderUpdateInfo packages an OrderStatus change with an OrderEventInfo for updates flowing through APIs/services.
- Shipment lifecycle
  - ShipmentRecord is tied to an Order (by orderId), holds delivery specifics (DeliveryAddress), contact channels (PhoneInfo), and a series of ShipmentEventInfo entries as a tracking timeline.
- Delivery aggregate
  - Delivery composes the three pillars—Quote, Order, ShipmentRecord—into a single container used by fulfillment workflows to traverse pricing context, transactional state, and logistics tracking cohesively.
- Cross-cutting behaviors
  - Minimal validation methods on key models catch missing or malformed essentials (e.g., names, IDs, core address fields) and produce simple error payloads.
  - Copy constructors and equals/hashCode are implemented where needed to enable cloning, caching, comparison, and consistent behavior in collections.
  - All models expose JavaBean-style getters/setters to support REST JSON (de)serialization and MongoDB document mapping.

3) Key functionalities provided
- Canonical DTOs for:
  - Pricing and quoting: Quote, QuoteItemInfo, DealerInfo
  - Orders and lifecycle: Order, OrderStatus, OrderEventInfo, OrderUpdateInfo
  - Shipments and tracking: ShipmentRecord, ShipmentEventInfo, DeliveryAddress, PhoneInfo
  - Aggregate composition: Delivery (linking Quote + Order + ShipmentRecord)
- Lifecycle tracking and audit trails via lists of OrderEventInfo and ShipmentEventInfo.
- Lightweight validation for required fields to improve data quality at API boundaries.
- Deterministic equality, hashing, and sorting to support deduplication, caching, and stable UI/API behavior.
- Convenience mutators to append items/events and aid service-layer orchestration.

4) Notable patterns and architectural decisions
- DTO/POJO-first design: Models are serialization-friendly, with simple types (often String for dates) to minimize coupling to frameworks and ease REST/MongoDB usage.
- DDD-inspired separation:
  - Entities/aggregates: Order, Quote, ShipmentRecord, and the Delivery aggregate.
  - Value objects: QuoteItemInfo, PhoneInfo, DeliveryAddress; enums for constrained states (OrderStatus).
  - Timeline/event records: OrderEventInfo and ShipmentEventInfo as lightweight, append-only history entries.
- Aggregate composition: Delivery acts as a read/write façade over the three core artifacts, simplifying cross-cutting workflows (pricing → order → logistics).
- Minimal domain logic in models: Validation is intentionally light; business rules and transitions are expected to live in services, keeping models portable and testable.
- Equality and ordering semantics: Implemented where business identity matters (e.g., order-insensitive item comparison in Quote, SKU-based sorting in QuoteItemInfo) to support consistent calculations and deduplication.
- Pragmatic simplicity over strict typing: Dates stored as Strings favor interoperability and lenient parsing at the edges, at the cost of stronger temporal semantics—consistent with a microservice DTO layer.

In summary, smpl.ordering.models is the canonical data contract for the ordering/fulfillment domain, stitching together quotes, orders, and shipments with consistent, REST-ready models. It underpins validation, auditability, state management, and aggregate navigation required by manufacturing and supply chain workflows across the repository.

### 8. Package: `smpl.ordering.repositories.mongodb.models`
**Files**: 7

Package-level summary:

1) Overall purpose and role
- This package defines the MongoDB document models used by the ordering service in the Parts Unlimited MRP system. These classes are the storage-facing representations of core business entities (catalog items, quotes, orders, dealers, shipments) and form the translation layer between Spring Data MongoDB repositories and the domain/service models.
- By isolating persistence concerns and providing explicit mappings to and from domain objects/DTOs, the package preserves domain purity while enabling efficient, document-oriented storage that supports pricing, availability, order processing, shipment tracking, and audit workflows.

2) How the files work together
- CatalogItem: Stores SKU attributes (skuNumber, description, price, inventory, leadTime) with an index on SKU for fast lookup. It maps to the domain CatalogItem and computes an effective lead time (0 if inventory is available; otherwise stored leadTime) used by quoting and planning.
- QuoteDetails: Persists a Quote (identity, validity window, customer/dealer, address, pricing, and line items). It reconstructs the domain Quote for business operations like pricing, approval, and conversion to orders.
- OrderDetails: Persists an Order (orderId, quoteId, status, orderDate) and its event history. It reconstructs the domain Order used for processing, tracking, and audit, and maintains a non-null events array for robustness.
- Dealer: Persists dealer master data and maps to/from DealerInfo DTOs used by APIs and services.
- ShipmentDetails: Persists shipment records (delivery address/contact, events) with an index on orderId to tie shipments to orders and to reconstruct ShipmentRecord for fulfillment tracking and notifications.
- Interaction pattern: Services work with domain objects and DTOs. When persisting, they construct the corresponding MongoDB model from the domain object and save via Spring Data repositories. When reading, repositories return these document models, which are converted back to domain objects for business logic. Cross-entity relationships are maintained by IDs (e.g., orderId, quoteId, skuNumber) rather than MongoDB DBRefs, enabling loose coupling and scalable join-by-application logic.

3) Key functionalities provided
- Document schemas for core ordering artifacts with Spring Data annotations (@Document, @Id, and selective @Indexed).
- Bi-directional mapping utilities:
  - Constructors that copy from domain/DTO objects into persistence models.
  - toDomain methods (e.g., toCatalogItem, toQuote, toOrder, toShipmentRecord, toDealerInfo) that reconstruct domain/DTO objects.
- Business-relevant persistence semantics:
  - Effective lead-time calculation in CatalogItem based on inventory, aligning stored data with availability logic used by MRP.
  - Storage of ordered event histories for orders and shipments to support tracking and auditing.
  - Defensive null handling (e.g., non-null arrays) to ensure resilience on read/write.
- Identity handling for repository operations (separate MongoDB id plus business identifiers like skuNumber, orderId, quoteId for correlation).
- Read performance considerations via indexes (e.g., skuNumber, orderId).

4) Notable patterns and architectural decisions
- Data Mapper pattern: Each document model maps to a domain entity or DTO, keeping persistence logic out of the domain layer.
- Anti-corruption/boundary layer: These models shield domain objects from persistence-specific concerns and schema drift, preserving a clean separation between the domain/service layers and the database.
- Document-oriented, denormalized modeling: Event histories and line items are embedded within aggregate documents to optimize read scenarios typical in ordering and shipment tracking.
- Id-based referencing: Cross-aggregate relationships use IDs (orderId, quoteId) instead of DBRefs, improving service autonomy and scalability in a microservice context.
- Consistent conventions: Spring Data annotations, explicit id accessors, null-safety for collections, and simple, serializable field structures across all models.

In sum, smpl.ordering.repositories.mongodb.models provides the persistence-facing representations and mapping logic that let the ordering service store, retrieve, and faithfully reconstruct its core domain objects while aligning document schemas and indices with the access patterns of MRP workflows.

### 9. Package: `smpl.ordering`
**Files**: 15

Package-level summary: smpl.ordering

1) Overall purpose and role
- The smpl.ordering package provides the operational backbone for the Parts Unlimited MRP ordering service. It boots and wires the service (web tier, persistence, repository strategy, telemetry), enforces consistent request/error semantics, and supplies cross-cutting infrastructure (CORS, observability, configuration). It also includes the test scaffolding needed to bring up a representative runtime in CI and local development. In short, it is the service’s “infrastructure and platform” layer that enables order, quote, and shipment APIs to run reliably and be observable across environments.

2) How the files work together
- Bootstrapping and runtime wiring:
  - OrderingInitializer is the servlet container entry point (WAR deployments). It registers OrderingConfiguration and exposes the app’s context path.
  - OrderingConfiguration is the main Spring Boot configuration. It:
    - Builds the MongoTemplate using MongoDBProperties and environment variables (supports Docker-linked MONGO_PORT).
    - Chooses the storage/repository strategy via OrderingServiceProperties (e.g., in-memory vs. persistent) and configures a global RepositoryFactory used by the ordering domain.
    - Creates and exposes an Application Insights TelemetryClient (per-thread access) using the instrumentation key in OrderingServiceProperties.
    - Exposes the Spring ApplicationContext for global access.
  - PostgresqlProperties and MongoDBProperties externalize database settings for optional/alternate persistence endpoints.
  - OrderingServiceProperties centralizes operational toggles: storage backend selection, health/ping message, validation status, and telemetry key.

- Cross-cutting web concerns:
  - AppInsightsFilter instruments every HTTP request/response with timings, status, exception details, and correlation (operation/session IDs) using the TelemetryClient provisioned by OrderingConfiguration (and retrievable via Utility). It degrades gracefully if telemetry is not configured.
  - SimpleCORSFilter adds permissive CORS headers to enable browser-based and partner integrations to call the service across origins.

- Error handling and validation:
  - BadRequestException and ConflictingRequestException provide domain-specific exception types that surface clear messages and map naturally to HTTP 400 and 409 responses in controllers/services. They standardize how validation failures and resource/version conflicts propagate through the stack.
  - Utility offers common validation helpers (null/empty checks, multi-field validation message building) and a way to fetch the TelemetryClient from the global ApplicationContext.

- Configuration plumbing:
  - PropertyHelper loads classpath properties into a shared Properties object, enabling legacy/utility-style configuration access alongside Spring Boot’s @ConfigurationProperties.

- Testing support:
  - TestOrderingConfiguration mirrors production wiring for tests: a shared MongoClient/MongoTemplate, RepositoryFactory selection from properties, and a TelemetryClient if configured. It also exposes the ApplicationContext statically for non-managed test code.
  - ConfigurationRule brings up the Spring context (using TestOrderingConfiguration) before JUnit tests run.
  - TestPath is a minimal functional interface to standardize resetting stateful test components.
  - UtilityTest validates Utility behavior and guards against unintended telemetry in unit tests.

3) Key functionalities provided
- Service bootstrap and environment-aware wiring for the ordering component (Spring Boot application configuration, WAR support).
- Persistence configuration and selection (MongoDB primary, PostgreSQL properties available for integration/reporting; repository strategy chosen via configuration).
- Observability/telemetry via Azure Application Insights (per-request metrics, exception tracking, correlation IDs).
- Cross-origin access enablement through a CORS filter.
- Consistent, domain-specific error signaling (bad requests, conflicts) that aligns with REST semantics.
- Input validation utilities and common helpers.
- Centralized, externalized configuration for databases, telemetry, and service health/ping strings.
- Test-time infrastructure that faithfully represents production wiring with safe defaults.

4) Notable patterns and architectural decisions
- Spring Boot-centric microservice with dual deployment modes: fat-jar and WAR (SpringBootServletInitializer).
- Externalized configuration via @ConfigurationProperties, enabling environment-driven behavior without code changes.
- Strategy pattern for repository/storage selection driven by OrderingServiceProperties, allowing easy toggling between in-memory and persistent backends.
- Cross-cutting concerns implemented as servlet filters (telemetry, CORS) to keep controllers/services focused on domain logic.
- Domain-specific exception taxonomy aligned to HTTP semantics (400/409), improving API clarity and client diagnostics.
- Observability-first design through Application Insights, including operation and session correlation for distributed tracing.
- Environment awareness (e.g., Docker-linked MONGO_PORT) for frictionless local/CI deployments.
- Practical but debatable conveniences:
  - Static access to ApplicationContext and context path simplifies integration but couples code to the container and can complicate testing.
  - Test context is not explicitly shut down in ConfigurationRule, risking resource leaks across large test suites.
  - Broad, permissive CORS defaults ease development and integrations but may need tightening for production security.

Collectively, these classes form the infrastructure layer that makes the ordering service configurable, observable, deployable, and testable, while standardizing error handling and cross-origin access for the broader manufacturing and supply chain workflows in the repository.

### 10. Package: `smpl.ordering.controllers`
**Files**: 11

Package-level summary: smpl.ordering.controllers

1) Overall purpose and role
This package is the HTTP/API layer of the Parts Unlimited MRP ordering service. It exposes RESTful endpoints that drive the end-to-end order lifecycle used in manufacturing and supply chain processes: managing master data (catalog items and dealers), creating and managing quotes, converting quotes to orders, tracking orders through status and events, creating and updating shipments, and surfacing delivery confirmations. It also provides operational health/status endpoints. In short, this package is the main integration surface for internal web tiers and external systems to interact with the ordering domain, with built-in validation, consistent REST semantics, and telemetry for observability.

2) How the files work together
- QuoteController is the starting point for customer intent. It creates and manages quotes and is used downstream by the OrderController when converting a quote into an order.
- OrderController governs the order lifecycle. It pulls data from Quote repositories to create orders and records order status changes as timestamped events. It exposes retrieval by ID, dealer, and status to support fulfillment and visibility.
- ShipmentController continues the lifecycle by validating that related quotes and orders exist before creating shipments. It appends shipment events, updates shipments, filters by shipment status, and aggregates shipment, order, and quote data to construct Delivery views and lists of confirmed deliveries.
- CatalogController manages the parts master data relied on by quotes and orders to ensure accurate pricing and item information.
- DealerController manages dealer master data referenced by orders and quotes. It deliberately includes a load-generating list operation to support APM/performance demonstrations.
- PingController provides /ping and /status endpoints to support liveness checks, deployment verification, and build traceability.

All controllers access persistence through a shared RepositoryFactory, which enables swappable backends (e.g., in-memory for tests) and consistent repository acquisition. They report unexpected exceptions to Application Insights via TelemetryClient for unified observability. Location header construction uses common path utilities (e.g., OrderingInitializer/TestPath) to standardize URI generation.

The test suites (OrderControllerTest, QuoteControllerTest, ShipmentControllerTest, CatalogControllerTest, DealerControllerTest) orchestrate integration-style flows over in-memory repositories. They validate cross-controller interactions such as quote-to-order conversion, order-to-shipment progression, event recording, status filtering, and data integrity, thereby safeguarding the package’s behavior and contracts.

3) Key functionalities provided
- Catalog management:
  - Create, retrieve (all/by SKU), update/upsert, and delete catalog items with validation, SKU uniqueness checks, and correct HTTP semantics.
- Dealer management:
  - Create, retrieve (all/by name), update, and delete dealers with validation and conflict detection; includes an intentionally heavy list operation for performance testing.
- Quote lifecycle:
  - Create quotes (auto-generate IDs when needed), retrieve (by ID/by customer), update, and delete with strict validation and Location headers.
- Order lifecycle:
  - Create orders from quotes, retrieve (by ID/by dealer with optional status), update orders/status, append date-normalized events, and delete orders.
- Shipment and delivery:
  - Create shipments (with precondition checks against related quote/order), retrieve (all/by status/by ID), update shipments, append shipment events, delete shipments, and produce delivery confirmations by aggregating shipment/order/quote data.
- Health, status, and diagnostics:
  - Lightweight liveness and detailed status endpoints, including build metadata loaded from buildinfo.properties and cached after first access.
- Cross-cutting behavior:
  - Request validation and ID consistency enforcement.
  - Consistent REST semantics (200/201/204/400/404/409/500) and Location header on resource creation.
  - Centralized exception logging to Application Insights for production-grade observability.

4) Notable patterns and architectural decisions
- REST-first, resource-oriented design: Each domain entity (CatalogItem, Dealer, Quote, Order, Shipment) has a dedicated controller acting as the aggregate’s HTTP boundary, with predictable URIs and methods.
- Clear lifecycle orchestration: Quote → Order → Shipment → Delivery, with controllers enforcing the prerequisites between stages and capturing domain events (order and shipment events) to preserve state transitions and auditability.
- Consistent validation and error mapping: Controllers uniformly validate payloads, enforce identifier consistency, and map repository outcomes to standard HTTP status codes. Many endpoints return 404 when a collection is empty, keeping client signaling explicit.
- RepositoryFactory as a pluggable persistence abstraction: A simple service-locator-style factory supplies repositories, making controllers easy to test with an in-memory backend and decoupled from specific storage implementations.
- Observability baked in: All controllers log unexpected exceptions to Application Insights via TelemetryClient. DealerController’s list endpoint includes intentional load generation to demonstrate APM capabilities.
- Location header standardization: Controllers construct Location headers on successful POSTs, aligning with REST best practices and easing client navigation.
- Operational endpoints with build metadata: PingController exposes basic and detailed health signals and caches build properties, aiding deployment validation and diagnostics.
- Integration-focused testing: Tests are designed as integration-style suites over an in-memory store, exercising cross-controller flows, verifying event recording and date normalization, and ensuring contract stability across status transitions and searches.
- Upsert behavior where appropriate: CatalogController supports upsert semantics on PUT, a pragmatic choice for master data maintenance.

Together, these controllers and tests deliver a cohesive, observable, and testable API surface that powers the manufacturing ordering workflow—ensuring accurate master data, predictable quote-to-order-to-shipment transitions, delivery confirmation, and reliable operations within the broader MRP ecosystem.

### 11. Package: `smpl.ordering.repositories`
**Files**: 11

Package-level summary: smpl.ordering.repositories

1) Overall purpose and role
This package defines the data-access layer for core manufacturing domains in the Parts Unlimited MRP system—catalog items (parts), dealers (suppliers), quotes, orders, and shipments. It provides storage-agnostic repository contracts and a central factory that selects and wires concrete implementations (in-memory for tests/dev or MongoDB for production). By standardizing how the rest of the application reads and writes domain data, it decouples business logic from persistence, enforces optimistic concurrency, and preserves auditability and lifecycle rules critical to quoting, order processing, fulfillment, and shipment tracking.

2) How the files work together
- Repository interfaces (CatalogItemsRepository, DealersRepository, QuoteRepository, OrderRepository, ShipmentRepository) define the contracts for CRUD, queries, updates, and domain-specific behaviors (e.g., event histories, status filters, one-shipment-per-order).
- RepositoryFactory acts as a switchable service locator/abstract factory. It initializes and shares the chosen repository set (memory or MongoDB), wires inter-repository dependencies where needed, and exposes static accessors used by services and tests.
- Cross-domain flows are reflected in repository interactions validated by tests:
  - Dealers and catalog items are seeded as foundational data.
  - Quotes reference dealers and catalog parts.
  - Orders are created from quotes and carry status/event history.
  - Shipments bind to orders, enforce one-per-order, and accumulate events (scans/status changes).
- The JUnit test suites (CatalogItemsRepositoryTest, DealersRepositoryTest, QuoteRepositoryTest, OrderRepositoryTest, ShipmentRepositoryTest) define and verify the behavioral contract of each repository and the integrity of cross-repo workflows. They reset the repository graph, seed deterministic data, and assert semantics such as upsert behavior, optimistic concurrency, status-based queries, and event/audit persistence.

3) Key functionalities provided
- Unified CRUD and query APIs across domains:
  - Catalog: list all, get by SKU, upsert with eTag, conditional delete.
  - Dealers: list all, get by name, upsert with eTag, conditional delete.
  - Quotes: get by ID, list by customer, list IDs by dealer, create with validation, update/delete with eTag.
  - Orders: existence checks, get by ID/quote ID, filter by status and by dealer+status, create from quote with validation, full/partial updates (including event metadata), delete with eTag.
  - Shipments: get by ID, list by status, create with validation and one-per-order rule, append shipment events, update with optimistic concurrency, delete with eTag.
- Optimistic concurrency control via eTag consistently applied to updates and deletes (and upserts where applicable).
- Event and audit capabilities:
  - Shipment event history (status updates, scans).
  - Order event history (status transitions, dates, comments).
- Storage abstraction and environment switching via RepositoryFactory (MEMORY vs. MONGODB), enabling predictable tests and production deployment without changing business code.
- Test-backed contract enforcement for correctness, data integrity, idempotency, and stable ordering of results.

4) Notable patterns and architectural decisions
- Repository pattern: Interfaces encapsulate persistence concerns and expose domain-focused operations, enabling multiple backends.
- Abstract Factory/Service Locator: RepositoryFactory centralizes selection and lifecycle of repository implementations for the entire app. The approach simplifies wiring but introduces global state and thread-safety/lifecycle caveats noted in the summary.
- Optimistic concurrency with eTags: Uniformly guards updates/deletes against lost updates in concurrent workflows typical of MRP systems.
- Contract-first, test-driven validation: Comprehensive JUnit suites codify expected repository behavior, cross-repository relationships, and business rules (e.g., one shipment per order), providing regression safety across backends.
- Auditability through event logs: Orders and shipments maintain histories to support traceability and operational visibility across fulfillment and tracking.
- Upsert semantics where natural (catalog, dealers), returning a boolean to distinguish update vs. insert, supporting efficient synchronization and integration scenarios.
- Input validation at the repository boundary (BadRequestException) to protect data integrity early in the persistence pipeline.

In sum, smpl.ordering.repositories is the backbone of the application’s persistence strategy. It standardizes and safeguards data operations for the core supply chain entities, cleanly separates business logic from storage concerns, and uses a factory plus rigorous tests to provide a reliable, switchable, and auditable data-access layer for manufacturing workflows.

### 12. Package: `smpl.ordering.repositories.mongodb`
**Files**: 6

Package-level summary: smpl.ordering.repositories.mongodb

1) Overall purpose and role
- This package is the MongoDB-backed data access layer for the Parts Unlimited MRP ordering domain. It implements repositories for core aggregates and master data—catalog items, dealers, quotes, orders, and shipments—and supplies a shared, resilient Mongo operations wrapper.
- It underpins the order-to-cash/fulfillment flow by persisting and querying data needed by APIs and services: creating quotes, converting them to orders, coordinating shipments, and maintaining supporting master data (catalog and dealers).
- Beyond CRUD, it enforces lightweight business rules at repository boundaries (e.g., dealer existence for quotes, preventing quote reuse for orders, one shipment per order), and standardizes access to MongoDB with retries and telemetry for operational reliability.

2) How the files work together
- MongoOperationsWithRetry is the shared infrastructure wrapper used by all repositories. It delegates to Spring Data MongoOperations, adds targeted retries on transient failures, and emits Application Insights telemetry. This creates a consistent, instrumented persistence layer across the module.
- MongoDealersRepository manages dealer master data used by other repositories. MongoQuoteRepository consults and upserts dealers to ensure quotes are tied to valid dealer records.
- MongoQuoteRepository bridges QuoteDetails (persistence) and Quote (domain). It generates quote IDs when needed, persists quotes, and offers queries by customer and dealer. It collaborates with MongoDealersRepository to keep dealer data in sync.
- MongoOrderRepository converts quotes into orders and maintains the order lifecycle. It verifies that a quote hasn’t been reused, maps OrderDetails to Order, updates status and event trails, and can query orders by quote, status, or dealer (leveraging quote data).
- MongoShipmentRepository manages shipments and enforces order-aware rules. It validates order existence via MongoOrderRepository, prevents multiple shipments per order, appends shipment events, and supports queries tied to order status or orderId.
- MongoCatalogItemsRepository maintains product master data used across quoting and ordering. It provides upsert semantics so catalog updates preserve identity while refreshing attributes.
- Cross-repository interactions mirror domain relationships:
  - Quote -> Dealer linkage (quote creation/upsert validates and maintains dealer).
  - Order -> Quote dependency (order creation from an existing quote with reuse prevention).
  - Shipment -> Order dependency (shipment creation allowed only for existing orders, one per order).
- All repositories provide reset/drop operations to support test/setup scenarios and clean initial states across collections.

3) Key functionalities provided
- CRUD and lifecycle operations for core entities:
  - Catalog: get, upsert, delete by SKU; full reset.
  - Dealers: list, get by name, upsert by name, delete by name; reset.
  - Quotes: create, get, update, delete; queries by customer and dealer; reset; quote ID generation.
  - Orders: existence checks, create from quote, get by order/quote/dealer/status, update with status/event append, delete; reset and counter reinit.
  - Shipments: get by orderId and order status, create with integrity checks, append events, update, delete by orderId; reset.
- Mapping between domain/API models and persistence entities (Details/DTOs), keeping business logic decoupled from storage specifics.
- Operational resilience and observability:
  - Retry on selected Mongo operations for transient failures (e.g., socket timeouts).
  - Telemetry for key operations (RemoteDependency events) to monitor latency and success.
- Business rule enforcement at the repository level:
  - Ensure dealer presence when creating/updating quotes.
  - Prevent quote reuse when creating orders.
  - Allow at most one shipment per order and only for existing orders.
- Utility behaviors:
  - Event-history append patterns for orders and shipments.
  - In-memory filtering for some queries where simple retrieval suffices.
  - Destructive resets for each collection to support test automation and environment seeding.

4) Notable patterns and architectural decisions
- Repository pattern: Each aggregate/master data type has a dedicated repository that encapsulates persistence logic and model mapping.
- Anti-corruption/data-mapper layer: Clear mapping between domain/API models and persistence “Details”/DTO types, isolating MongoDB schema from the rest of the application.
- Cross-cutting resilience and telemetry: A single MongoOperationsWithRetry wrapper centralizes retries and instrumentation without constraining Mongo’s feature surface.
- Aggregate-level coordination: Repositories collaborate to enforce cross-entity constraints (Quote→Order→Shipment flow; Dealer linkage for Quotes) while keeping strong boundaries between aggregates.
- Event append pattern: Orders and shipments maintain event histories by appending events to the stored document, offering simple traceability without a separate event store.
- Pragmatic trade-offs: Basic CRUD and queries, occasional in-memory filters, no pagination, and eTags currently ignored (no optimistic concurrency). Resets use dropCollection for simplicity in tests and demos.
- Consistency safeguards: Duplicate prevention (e.g., one shipment per order) and existence checks provide baseline integrity within a single datastore.

In sum, smpl.ordering.repositories.mongodb is the cohesive persistence backbone for the ordering subsystem, providing reliable, instrumented MongoDB access and enforcing essential domain rules across catalog, dealers, quotes, orders, and shipments to support end-to-end MRP workflows.

### 13. Package: `smpl.ordering.repositories.mock`
**Files**: 5

Package-level summary for smpl.ordering.repositories.mock

1) Overall purpose and role
- This package provides in-memory, database-free implementations of the repository interfaces used by the Parts Unlimited MRP ordering subsystem. It enables local development, demos, and automated tests to exercise end-to-end flows—catalog lookup, quote creation, order processing, and shipment tracking—without MongoDB or other external dependencies.
- By mirroring the contracts of the real repositories, these mocks are drop-in replacements wired via dependency injection. They let the rest of the application (services, controllers) run unchanged while keeping setup fast and deterministic.

2) How the files work together
- MockDealersRepository: Acts as the in-memory source of truth for dealers. It is used by MockQuoteRepository to validate and upsert dealer information during quote creation and updates.
- MockQuoteRepository: Manages quotes and links them to dealers. Provides search capabilities (by customer and by dealer). Serves as the upstream dependency for MockOrderRepository when turning quotes into orders.
- MockOrderRepository: Manages orders, enforces one-order-per-quote, supports status changes and event logging. It also filters by dealer by traversing back to the associated quote.
- MockShipmentRepository: Manages shipments and shipment events, enforcing a strict one-shipment-per-order rule. It uses MockOrderRepository to validate the order’s existence and to enable shipment queries filtered by order status.
- MockCatalogItemsRepository: Provides a seeded parts catalog (e.g., brake components) for lookups and CRUD-like updates. Though independent of the others, it underpins catalog-driven scenarios such as quote generation or inventory checks.
- Together, these repositories enable realistic flows: a dealer is validated/created when a quote is made; an order is created from a quote with uniqueness checks; a shipment is created for an existing order and annotated with events; the catalog supplies parts data used elsewhere in the ordering/MRP services.

3) Key functionalities provided
- CRUD-like operations across domain entities:
  - Dealers: upsert, list, find by name, remove, reset.
  - Quotes: create (auto-ID), read, update, remove, search by customer, list by dealer, reset.
  - Orders: create from quote with uniqueness validation, read by ID/quote ID, filter by status and dealer, update (replace or append event/status), reset.
  - Shipments: create (one per order), read by orderId, list filtered by order status, add shipment events, update/replace, reset.
  - Catalog items: list all, get by SKU (case-insensitive), upsert, remove, reset; seeded sample data for immediate use.
- Cross-entity validations and constraints:
  - Dealer validation/upsert on quote create/update.
  - One order per quote; one shipment per order.
  - Shipment queries by order status via order lookup.
- Operational characteristics for testing:
  - Defensive copying on reads to prevent external mutation.
  - Case-insensitive matching for keys like SKU and dealer name.
  - Simple ID generation for quotes; straightforward replacement updates for orders/shipments.
  - Reset methods across repositories for test isolation.
  - Lightweight and intentionally non-thread-safe; ignores eTag/concurrency controls.

4) Notable patterns and architectural decisions
- Repository pattern with in-memory backing: Each mock implements the corresponding repository interface, enabling clean substitution for persistence adapters (hexagonal/ports-and-adapters style).
- Dependency injection and composition: Mocks depend on each other (e.g., shipment→order, order→quote, quote→dealer) to enforce domain rules while remaining modular.
- Defensive copying: All reads return new instances to protect internal state—important in tests where objects may be mutated by callers.
- Happy-path coverage with minimal infrastructure: Concurrency, synchronization, and persistence are intentionally omitted to keep the mocks fast and simple.
- Deterministic testing support: Seeded data (catalog), reset capabilities, and simple ID generation facilitate repeatable test runs.
- Event-style updates: Orders and shipments support appending events alongside status/state transitions, approximating event logging without a full event-sourcing stack.
- Pragmatic stubbing: Some operations (e.g., delete in orders, remove in shipments) are stubbed or simplified, signaling intended behavior without overbuilding the mock layer.

In summary, smpl.ordering.repositories.mock provides a cohesive, in-memory simulation of the ordering subsystem’s persistence layer, enabling rapid, end-to-end development and testing of catalog, quote, order, and shipment workflows while preserving the application’s repository contracts.

### 14. Package: `smpl.ordering.repositories.mock.test`
**Files**: 5

Package-level summary: smpl.ordering.repositories.mock.test

1) Overall purpose and role
This package contains mock-backend conformance tests for the Parts Unlimited MRP system’s repository layer. Its purpose is to validate that the in-memory (mock) implementations of key repositories—Catalog Items, Dealers, Quotes, Shipments, and Orders—adhere to the same behavioral contracts as their real (e.g., MongoDB) counterparts. By doing so, it provides fast, deterministic, and infrastructure-free verification of core data-access operations that underpin manufacturing and supply chain workflows (catalog management, dealer/supplier data, quoting, orders, and shipments). These tests enable reliable CI and rapid developer feedback without requiring external services.

2) How the files work together
- Each test class is a thin JUnit subclass that:
  - Forces the repository layer to use the in-memory backend via RepositoryFactory.reset("memory") or equivalent setup.
  - Inherits and executes a shared base test suite for its respective repository type.
- The shared base tests (located elsewhere) define the repository contract and the expected semantics for CRUD and domain-specific queries. These mock test subclasses supply the backend configuration and then delegate all assertions to the base tests.
- Collectively, these classes ensure that all major domain repositories behave correctly when the in-memory implementation is in use. Although they do not call each other directly, they operate as a cohesive set of contract tests that cover the entire repository landscape for the MRP domain.

3) Key functionalities provided
- Backend switching for tests:
  - Centralized reconfiguration of RepositoryFactory to use the in-memory “memory” implementation, isolating tests from external databases.
- Contract verification across repositories:
  - Catalog Items: get all, get single, upsert, delete.
  - Dealers: get all, get single, upsert, delete.
  - Quotes: retrieval, customer-specific queries, create/update/delete.
  - Orders: existence, retrieval, queries by quote ID/status/dealer name, create/update.
  - Shipments: get by ID, list, create/update, add event.
- Fast, deterministic CI validation:
  - Ensures core data-access behaviors are stable and consistent for workflows like quoting, order processing, and shipment tracking without requiring MongoDB or other infrastructure.

4) Notable patterns and architectural decisions
- Repository Pattern with pluggable backends:
  - A RepositoryFactory selects between multiple implementations (e.g., in-memory vs. persistent), decoupling domain logic from storage.
- Contract testing via inheritance:
  - Abstract/base test suites define shared expectations for each repository. Mock test subclasses simply configure the backend and reuse the full suite, promoting DRY test design and consistency.
- In-memory test doubles:
  - The memory implementation serves as a fast, deterministic test double, enabling quick feedback and reliable CI pipelines.
- Separation of concerns and portability:
  - Tests affirm that application logic depending on repositories behaves consistently regardless of the underlying storage technology, reducing integration risk and supporting local development without external services.

In summary, smpl.ordering.repositories.mock.test ensures that the in-memory repository implementations faithfully match the repository contracts. This underpins stable, rapid testing of critical manufacturing and supply chain operations by validating the data-access layer independently of real infrastructure.

### 15. Package: `smpl.ordering.repositories.mongodb.test`
**Files**: 6

Package-level summary:

1) Overall purpose and role
- This package provides MongoDB-specific contract tests for the core repositories used by the Parts Unlimited MRP system (orders, quotes, shipments, dealers, catalog items). Its purpose is to validate that the Mongo-backed persistence adapters behave exactly like the shared repository contract expects, safeguarding backend-agnostic behavior for critical manufacturing workflows such as procurement, inventory management, order processing, and shipment coordination.

2) How the files work together
- Each Mongo*RepositoryTest class is a thin adapter-specific harness that:
  - Selects the MongoDB implementation via RepositoryFactory.reset("mongodb") before each test, ensuring a clean, deterministic backend.
  - Applies a common ConfigurationRule to standardize environment/policy setup.
  - Delegates all behavioral testing to a shared superclass that contains the canonical repository contract tests.
- IntegrationTests is a marker interface used to categorize and orchestrate these tests as integration tests in CI pipelines.
- Together, these tests run the same suite of assertions used for other persistence backends, but against MongoDB, guaranteeing parity and preventing regressions when switching or evolving storage implementations.

3) Key functionalities provided
- Backend selection and isolation: Forces the repository layer to MongoDB for each test run to ensure consistency and independence from other backends.
- Contract test reuse: Executes shared test suites covering core repository behaviors:
  - Dealers: list, fetch, upsert, remove.
  - Catalog items: list, fetch, upsert, remove.
  - Shipments: fetch by ID, list, create, update, add event.
  - Quotes: get, query by customer name, create, update, remove.
  - Orders: existence, retrieval, creation, updates, queries by quote ID, status, and dealer name.
- Environment standardization: Uses a ConfigurationRule to keep configuration consistent across tests.
- Test categorization: Uses IntegrationTests to tag and control execution of longer-running, environment-dependent tests in build pipelines.

4) Notable patterns and architectural decisions
- Repository pattern with pluggable persistence: The RepositoryFactory enables swapping storage backends (e.g., MongoDB) while keeping domain logic and tests backend-agnostic.
- Contract testing via inheritance: Mongo-specific test classes extend shared base test suites, ensuring a single source of truth for repository behavior and maximizing reuse (template method/polymorphic testing style).
- Clear separation of concerns: Adapter-specific tests focus solely on selecting/configuring the backend; behavior assertions remain in shared tests.
- CI-friendly test orchestration: Marker interface (category) allows selective execution of integration tests, balancing fast unit feedback with deeper end-to-end validation.
- Consistency and regression prevention: Running the same contract across all backends enforces parity and preserves data integrity and lifecycle correctness essential to manufacturing and supply chain flows.

---
## File Summaries
### Package: `integration`
#### Main.java
- Role: Entry-point bootstrapper for the integration module, launching scheduled background processes that connect MRP workflows with external systems.

- Key Functionality: Starts a Spring application context with CreateOrderProcessTask and UpdateProductProcessTask so their schedulers run; initializes and keeps alive background threads that process order creation and product updates.

- Purpose: Enable continuous, automated integration tasks—such as synchronizing product data and processing orders—supporting Parts Unlimited MRP’s inventory, order fulfillment, and supplier coordination workflows by running essential background jobs without requiring command-line configuration.

#### Constants.java
- Role: Defines a shared timing constant for the integration layer, acting as a small utility holder to standardize scheduling cadence across integration tasks.

- Key Functionality: 
  - Exposes a public static final interval (30,000 ms) for scheduled/periodic operations (e.g., polling external systems, cache refresh, heartbeat).
  - Prevents instantiation via a private constructor, reinforcing its use as a constants/utility class.

- Purpose: Ensures consistent, centralized control of the polling interval used by integration processes (such as supplier updates, order/shipment status syncs, and external system coordination), reducing magic numbers and supporting reliable, predictable synchronization in the MRP workflows.


### Package: `integration.infrastructure`
#### ConfigurationHelpers.java
- Role: Configuration utility for the integration layer, centralizing access to application properties used by the Parts Unlimited MRP integration services.

- Key Functionality: 
  - Loads properties from classpath files into a shared Properties store.
  - Retrieves configuration values as strings and integers with fallback defaults.
  - Exposes the underlying Properties for broader configuration access.
  - Note: Current implementation swallows exceptions, may mask misconfigurations (returning "", 0, or null), and risks resource leaks by not closing input streams.

- Purpose: Provide a simple, centralized mechanism for reading and using configuration needed by manufacturing integration workflows (e.g., endpoints, credentials, timeouts for supplier, order, and shipment integrations). This supports environment-specific configuration to enable reliable order processing, inventory updates, and shipment coordination across external systems.

#### ConfigurationManager.java
- Role: Centralized configuration accessor for the integration layer, exposing environment-specific settings required by the Parts Unlimited MRP application to communicate with external services.

- Key Functionality:
  - Provides static, typed getters for critical configuration values:
    - Azure Storage connection string
    - Azure Storage Queue names for orders and inventory
    - Azure Queue message/timeout setting
    - MRP service endpoint
  - Delegates retrieval to ConfigurationHelpers, centralizing configuration keys and avoiding duplication.
  - Maintains no internal state and performs no validation, relying on the helper for parsing and error behavior.

- Purpose: To decouple business logic from configuration sources, enabling flexible deployment across environments (dev/test/prod) and consistent access to integration settings. This supports core MRP workflows—order processing, inventory updates, and service-to-service communication—by reliably wiring the application to Azure queues and the MRP endpoint.


### Package: `integration.models`
#### QueueResponse.java
- Role: A lightweight integration-layer model that pairs an Azure Storage Queue message with a strongly typed payload, enabling downstream components to process queue-driven work while retaining access to the original message metadata and lifecycle controls.

- Key Functionality: 
  - Encapsulates a CloudQueueMessage and a generic response body (T) in a single immutable container.
  - Exposes simple getters to retrieve the queue message and the typed payload used by business logic.
  - Serves as a handoff object within the integration pipeline, keeping the message reference available for completion, retry, or visibility updates after processing.

- Purpose: Provides a type-safe wrapper for asynchronous queue processing across MRP integration workflows (e.g., order updates, shipment events, supplier feeds). By bundling the queue metadata with the processed/parsed payload, it simplifies message handling, promotes consistent processing, and supports reliable acknowledgment/visibility management—improving robustness and clarity in Parts Unlimited’s manufacturing and supply chain integrations.


### Package: `integration.models.website`
#### OrderMessage.java
- Role: A website-facing integration DTO/POJO that models an order message used to move order data between the web front end and backend MRP services.
- Key Functionality: Encapsulates customer/dealer info, contact and shipping address, String-based order date, financials (totalCost, discount), and a list of line items; provides JavaBean getters/setters and is Jackson-friendly for JSON serialization/deserialization in REST APIs.
- Purpose: Standardizes the payload for order intake and handoff, enabling order processing, inventory checks, quote calculation, and shipment coordination across Parts Unlimited MRP’s integration layer.

#### OrderItem.java
- Role: A lightweight integration-layer model/DTO representing a single order line item from the website for the Parts Unlimited MRP system.

- Key Functionality: Encapsulates a product’s SKU (business key) and unit price with simple JavaBean getters/setters, enabling serialization (e.g., JSON), logging, and lookups that tie website orders to catalog, pricing, and inventory data.

- Purpose: Provide a stable, minimal contract for exchanging order item details between the web front end and backend services, supporting core MRP workflows such as inventory checks, quote/totals calculation, order fulfillment, and shipment coordination while maintaining consistent part identification across systems.

#### ProductItem.java
- Role: Website-facing integration model (DTO) that bridges the core MRP CatalogItem to the web/integration layer, representing a product SKU with availability and timing metadata.

- Key Functionality:
  - Encapsulates SKU identity (skuNumber), stock level (inventory), and fulfillment buffer (leadTime).
  - Provides a no-arg constructor and a mapper-style constructor that copies values from an MRP CatalogItem.
  - Supplies simple getters/setters for serialization and transport across REST APIs and UI layers.

- Purpose: Enable the website and integration services to consume and display essential product availability data from the MRP system, supporting catalog views, quote lead-time estimations, and order fulfillment decisions. It standardizes SKU identity across systems and surfaces inventory and lead-time information for downstream processes, albeit as a lightweight POJO without validation or concurrency guarantees.

#### ProductMessage.java
- Role: A website-integration DTO that packages and transports collections of products from the MRP catalog to the web/UI or external clients, acting as an envelope for product listings in the integration layer.

- Key Functionality: 
  - Maintains a list of ProductItem objects representing products to display or process.
  - Builds its product list from MRP CatalogItem entries, converting them to ProductItem instances.
  - Provides basic accessors (getter/setter) for the product list, enabling population and serialization for REST responses or UI consumption.

- Purpose: To bridge core MRP catalog data to website-facing models, enabling catalog browsing, search results, carts, and order/quote workflows. It standardizes product payloads for integration, simplifying data flow between services and the web tier, though its current design exposes a mutable list that may warrant tighter encapsulation.


### Package: `integration.scheduled`
#### UpdateProductProcessTask.java
- Role: Scheduled integration job that bridges the MRP catalog to the messaging layer, feeding product updates from the MRP service into the system’s Azure queue. It resides in the integration layer to synchronize catalog data across Parts Unlimited MRP components.

- Key Functionality:
  - Periodically invokes the external MRP endpoint (via MrpConnectService) to fetch CatalogItem records.
  - When items are present, batches them into a single ProductMessage and publishes it to the configured Azure inventory queue (via QueueService).
  - Uses SLF4J for class-scoped logging; logs and swallows exceptions to prevent scheduler disruption.
  - Stateless execution per run: constructs service instances on each invocation and no-ops when no items are returned.

- Purpose: Ensure timely, decoupled propagation of product and inventory updates from the MRP system to downstream services (e.g., web front end, order/quote processing) through a resilient queue. This supports accurate catalog visibility, smoother order fulfillment, and consistent data across manufacturing and supply chain workflows.

#### CreateOrderProcessTask.java
- Role: Scheduled integration worker that bridges the order intake pipeline to the external MRP system by consuming queued order messages and creating corresponding orders.

- Key Functionality: 
  - Periodically executes (via Spring scheduling) using ConfigurationManager to resolve the MRP endpoint and Azure order queue.
  - Iteratively pulls OrderMessage items from the Azure queue, invokes MrpConnectService.createNewOrder to create orders, and deletes processed messages.
  - Logs each step and aborts the current run on any exception, resulting in at-least-once processing semantics (possible duplicates if failures occur after order creation but before message deletion).

- Purpose: Automates the handoff of customer orders into the MRP backend to drive production planning and fulfillment, reducing manual effort, improving timeliness, and keeping inventory and order state synchronized across systems within the Parts Unlimited MRP application.


### Package: `integration.services`
#### QueueService.java
- Role: Generic Azure Queue integration service that enables the integration layer to exchange typed messages between Parts Unlimited MRP components and external systems, supporting asynchronous workflows across the application.

- Key Functionality:
  - Retrieves one message from an Azure Storage Queue with a configurable visibility timeout, deserializes JSON payloads into a target type T, and returns both the raw message and typed content via QueueResponse.
  - Adds messages by serializing objects of type T to JSON and enqueuing them.
  - Deletes messages explicitly, including automatic cleanup of poison messages when deserialization fails (with structured SLF4J logging).
  - Relies on ConfigurationManager for queue settings and resolves CloudQueue instances via a queue factory; propagates Storage/URI/credential-related exceptions.

- Purpose: Provide a reliable, type-safe messaging conduit for MRP integrations—decoupling processes like parts procurement, order processing, quote generation, and shipment updates—thereby improving scalability, resilience, and observability in the manufacturing and supply chain workflows.

#### QueueFactory.java
- Role: Factory/registry for Azure Storage Queue clients used by the Integration Service to support asynchronous messaging across the MRP system (orders, inventory updates, shipments, supplier events).

- Key Functionality: 
  - Lazily creates and retrieves CloudQueue instances by name using the configured Azure Storage connection.
  - Ensures queues exist in Azure (create-if-not-exists) before use.
  - Caches CloudQueue references in a static, thread-safe map for reuse across calls and components, reducing connection and lookup overhead.

- Purpose: Provide a centralized, efficient, and consistent way to access Azure queues, enabling scalable, reliable message-based integration for manufacturing workflows such as order processing, production planning updates, procurement events, and shipment tracking within Parts Unlimited MRP.

#### MrpConnectService.java
- Role: Integration-layer service that acts as a REST client/adapter between the web front end and the MRP back-end services for Parts Unlimited MRP.

- Key Functionality: 
  - Orchestrates the order lifecycle: creates a quote, converts it to an order, and initiates a shipment (createNewOrder).
  - Performs targeted REST calls to MRP endpoints to create quotes, orders, and shipments.
  - Retrieves product catalog items from the MRP catalog service.
  - Centralizes HTTP communication via Spring RestTemplate and logs operational milestones via SLF4J.

- Purpose: Provide a simple, centralized connector to MRP services that enables seamless order processing and shipment coordination from website-originated requests, supporting core manufacturing workflows such as quote generation, order fulfillment, and catalog access to drive efficient supply chain and production operations.


### Package: `smpl.ordering`
#### BadRequestException.java
- Role: A domain-specific exception representing client-side request errors within the Parts Unlimited MRP ordering and service layers.

- Key Functionality: 
  - Encapsulates “bad request” conditions with a clear, human-readable message.
  - Enables consistent propagation of input/validation failures through the stack.
  - Facilitates mapping to HTTP 400 responses in REST controllers and service endpoints.

- Purpose: Provides a standardized mechanism to reject invalid inputs early (e.g., malformed order data, invalid part IDs, incorrect quote parameters), improving API clarity, protecting data integrity in inventory/order workflows, and reducing troubleshooting overhead across order processing, quote generation, and shipment tracking.

#### ConflictingRequestException.java
- Role: Defines a domain-specific exception for signaling request conflicts within the ordering component of the Parts Unlimited MRP system.

- Key Functionality: 
  - Encapsulates conflict/error conditions (e.g., duplicate orders, concurrent update/version conflicts, inventory reservation clashes) as a dedicated exception type.
  - Enables consistent propagation of conflict states through service layers and REST endpoints, typically mapping to HTTP 409 Conflict.
  - Provides clear, human-readable messages via its constructor for diagnostic logging and client feedback.

- Purpose: Improves robustness and clarity of error handling across order processing and related workflows by standardizing how conflicting requests are identified and communicated, reducing ambiguity for both developers and API consumers and supporting reliable MRP operations such as order fulfillment and inventory management.

#### AppInsightsFilter.java
- Role: Servlet filter that instruments the web tier with Azure Application Insights telemetry for the Parts Unlimited MRP application’s ordering and related REST endpoints.

- Key Functionality: 
  - Wraps each HTTP request to record start time, duration, method, URL (including query), response status, and success.
  - Captures and reports exceptions with handled-at context.
  - Sets session and operation correlation IDs in TelemetryContext to enable distributed tracing across services.
  - Falls back to pass-through behavior when telemetry is not configured; init/destroy are no-ops.

- Purpose: Provide observability and operational monitoring for manufacturing workflows (e.g., catalog, quote, order processing, shipment tracking) by tracking request performance and errors, enabling faster diagnostics, SLA tracking, and performance tuning across the MRP system’s web and service layers.

#### OrderingConfiguration.java
- Role: Central Spring Boot configuration and bootstrap class for the Ordering service in the Parts Unlimited MRP system, acting as the entry point and infrastructure wiring hub.

- Key Functionality:
  - Starts the application via a main method using OrderingConfiguration.
  - Builds a MongoTemplate for MongoDB using externalized properties and optional Docker-linked MONGO_PORT environment settings.
  - Configures the global RepositoryFactory based on OrderingServiceProperties.
  - Manages Application Insights telemetry by providing a per-thread TelemetryClient with an instrumentation key from configuration.
  - Exposes and holds the Spring ApplicationContext for global access where needed.

- Purpose: To provide the operational backbone (persistence, repository selection, telemetry, and context management) required for order processing, inventory interactions, and related MRP workflows, enabling environment-agnostic deployment and observability for manufacturing and supply chain operations.

#### SimpleCORSFilter.java
- Role: A cross-cutting HTTP servlet filter that configures CORS for the Parts Unlimited MRP web/service layer, ensuring browser clients can access REST endpoints hosted on Tomcat across different origins.
- Key Functionality: Intercepts all requests and adds permissive CORS headers (Allow-Origin “*”, methods PUT/POST/GET/OPTIONS/DELETE, short max-age, common request headers) before continuing the filter chain; includes no-op init and destroy; does not special-case OPTIONS/preflight.
- Purpose: Facilitate cross-origin access for the MRP web front end, integration services, and external supplier/partner systems, reducing CORS-related failures during development and operations; note that its open policy is convenient but may require tightening for production security and better preflight handling.

#### TestPath.java
- Role: A minimal functional interface in the ordering module that standardizes a reset capability for components involved in tests or stateful workflows.

- Key Functionality: 
  - Declares a single reset() method, enabling polymorphic handling of resettable objects.
  - Usable as a functional interface (lambdas/method references) to simplify test and utility code.
  - Provides a common contract without prescribing implementation details.

- Purpose: Ensures consistent, repeatable initialization of stateful components—such as test harnesses, workflow path objects, or fixtures—supporting clean test runs and reliable behavior in MRP scenarios like order processing, quote generation, and shipment coordination.

#### OrderingInitializer.java
- Role: Servlet container bootstrapper for the Ordering service, wiring Spring Boot into a WAR deployment and exposing the web application’s context path for global use within the Parts Unlimited MRP system.

- Key Functionality:
  - Configures the Spring application context by registering OrderingConfiguration as the primary source.
  - Hooks into servlet startup (via SpringBootServletInitializer) to perform standard initialization.
  - Captures and stores the application’s context path at startup and exposes it through a static accessor.

- Purpose: Ensure reliable initialization of the Ordering component in environments like Apache Tomcat and provide a consistent application path reference for constructing URLs and integrating web-layer resources. This supports stable delivery of ordering, quote, and shipment-related APIs critical to manufacturing order processing and coordination within the MRP platform.

#### PostgresqlProperties.java
- Role: Spring Boot configuration properties holder for PostgreSQL connectivity used by the ordering components of the Parts Unlimited MRP system.

- Key Functionality:
  - Encapsulates JDBC connection details (driverClass, url, username, password).
  - Exposes simple getters/setters for property binding via @ConfigurationProperties.
  - Enables switching drivers/URLs per environment without code changes.

- Purpose: Centralizes and externalizes database connection settings to support secure, portable, and configurable access to a PostgreSQL data source for order processing and related MRP workflows (e.g., quotes, shipments), complementing the repository’s primary persistence and facilitating integration/reporting needs.

#### OrderingServiceProperties.java
- Role: Configuration holder for the Ordering Service, encapsulating externally supplied settings that influence storage selection, health reporting, and telemetry in the Parts Unlimited MRP system.

- Key Functionality:
  - Exposes a configurable storage backend selector (e.g., in-memory vs. persistent) to drive repository/strategy choice.
  - Provides a ping/health message for lightweight service availability checks.
  - Maintains a validation message reflecting configuration/initialization status.
  - Supplies an instrumentation key for wiring application telemetry (e.g., Azure Application Insights).
  - Integrates with Spring Boot configuration binding via simple getters/setters for runtime/environment-driven customization.

- Purpose: Enable deployment-time flexibility, observability, and reliability of the Ordering Service by centralizing key operational settings. This supports faster dev/test cycles (in-memory storage), production-ready persistence, clear health signaling, and telemetry routing—contributing to robust order processing within the MRP workflow.

#### Utility.java
- Role: Shared utility helper for the ordering service, providing common validation and telemetry access used across Parts Unlimited MRP’s web and service layers.

- Key Functionality: 
  - Validates string inputs and accumulates human-readable error messages for multiple fields.
  - Provides a simple null/empty string check.
  - Retrieves the Application Insights TelemetryClient from Spring’s ApplicationContext for instrumentation.

- Purpose: Streamline request validation and operational telemetry within order processing, quote generation, and shipment workflows, improving data quality (by catching missing fields) and observability (by enabling metrics/logging) across the MRP application.

#### MongoDBProperties.java
- Role: Spring Boot configuration properties bean for the Ordering service that centralizes MongoDB connection settings used across Parts Unlimited MRP.
- Key Functionality: Encapsulates MongoDB host and database name with getters/setters, enabling externalized configuration (e.g., via application properties or environment variables) and consistent connection targeting across the service.
- Purpose: Provide environment-portable, maintainable control over persistence endpoints so the ordering, inventory, and shipment workflows reliably connect to the correct MongoDB instance/database without hard-coded values.

#### PropertyHelper.java
- Role: Utility for centralized configuration loading and access within the Parts Unlimited MRP system, supporting shared settings across web, order, and integration services.
- Key Functionality: 
  - Loads .properties files from the classpath into a java.util.Properties instance.
  - Exposes a shared static Properties store for global configuration access.
- Purpose: Externalizes environment-specific settings (e.g., inventory, order processing, shipment tracking, MongoDB, REST endpoints) to simplify deployment and integration, reducing hardcoded values and enabling consistent configuration management across manufacturing workflows.

#### ConfigurationRule.java
- Role: JUnit test bootstrapper for the ordering module, responsible for bringing up a Spring ApplicationContext during test execution.

- Key Functionality: 
  - Initializes a Spring AnnotationConfigApplicationContext with TestOrderingConfiguration when the rule is applied.
  - Leaves the original JUnit Statement unmodified, affecting only test setup.
  - Implicitly triggers bean creation and initialization used by order processing tests.
  - Does not manage context shutdown, which can lead to resource leaks across multiple tests.

- Purpose: Simplify and standardize test setup for Parts Unlimited MRP’s ordering-related components (e.g., order processing, catalog, and integration services) by auto-wiring Spring beans in tests. This supports reliable testing of manufacturing workflows and service interactions, improving confidence in MRP operations while acknowledging the need for better context lifecycle management.

#### TestOrderingConfiguration.java
- Role: Spring Boot test configuration for the Ordering module, wiring core infrastructure (database, repositories, telemetry) used across Parts Unlimited MRP services.

- Key Functionality:
  - Provides a lazily initialized, shared MongoClient and exposes a MongoTemplate configured via application properties and Docker-linked environment variables (supports multi-host configs).
  - Supplies a TelemetryClient bound to the active Application Insights configuration (or null if unavailable).
  - Configures and returns a RepositoryFactory based on the “ordering.storage” property, resetting global repository state accordingly.
  - Holds and exposes the Spring ApplicationContext statically for access by non-Spring-managed code.

- Purpose: To reliably bootstrap persistence, repository selection, and telemetry for ordering-related tests and runtime scaffolding, enabling consistent data access and observability for MRP workflows such as inventory management, order processing, quote generation, and shipment tracking. This configuration ensures environment-aware setup while centralizing critical infrastructure needed by the ordering service.

#### UtilityTest.java
- Role: JUnit test class validating core utility behavior within the ordering module of the Parts Unlimited MRP system, ensuring common helpers behave predictably across services.

- Key Functionality:
  - Tests Utility.isNullOrEmpty to confirm correct handling of null, empty, and whitespace-only strings.
  - Intended (but currently ineffective) test to ensure Application Insights TelemetryClient is disabled during unit tests to avoid external telemetry in CI.
  - Contains a public, mutable rule field representing configuration policy/rules, enabling per-instance customization in tests (though not actively exercised here).

- Purpose: Safeguards foundational utility behavior that impacts input validation across catalog, order, quote, and shipment workflows, and helps keep test runs isolated from external telemetry. This reduces regressions and noise in the MRP platform’s CI pipeline, supporting reliable manufacturing and supply chain operations.


### Package: `smpl.ordering.controllers`
#### ShipmentController.java
- Role: Spring MVC REST controller for managing shipment data within the Parts Unlimited MRP application’s ordering domain, acting as the HTTP interface to shipment and delivery operations.

- Key Functionality: 
  - Exposes endpoints to retrieve shipments (by status or ID), create shipment records, update existing shipments, add shipment events, delete shipments, and list confirmed deliveries.
  - Aggregates shipment data with related orders and quotes to build Delivery views.
  - Validates request payloads, enforces ID consistency, sets Location headers on creation, and returns appropriate HTTP statuses.
  - Integrates with persistence via repository factory (Shipment, Order, Quote) and logs exceptions to Application Insights telemetry.

- Purpose: Provide the API layer that powers shipment tracking and delivery confirmation in the manufacturing order fulfillment workflow, enabling downstream processes (e.g., inventory updates, customer notifications, and integration with external systems) through reliable, validated, and observable shipment operations.

#### CatalogController.java
- Role: Spring MVC controller that exposes REST endpoints for managing the parts catalog in the Parts Unlimited MRP application’s web/service tier.

- Key Functionality: 
  - Retrieves all catalog items and individual items by SKU.
  - Adds new catalog items with validation, SKU uniqueness checks, and Location header on success.
  - Updates (upsert-style) existing items by SKU and removes items.
  - Maps repository outcomes to appropriate HTTP status codes (200/201/204/404/500).
  - Integrates with the CatalogItemsRepository via a factory and logs exceptions to Application Insights telemetry.

- Purpose: Provide the API layer for parts master data management, enabling downstream MRP workflows such as procurement, quote generation, order fulfillment, and shipment coordination by offering reliable, validated, and observable access to catalog data.

#### PingController.java
- Role: Spring MVC controller for the Ordering Service that exposes lightweight health and status endpoints used by monitors and orchestrators in the Parts Unlimited MRP system.

- Key Functionality:
  - Provides a /ping endpoint that returns HTTP 200 for basic liveness checks.
  - Provides a /status endpoint that returns configured ping/validation messages and optional build metadata (build number and timestamp) loaded from buildinfo.properties.
  - Caches build properties on first access and reports exceptions to Application Insights via TelemetryClient, returning HTTP 500 on errors.

- Purpose: Enable operational visibility and reliability for the ordering workflow by supporting health probes, deployment verification, and diagnostics (including build traceability), ensuring the service can be safely integrated and monitored within the broader MRP, inventory, and order processing ecosystem.

#### OrderController.java
- Role: Spring MVC web controller that exposes REST endpoints for managing Orders in the Parts Unlimited MRP application’s ordering service.

- Key Functionality: Retrieves orders (by ID, dealer, and status), creates orders from quotes, updates orders and order status, appends dated order events, and deletes orders. Maps domain outcomes to HTTP responses (200/201/204/400/404/409/500), sets Location headers on creation, logs unexpected errors to Application Insights via TelemetryClient, and accesses Order/Quote repositories through a RepositoryFactory.

- Purpose: Serve as the API surface for order lifecycle operations to support manufacturing workflows (order fulfillment, production coordination, and shipment tracking), bridging the web tier and persistence while ensuring consistent behavior, validation, and observability across order-related operations.

#### QuoteController.java
- Role: Spring MVC controller for the Quote resource in the Parts Unlimited MRP ordering service, acting as the REST API layer between the web tier and the data repository for quote operations.

- Key Functionality:
  - Retrieves quotes by ID and by customer name
  - Creates quotes with validation and Location header construction
  - Updates existing quotes with validation
  - Deletes quotes by ID
  - Maps outcomes to appropriate HTTP status codes (200/201/204/400/404/409/500)
  - Logs unexpected exceptions to Application Insights via TelemetryClient
  - Accesses the persistence layer through a QuoteRepository obtained from RepositoryFactory

- Purpose: Provide a reliable, observable API for managing the quote lifecycle—supporting quote creation, lookup, update, and deletion—to enable downstream MRP workflows such as order generation, inventory planning, and fulfillment. This controller enforces input validation, standardizes HTTP responses, and integrates telemetry for operational visibility.

#### DealerController.java
- Role: Spring MVC controller that exposes REST endpoints for managing dealer master data in the Parts Unlimited MRP ordering service, bridging the HTTP layer with the DealersRepository.

- Key Functionality:
  - Retrieve all dealers (GET /dealers), with an intentionally repeated repository call loop to generate load for APM/performance demonstration.
  - Retrieve a dealer by name (GET /dealers/{name}).
  - Create a new dealer with validation and duplicate checks, returning a Location header (POST /dealers).
  - Update an existing dealer with validation and existence checks (PUT /dealers/{name}).
  - Remove a dealer (DELETE /dealers/{name}).
  - Consistent HTTP status handling (200/201/204/404/409/500) and exception telemetry via Application Insights.
  - Repository access centralized through RepositoryFactory and application path construction via OrderingInitializer.

- Purpose: Maintain and govern dealer data used across manufacturing and supply chain workflows—such as procurement, order fulfillment, and shipment coordination—while providing observability hooks and (via the load-generating list call) a means to test system performance under stress.

#### OrderControllerTest.java
- Role: Integration-style JUnit test suite for the OrderController (and its interaction with QuoteController) in the Parts Unlimited MRP application.

- Key Functionality:
  - Initializes a clean, in-memory repository state for deterministic tests.
  - Verifies end-to-end order flows: create order from quote, retrieve by ID, and query by dealer name with optional status filters.
  - Asserts REST semantics: HTTP status codes, response bodies, and Location headers.
  - Validates business rules around order lifecycle: status updates recorded as events, event comment conventions, and date normalization to “today.”
  - Confirms case-insensitive dealer matching and acceptance of optional parameters.

- Purpose: Ensures the correctness and reliability of core order-processing behaviors critical to manufacturing and supply chain operations—covering quote-to-order conversion, order retrieval, lifecycle tracking via events, and dealer-centric queries—thereby safeguarding order fulfillment, status visibility, and integration expectations across the MRP system.

#### ShipmentControllerTest.java
- Role: Integration-style unit test suite for the ShipmentController that validates shipment lifecycle behavior within the Parts Unlimited MRP application, exercising interactions with QuoteController and OrderController over an in-memory repository backend.

- Key Functionality:
  - Initializes a clean, in-memory test environment by resetting all repositories and instantiating fresh controllers.
  - Verifies shipment creation preconditions (rejects when related quote/order is missing; succeeds when present).
  - Retrieves shipments globally and by order status (Created, Shipped), and verifies empty-result behavior (404 for Delivered when none).
  - Confirms shipment read/update flows (e.g., persisting contact name changes) and event handling (adding and retrieving shipment events).
  - Asserts correct HTTP semantics (201/200/400/404) and payload integrity (orderId matching, non-null events).

- Purpose: Ensure the shipment API’s correctness and business rules for order fulfillment and shipment tracking—covering quote-to-order-to-shipment progression, status filtering, and event logging—thereby safeguarding integration contracts and data integrity across inventory, orders, and shipments in the MRP workflow.

#### CatalogControllerTest.java
- Role: JUnit test suite for the CatalogController, validating catalog-related REST behaviors within the Parts Unlimited MRP application

- Key Functionality: 
  - Sets up an isolated, in-memory repository environment for deterministic tests
  - Verifies add, upsert (update), get (single and list), and remove operations for catalog items
  - Asserts correct HTTP semantics and validation rules, including:
    - 201 Created on successful add
    - 400 Bad Request for invalid input
    - 409 Conflict for duplicate SKUs
    - 404 Not Found for missing items/lists
    - 200 OK for successful retrieval/update
    - 204 No Content for successful deletion
  - Confirms state changes persist across operations (e.g., update reflects in subsequent retrieval)

- Purpose: Ensures the catalog management API enforces business rules and reliable behavior for parts inventory in MRP workflows—supporting accurate parts catalog integrity, order quoting, and downstream fulfillment—by providing fast, repeatable tests decoupled from external persistence.

#### DealerControllerTest.java
- Role: JUnit test suite for the DealerController, validating dealer CRUD REST endpoints within the Parts Unlimited MRP application’s ordering/web tier.

- Key Functionality:
  - Initializes an isolated, in-memory repository for clean test runs (RepositoryFactory.reset("memory")).
  - Verifies addDealer behavior: successful creation (201), bad requests for null/empty IDs (400), and duplicates (409).
  - Verifies updateDealer: successful update and persistence (200), invalid payload handling (400), and not-found cases (404).
  - Verifies retrieval: getDealers returns 404 when empty and 200 with a populated list after seeding; getDealer returns 404 when absent and 200 with correct data when present.
  - Verifies removeDealer: proper deletion flow with 204 and count reduction in subsequent list retrieval.
  - Provides a helper to seed four dealers for list and retrieval scenarios.

- Purpose: Ensures the correctness and robustness of dealer management APIs that underpin supplier/dealer relationships in the MRP ecosystem. By enforcing REST contract compliance and data integrity across create, read, update, and delete operations, it safeguards core workflows such as order processing, inventory coordination, and shipment orchestration that depend on accurate dealer records.

#### QuoteControllerTest.java
- Role: JUnit test suite for the QuoteController, validating the REST API behavior for quote management in the Parts Unlimited MRP ordering service.

- Key Functionality:
  - Initializes a clean, in-memory repository environment for test isolation (via RepositoryFactory.reset("memory") and TestPath.reset()).
  - Verifies quote lifecycle operations:
    - Create: supports explicit IDs, auto-generates numeric IDs for null/empty inputs, sets Location header, rejects duplicates with 400.
    - Update: returns 404 for missing quotes, 200 on successful update after creation.
    - Retrieve: 404 for non-existent IDs; 200 with body for existing quotes; filters by customer name returning 404 when empty and correct list sizes thereafter.
    - Delete: 404 when missing, 204 on success, and subsequent 404 on retrieval.
  - Asserts correct HTTP status codes and response entities throughout.

- Purpose: Ensures the QuoteController reliably enforces API contracts and edge cases critical to manufacturing quote generation and order processing workflows. By validating CRUD semantics and search behavior, it safeguards integration with the web front end and external systems, reduces regression risk, and supports consistent, predictable quote management in the MRP system.


### Package: `smpl.ordering.models`
#### QuoteItemInfo.java
- Role: Data model (value object) for a single quoted line item within the Parts Unlimited MRP system’s ordering/quote domain, used across services to represent an item on a quote by SKU and amount.

- Key Functionality:
  - Encapsulates SKU identity (skuNumber) and a numeric amount (e.g., quoted price or value).
  - Provides constructors, getters/setters for JavaBean-style use, serialization, and persistence.
  - Implements Comparable to enable sorting by SKU.
  - Defines equals/hashCode based on SKU and amount for correct behavior in collections and REST/MongoDB round-trips.

- Purpose: To standardize how quote items are represented and compared throughout quote generation, order processing, and integration workflows—facilitating consistent linking to catalog/inventory by SKU and enabling reliable calculations, deduplication, and transport between the web tier, order service, and persistence layers.

#### ShipmentEventInfo.java
- Role: Lightweight model/DTO representing a shipment event within the ordering and shipment tracking flows of the Parts Unlimited MRP system.

- Key Functionality: 
  - Stores a shipment event’s date (as a String) and human-readable comments.
  - Provides basic JavaBean constructors and getters/setters.
  - Performs minimal validation to ensure comments are present (no date format enforcement).
  - Suited for REST payloads and persistence by using simple String fields.

- Purpose: To capture and transport concise shipment timeline entries (date plus notes) that support order fulfillment tracking, customer communication, and auditing within the manufacturing and supply chain processes.

#### ShipmentRecord.java
- Role: Domain model for representing a shipment tied to an order within Parts Unlimited MRP. It encapsulates delivery details, contact info, and a timeline of shipment events used across order processing and shipment tracking workflows.

- Key Functionality: 
  - Stores core shipment attributes: orderId, deliveryDate, deliveryAddress, contactName, primary/alternate phones.
  - Manages shipment event history via a mutable List<ShipmentEventInfo> with methods to add events (by object or by date/comments).
  - Provides basic validation that aggregates missing/invalid fields (orderId, deliveryDate, deliveryAddress completeness, and contact info).
  - Supports cloning via a copy constructor (deep-copies events, shallow-copies other references).
  - Supplies standard getters/setters for serialization (e.g., REST payloads) and persistence integration.

- Purpose: Provide a simple, transportable data structure to track and validate shipment details and event timelines for order fulfillment. It enables shipment coordination, status tracking, and communication with external systems in the MRP workflow, delivering essential business value by centralizing delivery and contact information needed to execute and monitor shipments.

#### Quote.java
- Role: Domain model/DTO for a sales quote in the Parts Unlimited MRP ordering subsystem, used across web/API layers, services, and persistence.

- Key Functionality:
  - Encapsulates quote identity, parties (customer/dealer), validity window, and address (city/state/postalCode).
  - Maintains itemized quote content via a List<QuoteItemInfo> with helper to add items.
  - Carries pricing data (totalCost, discount) for totals/discount logic.
  - Provides lightweight validation of key fields (dealerName, customerName) returning a JSON error payload.
  - Supplies standard getters/setters and a copy constructor for easy transfer and cloning.
  - Implements equals/hashCode (including order-insensitive comparison of items) for reliable use in collections and caching.
  - Uses simple, serialization-friendly types suitable for REST payloads and MongoDB document mapping.

- Purpose: Represent and transport quote data through quote generation, review, and conversion to orders, enabling pricing, validation, and integration workflows essential to inventory, procurement, and order fulfillment in the MRP system.

#### PhoneInfo.java
- Role: A lightweight model/value object in the ordering domain representing a contact phone entry associated with customers, orders, quotes, or shipments within the Parts Unlimited MRP system.

- Key Functionality: 
  - Stores a phone number and its category/type (e.g., mobile, home, work).
  - Provides basic constructors plus simple getters and setters for phoneNumber and kind.
  - Acts as a serializable-friendly, persistence-ready data holder for REST payloads and MongoDB documents, without built-in validation or formatting.

- Purpose: To provide a consistent, reusable representation of phone contact information across order processing, quote generation, and shipment coordination workflows, enabling communication (notifications, coordination) with customers, suppliers, and logistics partners in the MRP ecosystem.

#### Order.java
- Role: Domain model for representing an order in the Parts Unlimited MRP system; serves as a core DTO/entity across web, service, and integration layers.

- Key Functionality:
  - Captures key order attributes: orderId, quoteId, orderDate, status, and a sequenced list of lifecycle events.
  - Provides basic validation of required string fields (quoteId, orderDate), emitting a JSON-formatted error payload.
  - Maintains an event history (audit trail) via a mutable List, with support to append events.
  - Implements equals and hashCode for use in collections and cross-object comparisons.
  - Exposes standard getters/setters for serialization, persistence, and API interactions.

- Purpose: Enable consistent representation, validation, and tracking of an order’s lifecycle for manufacturing workflows (procurement, order fulfillment, shipment coordination). Acts as the foundational object linking quotes, shipments, and status progression, supporting API payloads, MongoDB persistence, and auditing within the MRP application.

#### DeliveryAddress.java
- Role: A simple address model (JavaBean/DTO) used by the ordering/shipping components to capture and exchange delivery address details within the Parts Unlimited MRP system.

- Key Functionality: 
  - Encapsulates delivery fields: street, city, state, postalCode, and specialInstructions.
  - Provides no-arg and full constructors to support framework serialization/deserialization (e.g., REST JSON, MongoDB).
  - Exposes standard getters/setters for all fields.
  - Offers minimal validation via validate(), ensuring only city and postalCode are non-empty (per a basic isNotEmpty check).

- Purpose: Standardize how delivery information and special handling notes are represented across order processing, quote generation, and shipment coordination. This enables consistent persistence, API payloads, shipping label generation, and downstream logistics logic, contributing to reliable order fulfillment workflows in the MRP application.

#### QuoteItemInfo.java
- Role: Data model (value object) for a single quoted line item within the Parts Unlimited MRP system’s ordering/quote domain, used across services to represent an item on a quote by SKU and amount.

- Key Functionality:
  - Encapsulates SKU identity (skuNumber) and a numeric amount (e.g., quoted price or value).
  - Provides constructors, getters/setters for JavaBean-style use, serialization, and persistence.
  - Implements Comparable to enable sorting by SKU.
  - Defines equals/hashCode based on SKU and amount for correct behavior in collections and REST/MongoDB round-trips.

- Purpose: To standardize how quote items are represented and compared throughout quote generation, order processing, and integration workflows—facilitating consistent linking to catalog/inventory by SKU and enabling reliable calculations, deduplication, and transport between the web tier, order service, and persistence layers.

#### OrderStatus.java
- Role: Central enumeration defining the authoritative lifecycle states of an order across the Parts Unlimited MRP system.

- Key Functionality: 
  - Provides type-safe order status values: None, Created, Confirmed, Started, Built, DeliveryConfirmed, Shipped, Delivered, Installed.
  - Enables consistent use in business logic (switches, validations, transitions), persistence, queries, and UI progress indicators.
  - Supports efficient collections and stable keys via standard Java enum capabilities (values(), valueOf(), EnumSet/EnumMap).

- Purpose: Establish a consistent, constrained state model for order processing—from creation and production through shipping, delivery, and installation—facilitating validation, workflow orchestration, reporting, and integration across services (web front end, order service, integration service) within the manufacturing and supply chain context.

#### ShipmentEventInfo.java
- Role: Lightweight model/DTO representing a shipment event within the ordering and shipment tracking flows of the Parts Unlimited MRP system.

- Key Functionality: 
  - Stores a shipment event’s date (as a String) and human-readable comments.
  - Provides basic JavaBean constructors and getters/setters.
  - Performs minimal validation to ensure comments are present (no date format enforcement).
  - Suited for REST payloads and persistence by using simple String fields.

- Purpose: To capture and transport concise shipment timeline entries (date plus notes) that support order fulfillment tracking, customer communication, and auditing within the manufacturing and supply chain processes.

#### ShipmentRecord.java
- Role: Domain model for representing a shipment tied to an order within Parts Unlimited MRP. It encapsulates delivery details, contact info, and a timeline of shipment events used across order processing and shipment tracking workflows.

- Key Functionality: 
  - Stores core shipment attributes: orderId, deliveryDate, deliveryAddress, contactName, primary/alternate phones.
  - Manages shipment event history via a mutable List<ShipmentEventInfo> with methods to add events (by object or by date/comments).
  - Provides basic validation that aggregates missing/invalid fields (orderId, deliveryDate, deliveryAddress completeness, and contact info).
  - Supports cloning via a copy constructor (deep-copies events, shallow-copies other references).
  - Supplies standard getters/setters for serialization (e.g., REST payloads) and persistence integration.

- Purpose: Provide a simple, transportable data structure to track and validate shipment details and event timelines for order fulfillment. It enables shipment coordination, status tracking, and communication with external systems in the MRP workflow, delivering essential business value by centralizing delivery and contact information needed to execute and monitor shipments.

#### Quote.java
- Role: Domain model/DTO for a sales quote in the Parts Unlimited MRP ordering subsystem, used across web/API layers, services, and persistence.

- Key Functionality:
  - Encapsulates quote identity, parties (customer/dealer), validity window, and address (city/state/postalCode).
  - Maintains itemized quote content via a List<QuoteItemInfo> with helper to add items.
  - Carries pricing data (totalCost, discount) for totals/discount logic.
  - Provides lightweight validation of key fields (dealerName, customerName) returning a JSON error payload.
  - Supplies standard getters/setters and a copy constructor for easy transfer and cloning.
  - Implements equals/hashCode (including order-insensitive comparison of items) for reliable use in collections and caching.
  - Uses simple, serialization-friendly types suitable for REST payloads and MongoDB document mapping.

- Purpose: Represent and transport quote data through quote generation, review, and conversion to orders, enabling pricing, validation, and integration workflows essential to inventory, procurement, and order fulfillment in the MRP system.

#### Delivery.java
- Role: Domain model/DTO representing a Delivery aggregate in the ordering subsystem, linking pricing (Quote), transaction (Order), and logistics (ShipmentRecord) for fulfillment and tracking.

- Key Functionality:
  - Holds one-to-one associations to a Quote, an Order, and a ShipmentRecord.
  - Provides simple getters/setters to read and update these references for use across services, persistence, and REST APIs.
  - Serves as the container through which fulfillment workflows access pricing context, order details, and shipment state.

- Purpose: To centralize and serialize the key artifacts of the delivery phase—quoted pricing, the originating order, and shipment tracking—so MRP workflows (order processing, shipment coordination, and external integration) can reliably coordinate fulfillment, maintain traceability, and support end-to-end shipment tracking within Parts Unlimited MRP.

#### PhoneInfo.java
- Role: A lightweight model/value object in the ordering domain representing a contact phone entry associated with customers, orders, quotes, or shipments within the Parts Unlimited MRP system.

- Key Functionality: 
  - Stores a phone number and its category/type (e.g., mobile, home, work).
  - Provides basic constructors plus simple getters and setters for phoneNumber and kind.
  - Acts as a serializable-friendly, persistence-ready data holder for REST payloads and MongoDB documents, without built-in validation or formatting.

- Purpose: To provide a consistent, reusable representation of phone contact information across order processing, quote generation, and shipment coordination workflows, enabling communication (notifications, coordination) with customers, suppliers, and logistics partners in the MRP ecosystem.

#### OrderEventInfo.java
- Role: Lightweight model/DTO representing an event or note on an order within the ordering domain, used to record timeline entries (e.g., status changes, remarks) for MRP order processing.

- Key Functionality:
  - Captures an event date (as a String) and free-form comments tied to an order.
  - Provides constructors for empty, “now + comments,” and explicit date + comments initialization (uses locale-specific SHORT date formatting for the “now” case).
  - Simple getters/setters enabling serialization to/from REST/MongoDB and display in the UI.

- Purpose: Enable audit/history tracking and user-visible context for orders by persisting and transporting dated comments across the system. This supports order fulfillment transparency, customer service inquiries, and operational reporting within Parts Unlimited MRP.

#### Order.java
- Role: Domain model for representing an order in the Parts Unlimited MRP system; serves as a core DTO/entity across web, service, and integration layers.

- Key Functionality:
  - Captures key order attributes: orderId, quoteId, orderDate, status, and a sequenced list of lifecycle events.
  - Provides basic validation of required string fields (quoteId, orderDate), emitting a JSON-formatted error payload.
  - Maintains an event history (audit trail) via a mutable List, with support to append events.
  - Implements equals and hashCode for use in collections and cross-object comparisons.
  - Exposes standard getters/setters for serialization, persistence, and API interactions.

- Purpose: Enable consistent representation, validation, and tracking of an order’s lifecycle for manufacturing workflows (procurement, order fulfillment, shipment coordination). Acts as the foundational object linking quotes, shipments, and status progression, supporting API payloads, MongoDB persistence, and auditing within the MRP application.

#### DealerInfo.java
- Role: Domain model/DTO representing a dealer’s contact information within the ordering component of the Parts Unlimited MRP system.

- Key Functionality: 
  - Encapsulates dealer attributes (name, contact, address, email, phone) with standard getters/setters.
  - Provides multiple constructors, including a copy constructor, for flexible instantiation.
  - Offers basic validation of the name field, returning a JSON-formatted error payload via Utility.validateStringField.
  - Suited for serialization/deserialization in REST APIs and persistence (e.g., MongoDB) as a simple POJO.

- Purpose: Supplies a consistent, lightweight container for dealer details used across order processing, quote generation, and shipment coordination. It improves data quality and interoperability by centralizing dealer/contact data, enabling reliable communication, tracking, and integration with external systems in the MRP workflow.

#### DeliveryAddress.java
- Role: A simple address model (JavaBean/DTO) used by the ordering/shipping components to capture and exchange delivery address details within the Parts Unlimited MRP system.

- Key Functionality: 
  - Encapsulates delivery fields: street, city, state, postalCode, and specialInstructions.
  - Provides no-arg and full constructors to support framework serialization/deserialization (e.g., REST JSON, MongoDB).
  - Exposes standard getters/setters for all fields.
  - Offers minimal validation via validate(), ensuring only city and postalCode are non-empty (per a basic isNotEmpty check).

- Purpose: Standardize how delivery information and special handling notes are represented across order processing, quote generation, and shipment coordination. This enables consistent persistence, API payloads, shipping label generation, and downstream logistics logic, contributing to reliable order fulfillment workflows in the MRP application.

#### OrderUpdateInfo.java
- Role: Lightweight model/DTO representing an order status update and its accompanying event context within the Parts Unlimited MRP order domain.

- Key Functionality:
  - Encapsulates the current OrderStatus for an order.
  - Holds OrderEventInfo (timestamp and comments) describing the update event.
  - Provides simple constructors (including one that timestamps the event) and getters/setters to pass update data through services, REST APIs, and persistence layers.

- Purpose: Standardize how order lifecycle changes (e.g., created, updated, shipped, canceled) and their context are conveyed across the system. This supports workflow progression, auditing, notifications, and integration with downstream services (inventory, billing, shipping) in manufacturing order processing, while keeping business rules/enforcement centralized elsewhere.


### Package: `smpl.ordering.repositories`
#### ShipmentRepository.java
- Role: Persistence boundary and contract for managing shipment data within the Parts Unlimited MRP system. It abstracts storage details and standardizes how services interact with shipment records throughout the fulfillment and tracking lifecycle.

- Key Functionality:
  - Retrieve shipments by status and by unique identifier.
  - Create new shipment records with validation (BadRequestException on invalid input).
  - Append shipment events (e.g., status updates, scans) to maintain a shipment history.
  - Update existing shipment records.
  - Remove shipments with optimistic concurrency control via eTag.

- Purpose: Enable consistent, testable, and decoupled access to shipment data across order processing, fulfillment, and tracking workflows. This interface supports business needs such as shipment visibility, auditability through event logs, and safe concurrent updates, while allowing flexible backend implementations (e.g., MongoDB) and smooth integration with other services in the MRP ecosystem.

#### QuoteRepository.java
- Role: Repository interface defining data-access operations for Quote entities within the Parts Unlimited MRP system

- Key Functionality:
  - Retrieve a quote by ID and list quotes by customer name
  - Fetch quote identifiers by dealer name for lightweight lookups
  - Create quotes with input validation (BadRequestException on invalid data)
  - Update and delete quotes using optimistic concurrency control via eTag
  - Provide a stable API for different persistence implementations (e.g., MongoDB-backed or in-memory)

- Purpose: To abstract and standardize quote persistence for the quote generation and sales pipeline, enabling reliable creation, lookup, and lifecycle management of quotes used in order processing and production planning. This supports efficient sales operations (by customer and dealer views), safeguards data integrity with concurrency checks, and decouples business logic from storage details, improving maintainability and scalability across the MRP application.

#### CatalogItemsRepository.java
- Role: Repository interface for managing CatalogItem entities (parts in the catalog) within the Parts Unlimited MRP application, abstracting data access for the web and service layers.
- Key Functionality: 
  - Retrieve all catalog items and fetch a single item by SKU.
  - Upsert (create or update) items by SKU with optimistic concurrency via eTag.
  - Remove items conditionally using eTag to prevent conflicting deletes.
- Purpose: Provide a clean, implementation-agnostic contract for catalog data operations that supports safe concurrent updates, enabling reliable parts catalog management for quoting, order processing, procurement, and inventory workflows across the MRP system.

#### OrderRepository.java
- Role: Defines the persistence contract for Order entities in the Order Service, acting as the repository-layer interface that decouples business logic from the underlying data store (e.g., MongoDB) within the Parts Unlimited MRP system.

- Key Functionality:
  - Existence checks and retrieval of orders by ID and by associated quote ID.
  - Filtered queries by order status and by dealer name plus status.
  - Creation of orders (from a source such as a quote), with domain validation via BadRequestException.
  - Full (PUT-style) and partial (PATCH-style) updates using Order and OrderUpdateInfo.
  - Optimistic concurrency control using eTags on update and delete operations.
  - Deletion of orders with conditional (eTag) safeguards.

- Purpose: Provide a consistent, testable abstraction for order lifecycle management—supporting order creation from quotes, status tracking, fulfillment, and integration workflows—while enforcing data integrity and concurrency controls critical to manufacturing, inventory, and shipment coordination processes.

#### DealersRepository.java
- Role: Repository interface for managing DealerInfo entities, abstracting data access for dealer/supplier records within the Parts Unlimited MRP system.

- Key Functionality: 
  - Retrieval of all dealers and lookup by unique dealer name.
  - Create/update (upsert) and delete operations guarded by eTag-based optimistic concurrency.
  - Provides a storage-agnostic contract (e.g., MongoDB-backed or in-memory) for use by services handling catalog, orders, quotes, and shipments.

- Purpose: Enable consistent, safe management of dealer/supplier data to support manufacturing workflows—such as quote generation, order fulfillment, and shipment coordination—while decoupling business logic from persistence and facilitating testing and integration across the application.

#### RepositoryFactory.java
- Role: Central repository factory/service locator that provides storage-aware repositories for core MRP domains (catalog items, dealers, quotes, orders, shipments) across the Parts Unlimited MRP application.

- Key Functionality:
  - Selects repository implementations based on configured storage kind (in-memory mock vs. MongoDB) via static accessors (getCatalogItemsRepository, getDealersRepository, getOrderRepository, getQuoteRepository, getShipmentRepository).
  - Initializes complete repository graphs for both backends and wires inter-repository dependencies; optionally retrieves a MongoTemplate from the Spring context.
  - Manages a global singleton instance with getFactory/reset and exposes storage identifiers (MEMORY, MONGODB).
  - Encapsulates and shares repository instances application-wide.
  - Notes: relies on global state, may return null for unsupported kinds, can throw NPE if uninitialized, swallows MongoTemplate acquisition errors, and has thread-safety/resource-lifecycle caveats.

- Purpose: Provide a consistent, switchable data-access layer across environments (test/in-memory vs. production/MongoDB) to support manufacturing workflows—inventory management, order processing, quote generation, and shipment tracking—without coupling business logic to a specific persistence backend.

#### QuoteRepositoryTest.java
- Role: JUnit test suite for the Quote repository layer, validating CRUD and query behavior for quote entities within the Parts Unlimited MRP system.

- Key Functionality:
  - Seeds deterministic test data by resetting repositories and inserting known dealers, catalog items, and a set of sample quotes.
  - Verifies retrieving a single quote (existent vs. non-existent), including content validation (IDs, customer names, items).
  - Tests querying quotes by customer name for exact, partial, and no-match scenarios.
  - Ensures quote creation works and that duplicates are rejected with a BadRequestException.
  - Validates update behavior (success/failure, persistence of changes).
  - Confirms delete behavior (success/failure) and downstream effects on query results.
  - Exercises repository methods’ handling of optional parameters (e.g., null context) to mirror real service usage.

- Purpose: To ensure the reliability and correctness of quote management operations—central to pricing, order capture, and downstream fulfillment—in the MRP application. By rigorously testing quote lifecycle behaviors against known fixtures and repository contracts, it safeguards business processes like quote generation, inventory-aware sales, and integration with dealer and catalog data, preventing regressions in core supply chain workflows.

#### OrderRepositoryTest.java
- Role: JUnit test suite for validating the OrderRepository in the Parts Unlimited MRP system, ensuring correct behavior across the order lifecycle and its integration with related repositories.

- Key Functionality:
  - Initializes a deterministic test environment by resetting repositories and seeding dealers, catalog items, quotes, and corresponding orders.
  - Verifies core repository operations: existence checks, retrieval by order ID and quote ID, and filtered queries by status and dealer.
  - Tests order creation from quotes, including duplicate-protection semantics.
  - Validates order updates, including status transitions (e.g., Created to Confirmed) and persistence of event history (date/comments).
  - Ensures cross-repository consistency using RepositoryFactory and TestPath reset mechanisms.

- Purpose: To guarantee the reliability and integrity of order persistence and querying in support of manufacturing workflows—such as order processing, fulfillment, and shipment coordination—by preventing regressions and confirming that orders, quotes, and dealer associations behave correctly under realistic scenarios.

#### CatalogItemsRepositoryTest.java
- Role: JUnit test class validating the CatalogItemsRepository (catalog data access layer) within the Parts Unlimited MRP system.

- Key Functionality:
  - Establishes a clean test fixture by resetting the repository (via TestPath) and seeding three known catalog items.
  - Verifies read operations: non-null retrieval, stable ordering by SKU, correct list sizing, and accurate pricing for specific SKUs.
  - Validates write operations: upsert behavior (true indicates update of an existing item; false indicates insertion of a new item), append behavior on insert, and item removal semantics (returns true for existing, false for non-existent; supports null audit parameter).
  - Confirms repository state transitions across CRUD operations to ensure predictable behavior.

- Purpose: Provides regression coverage for the catalog repository that underpins core MRP workflows—inventory accuracy, quote generation, order processing, and shipment coordination—by ensuring the catalog’s data integrity, ordering, and update semantics behave as expected.

#### DealersRepositoryTest.java
- Role: JUnit test suite that defines and verifies the contract for the DealersRepository data-access component, ensuring reliable CRUD behavior for dealer records within the Parts Unlimited MRP system.

- Key Functionality:
  - Resets repository state and seeds known test data via TestPath.reset and upsert operations.
  - Validates retrieval of all dealers with deterministic ordering and expected field values.
  - Verifies single-dealer lookups for existing and non-existing IDs.
  - Tests upsert semantics (true on update, false on insert) and persistence of changes.
  - Confirms correct deletion behavior, idempotency on repeated removes, and list size updates.
  - Provides a helper to generate consistent DealerInfo test objects.

- Purpose: To safeguard the integrity of dealer management in the manufacturing/supply chain workflow—supporting supplier/partner data used by catalog, orders, quotes, and shipments—by catching regressions in the MongoDB-backed repository and ensuring predictable interactions for higher-level MRP services.

#### ShipmentRepositoryTest.java
- Role: Integration-focused JUnit test suite for the ShipmentRepository, validating shipment behaviors within the Parts Unlimited MRP stack and its interactions with orders, quotes, dealers, and catalog data.

- Key Functionality:
  - Establishes a clean, deterministic test fixture by resetting repositories and seeding dealers, catalog items, quotes, orders, and initial shipments (first five orders marked Shipped).
  - Verifies shipment retrieval and filtering via getShipments by OrderStatus (None, Shipped, Delivered).
  - Ensures getShipmentById returns initialized records with proper event collections.
  - Confirms business rule enforcement: one shipment per order (duplicates throw BadRequestException).
  - Validates creation and update flows: successful shipment creation links to the correct order and increases counts; updateShipment and addEvent persist and reflect shipment event history.
  - Provides a helper to build standardized ShipmentRecord instances with predefined contact and delivery details.

- Purpose: To safeguard core fulfillment and shipment coordination workflows in the MRP system by ensuring the ShipmentRepository correctly manages shipment creation, retrieval, status-based queries, and event tracking, and that it maintains integrity with order state and business rules. This promotes reliability in inventory, order fulfillment, and shipment tracking processes.


### Package: `smpl.ordering.repositories.mock`
#### MockCatalogItemsRepository.java
- Role: In-memory mock implementation of the catalog repository used by the ordering/MRP services to manage parts data without relying on MongoDB or external systems. It supports development, demos, and tests for catalog-dependent features.

- Key Functionality:
  - Seeds a small sample catalog (e.g., brake components) for immediate use.
  - Provides read operations with defensive copies (list all items, fetch by SKU).
  - Supports write operations: upsert (update or insert), remove by SKU, and full reset.
  - Performs case-insensitive SKU matching.
  - Encapsulates internal state via a private, final list.

- Purpose: Enable rapid, database-free testing and prototyping of catalog-driven workflows such as quote generation, order processing, and inventory checks within the Parts Unlimited MRP application. It delivers basic CRUD-like behavior for parts catalog management while intentionally omitting concurrency control (eTag ignored) and persistence to keep the implementation lightweight.

#### MockShipmentRepository.java
- Role: In-memory mock implementation of the ShipmentRepository used to manage shipment records for orders within the Parts Unlimited MRP system, primarily for testing and lightweight runtime scenarios without a backing database.

- Key Functionality:
  - Retrieves shipments filtered by associated order status (including “all” via OrderStatus.None), using OrderRepository to resolve order state.
  - Looks up a shipment by orderId, returning defensive copies to prevent external mutation.
  - Creates new shipment records only when the referenced order exists and no shipment already exists, enforcing one shipment per order; throws BadRequestException on violations.
  - Adds shipment events (date/comments) to existing shipment records.
  - Updates shipment records by replacing them by orderId.
  - Resets the repository by clearing all in-memory records.
  - Includes a stubbed removeShipment method (currently non-functional).
  - Employs defensive copying but lacks synchronization and robust null checks, appropriate for a mock context.

- Purpose: Provide a simple, dependency-injected, in-memory shipment store that supports shipment tracking workflows (creation, event logging, querying by order status) and validates against orders, enabling unit/integration tests and development of MRP features (order fulfillment and shipment coordination) without requiring persistent storage.

#### MockOrderRepository.java
- Role: In-memory mock implementation of the OrderRepository used by the Parts Unlimited MRP system to simulate order persistence and behavior during development and testing, with integration to a QuoteRepository for quote-to-order workflows.

- Key Functionality: 
  - Stores orders in an internal list and supports lookups by order ID and quote ID.
  - Filters orders by status and by dealer name (via associated quotes).
  - Creates orders from quotes with basic validation (missing quote -> BadRequest, duplicate order for quote -> ConflictingRequest).
  - Updates orders either by full replacement or by appending events and changing status.
  - Provides a reset method to clear state; includes a static counter placeholder.
  - Includes a stubbed delete operation and ignores eTag/concurrency controls.

- Purpose: Enable rapid, dependency-free testing of core order management flows—quote-to-order creation, status progression, event tracking, and retrieval/filtering—without requiring MongoDB or full infrastructure, thereby speeding up development and validating business logic for MRP order processing.

#### MockQuoteRepository.java
- Role: In-memory mock repository for managing Quote entities within the ordering subsystem, used to simulate persistence and dealer linkage without hitting MongoDB.

- Key Functionality: 
  - CRUD operations for quotes (create with auto-ID, read by ID, update, remove).
  - Search utilities (filter quotes by customer name, list quote IDs by dealer name).
  - Dealer validation/upsert via DealersRepository during create/update.
  - State management utilities (reset) and simple ID generation using a shared Random.

- Purpose: Provide a lightweight, test-friendly substitute for the real QuoteRepository to support development, unit tests, and local runs of Parts Unlimited MRP workflows (quote generation and dealer association) without external dependencies, enabling rapid iteration on order and quoting features.

#### MockDealersRepository.java
- Role: In-memory mock implementation of the DealersRepository used by the Parts Unlimited MRP system to simulate dealer data storage for testing and development without a real database.

- Key Functionality:
  - Stores DealerInfo objects in a private, mutable list.
  - Retrieves all dealers as defensive copies to protect internal state.
  - Finds a dealer by name (case-insensitive) and returns a defensive copy.
  - Upserts a dealer by name (update if exists, insert if not), with an unused eTag parameter.
  - Removes a dealer by name and supports full reset of the repository.

- Purpose: Provide a lightweight, deterministic repository for dealer/supplier management workflows (e.g., catalog lookup, order routing, quote generation) enabling unit tests, integration tests, and local development to run without MongoDB or external dependencies, while preserving repository interface contracts.


### Package: `smpl.ordering.repositories.mock.test`
#### MockCatalogItemsRepositoryTest.java
- Role: JUnit test subclass that validates the in-memory/mock Catalog Items repository implementation used by the Parts Unlimited MRP system.

- Key Functionality:
  - Configures RepositoryFactory to use an in-memory repository for tests.
  - Reuses and executes the base repository tests (get all items, get single item, upsert, remove) in the mock context.
  - Ensures behavior parity between repository implementations without external dependencies.

- Purpose: Provide fast, isolated verification of catalog data operations critical to MRP workflows (parts catalog management, order/quote processing). This safeguards core inventory and catalog consistency while speeding regression testing and enabling reliable CI without requiring MongoDB or other infrastructure.

#### MockDealersRepositoryTest.java
- Role: A test harness that runs the shared DealersRepository tests against an in-memory/mock backend, ensuring repository behavior is consistent without relying on external persistence.

- Key Functionality:
  - Resets the RepositoryFactory to use a “memory” implementation prior to each test run.
  - Delegates core CRUD tests (get all dealers, get a dealer, upsert, remove) to the base test suite.
  - Validates repository semantics independent of MongoDB or other real storage.

- Purpose: To provide fast, isolated verification of dealer data access logic critical to MRP workflows (e.g., order processing, quote generation, and shipment coordination). By ensuring the DealersRepository behaves correctly with a mock backend, it safeguards higher-level manufacturing and supply chain operations that depend on accurate dealer/supplier information.

#### MockQuoteRepositoryTest.java
- Role: Concrete test harness for the Quote repository using an in-memory/mock backend, validating repository behavior for quote operations within the Parts Unlimited MRP system.

- Key Functionality: 
  - Forces repository configuration to an in-memory implementation via RepositoryFactory.reset("memory").
  - Reuses and executes inherited tests for quote retrieval, customer-specific queries, creation, update, and removal.
  - Ensures the in-memory repository conforms to the repository contract without adding new test logic.

- Purpose: Provide fast, deterministic validation of quote management persistence behavior (e.g., quote generation and lookup by customer) independent of external databases. This supports reliable CI for manufacturing workflows involving pricing and quoting, reinforcing inventory and order processing integrity in the MRP domain.

#### MockShipmentRepositoryTest.java
- Role: JUnit subclass that runs the base ShipmentRepository tests against an in-memory/mock repository implementation.

- Key Functionality:
  - Switches the repository backend to an in-memory “memory” implementation via RepositoryFactory.reset("memory") during setup.
  - Reuses and delegates all shipment-related test cases (get by id, list, create, update, add event) from the superclass to validate repository behavior without external dependencies.

- Purpose: Ensure fast, deterministic, and isolated verification of shipment persistence and event handling logic for the MRP system. By exercising the shipment repository using an in-memory backend, it provides reliable regression coverage for order fulfillment and shipment tracking workflows without requiring MongoDB or external services, improving CI stability and developer feedback cycles.

#### MockOrderRepositoryTest.java
- Role: JUnit test harness that binds the base OrderRepository test suite to an in-memory/mock implementation, validating repository behavior for the Order domain within the MRP system.

- Key Functionality: 
  - Configures RepositoryFactory to use an in-memory backend for isolated, fast tests.
  - Delegates a comprehensive set of order-related tests (existence checks, retrieval, queries by quote ID/status/dealer name, creation, and updates) to the superclass, ensuring the mock repository adheres to the repository contract.

- Purpose: Ensure reliable and deterministic verification of order data-access behavior—critical for order processing and fulfillment workflows—without relying on MongoDB or external systems. This supports CI, speeds development feedback, and guards against regressions in the mock repository used across the Parts Unlimited MRP application.


### Package: `smpl.ordering.repositories.mongodb`
#### MongoShipmentRepository.java
- Role: MongoDB-backed shipment repository for the ordering/shipment module, bridging domain shipment models with persistent storage and coordinating with the OrderRepository.

- Key Functionality: 
  - Retrieves shipments by related order status and by orderId.
  - Creates shipments with integrity checks (order existence, no duplicate shipment per order).
  - Updates shipments and appends shipment events, preserving document identity.
  - Deletes shipments by orderId and supports full collection reset.
  - Converts between persistence (ShipmentDetails) and domain (ShipmentRecord) models and uses a retry-capable Mongo operations layer.

- Purpose: Provide reliable, order-aware shipment persistence and retrieval to support MRP workflows such as order fulfillment and shipment tracking, enforcing basic business rules and enabling the web/API layers to manage shipment lifecycle consistently.

#### MongoDealersRepository.java
- Role: MongoDB-backed repository for managing dealer master data within the Parts Unlimited MRP system, providing the data-access layer for dealer entities used across ordering, quoting, and shipment workflows.

- Key Functionality:
  - Retrieve all dealers and map them to DealerInfo DTOs (getDealers).
  - Fetch a single dealer by exact name (getDealer).
  - Upsert dealer records by name, updating if existing or inserting if new (upsertDealer).
  - Remove a dealer by name (removeDealer).
  - Reset dealer data by dropping the “dealers” collection (reset).
  - Uses a MongoTemplate wrapped with retry-capable operations for resilience; performs entity-to-DTO conversion.

- Purpose: Provide reliable, straightforward CRUD-style persistence for dealer information to support core manufacturing workflows—such as order processing, quote generation, and shipment coordination—by maintaining accurate dealer master data. It enables integration with MongoDB while keeping business logic decoupled via DTOs, though it offers basic operations without pagination or concurrency control.

#### MongoQuoteRepository.java
- Role: MongoDB-backed repository for Quote entities in the Parts Unlimited MRP order service, bridging domain models (Quote) with persistence models (QuoteDetails) and coordinating with dealer data.

- Key Functionality:
  - Retrieve and map quotes: getQuote by ID, helper findExistingQuote, and conversions between QuoteDetails and Quote.
  - Query support: getQuotesByCustomerName (in-memory filter over all quotes) and getQuoteIdsByDealerName (exact match query).
  - Lifecycle operations: createQuote (ensures dealer exists, generates or validates quoteId, persists), updateQuote (upserts dealer, saves changes, ignores eTag), removeQuote (delete by quoteId).
  - Administration: reset drops the quotes collection.
  - Infrastructure: uses a retry-capable MongoOperations wrapper and a class-wide Random for ID generation; collaborates with DealersRepository to upsert dealer records.

- Purpose: Provide the data-access layer enabling quote creation, retrieval, update, and deletion to support quoting and order workflows in the manufacturing/supply chain domain. It enforces basic dealer linkage, persists quote details to MongoDB, and supplies query capabilities needed by REST APIs and services in the MRP system.

#### MongoOperationsWithRetry.java
- Role: A resilience and observability wrapper around Spring Data MongoDB (MongoOperations) used by the MRP system’s services to access MongoDB reliably.

- Key Functionality:
  - Delegates most MongoDB operations (queries, aggregations, map-reduce, geo, CRUD, index and collection management) to an underlying MongoOperations instance.
  - Adds limited retry behavior for specific operations on socket timeouts (e.g., find/findOne/findAll, save/insert, dropCollection, exists, findAndRemove).
  - Emits Application Insights RemoteDependency telemetry for selected operations to track duration and success (MongoDB.* events).
  - Preserves full MongoDB feature surface via thin pass-throughs (execute commands/queries, aggregation pipelines, upsert/update/remove, index ops, converter access).

- Purpose: Provide a centralized, instrumented MongoDB access layer that improves robustness and visibility of data operations across Parts Unlimited MRP (inventory, orders, shipments, quotes). By wrapping MongoOperations with targeted retries and telemetry, it helps stabilize critical manufacturing workflows and enables monitoring of persistence performance and reliability.

#### MongoOrderRepository.java
- Role: MongoDB-backed repository for orders that bridges the domain model (Order) with persistence entities (OrderDetails) in the Parts Unlimited MRP system’s order service.

- Key Functionality:
  - Read operations: existence checks, fetch by orderId, quoteId, status, and dealer name (via QuoteRepository), and list-all.
  - Write operations: create an order from a quote (with validation to prevent quote reuse), update orders (including status and event append), and delete by orderId.
  - Data mapping and persistence: converts between Order and OrderDetails and performs MongoDB queries/inserts/saves/removes via MongoOperations (with retry).
  - Utilities: collection reset and internal counter reinitialization.

- Purpose: Provide a reliable, centralized data-access layer for managing the order lifecycle—turning quotes into orders, updating order progress, and enabling operational queries—supporting core manufacturing workflows such as order fulfillment and integration across services in the MRP application.

#### MongoCatalogItemsRepository.java
- Role: MongoDB-backed repository for the Parts Unlimited MRP catalog, acting as the data access bridge between the domain/API model and the persistence model for catalog items.

- Key Functionality: 
  - Retrieves all catalog items and single items by SKU, converting persistence entities to API models.
  - Performs upsert (insert or update) operations by preserving existing IDs, and removes items by SKU.
  - Provides a destructive reset (drop collection) for test/setup scenarios.
  - Implements queries via Spring Data MongoOperations (with a retry-enabled wrapper) and handles model mapping between repository and API layers.

- Purpose: Centralizes catalog data persistence to support inventory, pricing/quotes, procurement, and order processing workflows in the MRP system, providing consistent CRUD access to product master data stored in MongoDB (note: eTag parameters are currently ignored, so no optimistic concurrency is enforced).


### Package: `smpl.ordering.repositories.mongodb.models`
#### CatalogItem.java
- Role: MongoDB persistence model for catalog items that bridges the Spring Data repository layer and the domain model in the Parts Unlimited MRP system.

- Key Functionality: 
  - Defines stored fields for id, skuNumber, description, price, inventory, and leadTime with Spring Data document/ID/index support.
  - Provides constructors including a cross-layer copy-from constructor to initialize from the domain model.
  - Converts back to the domain model via toCatalogItem, computing an effective lead time (0 when inventory is available, otherwise the stored lead time).
  - Exposes basic accessors (e.g., id getter/setter) for persistence identity.

- Purpose: Persist and translate SKU data used across catalog management, quoting, order processing, and shipment coordination, ensuring accurate availability and lead-time semantics that drive MRP decisions such as pricing, fulfillment timing, and procurement planning.

#### CatalogItem.java
- Role: MongoDB persistence model for catalog items that bridges the Spring Data repository layer and the domain model in the Parts Unlimited MRP system.

- Key Functionality: 
  - Defines stored fields for id, skuNumber, description, price, inventory, and leadTime with Spring Data document/ID/index support.
  - Provides constructors including a cross-layer copy-from constructor to initialize from the domain model.
  - Converts back to the domain model via toCatalogItem, computing an effective lead time (0 when inventory is available, otherwise the stored lead time).
  - Exposes basic accessors (e.g., id getter/setter) for persistence identity.

- Purpose: Persist and translate SKU data used across catalog management, quoting, order processing, and shipment coordination, ensuring accurate availability and lead-time semantics that drive MRP decisions such as pricing, fulfillment timing, and procurement planning.

#### QuoteDetails.java
- Role: MongoDB persistence model for quotes in the ordering/quote subsystem, acting as the storage-facing representation of a Quote and bridging between Spring Data MongoDB and the domain model.
- Key Functionality: Captures quote identity and metadata (quoteId, validity, customer/dealer, address), pricing (totalCost, discount), and line items; provides constructors to build from a domain Quote and a toQuote mapper to reconstruct the domain object; maintains a non-null items array; exposes standard getters/setters and an id for persistence (with Spring Data annotations).
- Purpose: Enable reliable storage, retrieval, and transformation of quote data to support MRP workflows like quote generation, pricing, and downstream order fulfillment and shipment coordination, ensuring efficient repository access and clean separation between persistence and domain logic.

#### OrderDetails.java
- Role: MongoDB persistence model for order records, bridging the domain Order entity and the database within the Parts Unlimited MRP ordering service.

- Key Functionality: 
  - Stores core order data (id, orderId, quoteId, orderDate, status) and the event history.
  - Constructs a persistence document from a domain Order and converts back via toOrder.
  - Ensures a non-null events array when copying from the domain model.
  - Exposes basic id accessors for Spring Data MongoDB (@Id/@Document) usage and indexing.

- Purpose: Provide a lightweight, serializable representation of orders tailored for MongoDB, enabling reliable storage, retrieval, and reconstruction of domain Order objects to support order processing, tracking, and audit workflows across the MRP system.

#### Dealer.java
- Role: MongoDB persistence model for dealer entities within the ordering service, bridging database documents and service-level DTOs (DealerInfo) in the Parts Unlimited MRP system.

- Key Functionality: 
  - Defines the stored structure of a dealer record (id, name, contact, address, email, phone) for Spring Data MongoDB.
  - Provides construction from a DealerInfo DTO and conversion back via toDealerInfo, enabling clean separation between persistence and service layers.
  - Exposes basic identifier accessors to support repository operations and API/resource correlation.

- Purpose: Persist and transport dealer contact and identity data used across order processing, quotes, and shipment coordination. This class delivers a lightweight, serializable document model that supports CRUD operations and inter-service communication while keeping domain objects decoupled from storage concerns.

#### ShipmentDetails.java
- Role: MongoDB persistence model for shipment tracking, bridging the domain ShipmentRecord and the database in the Parts Unlimited MRP system.
- Key Functionality: Stores shipment document data (@Document) with MongoDB id and indexed orderId; captures delivery address, contact details, and an ordered event history; provides null-safe construction from a ShipmentRecord and conversion back via toShipmentRecord to keep persistence concerns separate from domain logic.
- Purpose: Enable reliable storage and retrieval of shipment details and lifecycle events for order fulfillment and tracking, supporting MRP workflows like shipment coordination, customer notifications, and audit of delivery milestones in the supply chain.

#### CatalogItem.java
- Role: MongoDB persistence model for catalog items that bridges the Spring Data repository layer and the domain model in the Parts Unlimited MRP system.

- Key Functionality: 
  - Defines stored fields for id, skuNumber, description, price, inventory, and leadTime with Spring Data document/ID/index support.
  - Provides constructors including a cross-layer copy-from constructor to initialize from the domain model.
  - Converts back to the domain model via toCatalogItem, computing an effective lead time (0 when inventory is available, otherwise the stored lead time).
  - Exposes basic accessors (e.g., id getter/setter) for persistence identity.

- Purpose: Persist and translate SKU data used across catalog management, quoting, order processing, and shipment coordination, ensuring accurate availability and lead-time semantics that drive MRP decisions such as pricing, fulfillment timing, and procurement planning.


### Package: `smpl.ordering.repositories.mongodb.test`
#### MongoDealersRepositoryTest.java
- Role: MongoDB-specific test suite for the Dealers repository, validating that the Mongo-backed implementation conforms to the repository contract used across the Parts Unlimited MRP system.

- Key Functionality:
  - Resets the repository factory to the "mongodb" backend before each test to ensure a clean, consistent environment.
  - Leverages a configuration rule to manage test configuration and environment setup.
  - Delegates core dealer operations tests—listing dealers, fetching a dealer, upserting, and removing—to shared superclass logic to ensure behavior parity across persistence implementations.

- Purpose: To guarantee that dealer/supplier management functions behave correctly when persisted in MongoDB, ensuring reliable supplier relationships, order processing, and integration workflows within the MRP platform. This safeguards production planning, procurement, and fulfillment processes by validating the persistence layer’s correctness and consistency.

#### MongoCatalogItemsRepositoryTest.java
- Role: JUnit test harness validating the MongoDB-backed Catalog Items repository within the Parts Unlimited MRP system.

- Key Functionality: 
  - Resets the repository layer to use the MongoDB backend before each test via RepositoryFactory.reset("mongodb").
  - Leverages a JUnit ConfigurationRule to standardize test configuration.
  - Delegates to superclass tests to verify core catalog operations (list, fetch, upsert, remove) against MongoDB.
  - Ensures tests run in a clean, consistent state and inherit common assertions/fixtures.

- Purpose: Ensures the MongoDB implementation of the parts catalog repository behaves correctly for critical MRP workflows (inventory management, order fulfillment, quote generation, and shipment coordination). This safeguards data integrity and consistency across storage backends, reducing regressions and increasing confidence in the system’s persistence layer.

#### MongoShipmentRepositoryTest.java
- Role: MongoDB-specific test harness for the ShipmentRepository, ensuring the Mongo-backed implementation conforms to the shared repository contract used across the Parts Unlimited MRP system.

- Key Functionality: 
  - Resets and initializes the repository layer to a clean MongoDB state before each test via RepositoryFactory.
  - Reuses and executes the generic shipment repository tests (fetch by ID, list, create, update, add event) by delegating to the superclass.
  - Applies configuration through a JUnit rule to run tests under the correct environment settings.

- Purpose: Validates reliability and correctness of shipment persistence and lifecycle operations on the MongoDB backend, underpinning order fulfillment and shipment tracking in the manufacturing workflow. This ensures consistent behavior across storage implementations, reducing defects in shipment coordination and improving trust in inventory and order processing within the MRP application.

#### MongoQuoteRepositoryTest.java
- Role: MongoDB-specific integration test harness for the Quote repository, ensuring the Mongo-backed implementation adheres to the shared repository contract within the Parts Unlimited MRP system.

- Key Functionality:
  - Initializes the persistence layer to the MongoDB backend via RepositoryFactory.reset("mongodb") before tests run.
  - Reuses and executes the base QuoteRepositoryTest suite (get, query by customer name, create, update, remove) in a MongoDB context.
  - Manages test configuration through a JUnit ConfigurationRule to control environment and policy settings.

- Purpose: Validate that quote persistence and retrieval behave consistently when using MongoDB, safeguarding core MRP workflows such as quote generation, pricing, and order preparation. This ensures reliability, data integrity, and consistency across backends for inventory and order management processes.

#### MongoOrderRepositoryTest.java
- Role: MongoDB-specific test harness for the order repository, extending the shared order repository test suite to validate the MongoDB-backed implementation within the Parts Unlimited MRP system.

- Key Functionality:
  - Resets and configures the repository layer to use the MongoDB backend before each test via RepositoryFactory.reset("mongodb").
  - Applies a configurable test rule (ConfigurationRule) to manage environment and policy for tests.
  - Delegates all order-related tests (existence, retrieval, creation, updates, and queries by quote ID, status, and dealer name) to the base OrderRepositoryTest, running them against MongoDB.

- Purpose: Ensure the MongoDB persistence layer for orders behaves consistently with the repository contract, preventing regressions in core order processing operations (creation, updates, and queries) critical to MRP workflows such as order fulfillment, production planning, and shipment coordination. This safeguards backend-agnostic behavior and reliability of the order management subsystem across the manufacturing supply chain processes.

#### IntegrationTests.java
- Role: Marker interface used to categorize and control execution of integration tests within the Parts Unlimited MRP repository.

- Key Functionality: 
  - Provides a tag (via JUnit 4 Categories or similar tooling) to identify tests that exercise end-to-end behavior, including MongoDB persistence, REST API interactions, and Tomcat-based web components.
  - Enables build tools/CI pipelines to include or exclude longer-running, environment-dependent tests separately from unit tests.

- Purpose: 
  - To streamline test orchestration by isolating integration tests that validate real system interactions across inventory, orders, shipments, and external integrations—improving reliability and deployment confidence without impacting fast unit-test feedback loops.


