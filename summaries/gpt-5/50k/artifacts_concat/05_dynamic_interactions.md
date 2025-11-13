# Workflow 1 — Create Quote → Create Order from Quote → Create Shipment (Delivery) from Order

Purpose and triggers
- Purpose: End-to-end user flow to create a new quote, convert it to an order, and initiate shipment.
- Triggers: User actions in the Web Client (WinJS) — Save Quote, Create Order from Quote, Deliver/Create Shipment from Order.

Communication patterns
- Web Client → OrderService: synchronous REST/JSON
  - POST /quotes
  - POST /orders?fromQuote={quoteId}
  - POST /shipments (with ShipmentRecord)
- OrderService → MongoDB: synchronous DB writes via MongoOperationsWithRetry (retry-once on transient failures)
- Cross-cutting: SimpleCORSFilter for browser calls; AppInsightsFilter for request/exception telemetry

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Web Client (WinJS/Data)
  participant OS as OrderService (REST)
  participant QC as QuoteController
  participant OC as OrderController
  participant SC as ShipmentController
  participant RF as RepositoryFactory
  participant QR as QuoteRepository
  participant OR as OrderRepository
  participant SR as ShipmentRepository
  participant MR as MongoOperationsWithRetry
  participant DB as MongoDB

  U->>W: Enter Quote details and click Save
  W->>OS: POST /quotes (Quote JSON)
  activate OS
  OS->>RF: resolve repositories (storage=mongodb)
  OS->>QC: validate payload
  QC->>QR: createQuote(Quote)
  QR->>MR: insert QuoteDetails
  MR->>DB: insert document
  DB-->>MR: OK
  MR-->>QR: OK
  QR-->>QC: Quote with quoteId
  QC-->>OS: 201 Created (Location /quotes/{quoteId})
  deactivate OS
  OS-->>W: Quote(created)

  U->>W: From Quote, choose "Create Order"
  W->>OS: POST /orders?fromQuote={quoteId}
  activate OS
  OS->>OC: createOrder(fromQuoteId)
  OC->>OR: createOrder(fromQuoteId)
  OR->>QR: getQuote(quoteId) (validate exists)
  QR->>MR: findOne quoteId
  MR->>DB: find quote
  DB-->>MR: Quote found
  MR-->>QR: Quote
  OR->>MR: insert OrderDetails (orderId="order-{quoteId}", status=Created)
  MR->>DB: insert document
  DB-->>MR: OK
  MR-->>OR: OK
  OR-->>OC: Order
  OC-->>OS: 201 Created (Location /orders/{orderId})
  deactivate OS
  OS-->>W: Order(created)

  U->>W: From Order, choose "Deliver" (create shipment)
  W->>OS: POST /shipments (ShipmentRecord with orderId)
  activate OS
  OS->>SC: createShipment(record)
  SC->>SR: createShipment(record)
  SR->>OR: hasOrder(orderId) (validate)
  OR->>MR: findOne orderId
  MR->>DB: find order
  DB-->>MR: Order found
  MR-->>OR: OK
  SR->>MR: insert ShipmentDetails
  MR->>DB: insert document
  DB-->>MR: OK
  MR-->>SR: OK
  SR-->>SC: ShipmentRecord
  SC-->>OS: 201 Created (Location /shipments/{orderId})
  deactivate OS
  OS-->>W: Shipment(created)

  alt Duplicate order for quote
    OR-->>OC: ConflictingRequestException
    OC-->>OS: 409 Conflict
    OS-->>W: 409 Conflict
  end

  alt Transient Mongo failure on write
    MR->>DB: insert (timeout)
    DB-->>MR: SocketTimeoutException
    MR->>DB: retry insert
    DB-->>MR: OK
    MR-->>(caller): success
  end
```


# Workflow 2 — Update Order Status and Manage Order Events

Purpose and triggers
- Purpose: Record order lifecycle changes and audit history (events).
- Triggers: User modifies status and/or events in the Orders page; events may be edited in a popup and persisted on Save.

Communication patterns
- Web Client → OrderService: synchronous REST/JSON
  - PUT /orders/{orderId}/status with OrderUpdateInfo{status, eventInfo}
  - POST /orders/{orderId}/events with OrderEventInfo (optional, if Data layer posts discrete events)
- OrderService → MongoDB: synchronous writes via MongoOperationsWithRetry

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Web Client (Orders + Events popup)
  participant OS as OrderService (REST)
  participant OC as OrderController
  participant OR as OrderRepository
  participant MR as MongoOperationsWithRetry
  participant DB as MongoDB

  U->>W: Edit status and/or add events
  U->>W: Click Save on Orders page
  opt Status change present (single-call status+event atomic update)
    W->>OS: PUT /orders/{id}/status (OrderUpdateInfo)
    activate OS
    OS->>OC: updateStatus(id, info)
    OC->>OR: updateOrder(id, OrderUpdateInfo)
    OR->>MR: findOne orderId
    MR->>DB: find order
    DB-->>MR: Order
    OR->>OR: append event, update status
    OR->>MR: save OrderDetails
    MR->>DB: replace document
    DB-->>MR: OK
    MR-->>OR: OK
    OR-->>OC: Updated Order
    OC-->>OS: 200 OK
    deactivate OS
    OS-->>W: Updated Order
  end

  opt Additional unsynced events posted individually
    W->>OS: POST /orders/{id}/events (OrderEventInfo)
    activate OS
    OS->>OC: addEvent(id, info)
    OC->>OR: updateOrder(id, OrderUpdateInfo{event only})
    OR->>MR: save with appended event
    MR->>DB: replace document
    DB-->>MR: OK
    MR-->>OR: OK
    OR-->>OC: Updated
    OC-->>OS: 201 Created
    deactivate OS
    OS-->>W: Event created
  end

  alt Order not found
    OR-->>OC: NotFound
    OC-->>OS: 404 Not Found
    OS-->>W: 404 Not Found
  end

  alt Transient DB failure
    MR->>DB: replace (timeout)
    DB-->>MR: SocketTimeout
    MR->>DB: retry replace
    DB-->>MR: OK
  end
```


# Workflow 3 — IntegrationService Order Intake (Event-driven: Website → Azure Queue → MRP)

Purpose and triggers
- Purpose: Automatically ingest website orders into MRP as Quotes + Orders + Shipments.
- Triggers: Website enqueues OrderMessage into Azure Storage Queue “orders”; scheduled task runs every 30 seconds.

Communication patterns
- Website → Azure Storage Queue “orders”: asynchronous message enqueue (JSON)
- IntegrationService → Azure Queue: pull with visibility timeout; delete after success; malformed messages deleted immediately
- IntegrationService → OrderService: synchronous REST/JSON
  - POST /quotes
  - POST /orders?fromQuote={quoteId}
  - POST /shipments
- OrderService → MongoDB: synchronous DB writes via MongoOperationsWithRetry

```mermaid
sequenceDiagram
  autonumber
  participant WS as External Website
  participant QO as Azure Queue "orders"
  participant IS as IntegrationService (CreateOrderProcessTask)
  participant MRP as MrpConnectService (REST client)
  participant OS as OrderService
  participant QR as QuoteController
  participant ORC as OrderController
  participant SH as ShipmentController
  participant MR as MongoOperationsWithRetry
  participant DB as MongoDB

  WS-->>QO: Enqueue OrderMessage (JSON)
  Note over IS: @Scheduled fixedDelay=30s
  IS->>QO: getQueueMessage(visibility=300s)
  alt Malformed JSON
    QO-->>IS: Message bytes
    IS->>IS: Deserialize fails
    IS->>QO: deleteQueueMessage (poison protection)
    IS->>IS: Log error and continue
  else Valid OrderMessage
    IS->>MRP: createQuote(OrderMessage) → POST /quotes
    MRP->>OS: POST /quotes
    OS->>QR: createQuote → DB write via MR
    QR->>MR: insert
    MR->>DB: insert
    DB-->>MR: OK
    OS-->>MRP: 201 Quote(quoteId)

    IS->>MRP: createOrder(quoteId) → POST /orders?fromQuote
    MRP->>OS: POST /orders?fromQuote
    OS->>ORC: createOrder → DB write via MR
    ORC->>MR: insert
    MR->>DB: insert
    DB-->>MR: OK
    OS-->>MRP: 201 Order(orderId)

    IS->>MRP: createShipment(OrderMessage, orderId) → POST /shipments
    MRP->>OS: POST /shipments
    OS->>SH: createShipment → DB write via MR
    SH->>MR: insert
    MR->>DB: insert
    DB-->>MR: OK
    OS-->>MRP: 201 Shipment

    IS->>QO: deleteQueueMessage
  end

  alt HTTP error (e.g., 5xx) during any step
    IS->>IS: Log error, do not delete message
    Note over QO,IS: Message becomes visible after visibility timeout and will be retried
  end

  alt Duplicate order (409 Conflict)
    OS-->>MRP: 409 Conflict
    IS->>IS: Log warning and delete (idempotency policy)
    IS->>QO: deleteQueueMessage
  end
```


# Workflow 4 — IntegrationService Product Inventory Updates (MRP → Azure Queue “product”)

Purpose and triggers
- Purpose: Publish current catalog inventory/lead time snapshots for the website.
- Triggers: Scheduled task every 30 seconds.

Communication patterns
- IntegrationService → OrderService: synchronous REST/JSON
  - GET /catalog
- IntegrationService → Azure Queue “product”: asynchronous message enqueue with ProductMessage JSON
- Website → Azure Queue “product”: pull/consume (out of scope of codebase, shown for completeness)

```mermaid
sequenceDiagram
  autonumber
  participant IS as IntegrationService (UpdateProductProcessTask)
  participant MRP as MrpConnectService
  participant OS as OrderService
  participant CC as CatalogController
  participant QP as Azure Queue "product"
  participant WS as External Website

  Note over IS: @Scheduled fixedDelay=30s
  IS->>MRP: getCatalogItems() → GET /catalog
  MRP->>OS: GET /catalog
  OS->>CC: listCatalogItems
  CC-->>OS: 200 [CatalogItem]
  OS-->>MRP: [CatalogItem]
  IS->>IS: Map to ProductMessage [ProductItem]
  alt Non-empty catalog
    IS->>QP: addQueueMessage(ProductMessage)
    QP-->>IS: 201 Enqueued
  else Empty catalog
    IS->>IS: Skip enqueue
  end

  opt Website consumption (outside MRP)
    WS->>QP: getQueueMessage
    QP-->>WS: ProductMessage
  end
```


# Workflow 5 — Deliveries Page Composition (Read model aggregation and enrichment)

Purpose and triggers
- Purpose: Display deliveries with related order and quote context; optionally create a delivery from an order.
- Triggers: User navigates to Deliveries; Web Client loads list and enriches items for UI rendering.

Communication patterns
- Web Client → OrderService: synchronous REST/JSON
  - GET /shipments/deliveries (aggregated Delivery DTOs)
  - Optionally, per-item GET /orders/{orderId} and GET /quotes/{quoteId} for extra fields (client-side enrichment)
- OrderService → MongoDB: synchronous reads via MongoOperationsWithRetry across Shipment, Order, Quote repositories (server-side aggregation)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Web Client (Deliveries page)
  participant OS as OrderService
  participant SH as ShipmentController
  participant MR as MongoOperationsWithRetry
  participant DB as MongoDB

  U->>W: Navigate to Deliveries
  W->>OS: GET /shipments/deliveries
  activate OS
  OS->>SH: getDeliveries()
  SH->>MR: query Shipments with status=DeliveryConfirmed
  MR->>DB: find shipments
  DB-->>MR: [ShipmentDetails]
  SH->>MR: fetch related Orders by orderId(s)
  MR->>DB: find orders
  DB-->>MR: [OrderDetails]
  SH->>MR: fetch related Quotes by quoteId(s)
  MR->>DB: find quotes
  DB-->>MR: [QuoteDetails]
  SH-->>OS: 200 [Delivery{Shipment+Order+Quote}]
  deactivate OS
  OS-->>W: [Delivery]

  opt Client-side enrichment for UI template
    loop For each delivery item
      W->>OS: GET /orders/{orderId}
      OS-->>W: Order
      W->>OS: GET /quotes/{quoteId}
      OS-->>W: Quote
      W->>W: Bind __order and __quote into item
    end
  end

  opt Create delivery from selected order (when navigated with {order})
    W->>OS: POST /shipments (ShipmentRecord from order)
    OS-->>W: 201 Shipment
  end
```


# Workflow 6 — Dealers List Retrieval (Known APM Hotspot)

Purpose and triggers
- Purpose: Load list of dealers for maintenance; demonstrate performance anti-pattern (intentional lab).
- Triggers: User navigates to Dealers; Web Client loads dealer list.

Communication patterns
- Web Client → OrderService: synchronous REST/JSON
  - GET /dealers
- OrderService → MongoDB: synchronous reads; controller loops 100000x internally (intentional), causing heavy load
- Observability: AppInsightsFilter tracks long request durations/exceptions

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Web Client (Dealers page)
  participant OS as OrderService
  participant DC as DealerController
  participant DR as DealersRepository
  participant MR as MongoOperationsWithRetry
  participant DB as MongoDB

  U->>W: Navigate to Dealers
  W->>OS: GET /dealers
  activate OS
  OS->>DC: listDealers()
  loop 100000 times (APM lab hotspot)
    DC->>DR: getDealers()
    DR->>MR: findAll
    MR->>DB: find dealers
    DB-->>MR: [Dealers]
    MR-->>DR: [Dealers]
  end
  DC-->>OS: 200 [DealerInfo]
  deactivate OS
  OS-->>W: [DealerInfo]

  Note over OS: AppInsightsFilter records long request duration for diagnostics
```