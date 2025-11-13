### Workflow 1: Browse catalog (read-only)
Purpose and trigger:
- Purpose: Display available products (catalog) to a user.
- Trigger: User opens the MRP web UI and navigates to the Catalog page.

Communication patterns:
- REST: GET /catalog
- Database: MongoDB findAll on catalog collection
- Rendering in browser

```mermaid
sequenceDiagram
    autonumber
    participant U as User (Browser)
    participant Web as MRP Web Client (Tomcat@9080)
    participant OS as OrderService (Spring Boot@8080)
    participant DB as MongoDB

    U->>Web: Navigate to /mrp (static site)
    Web->>OS: GET /catalog
    OS->>DB: findAll(Catalog)
    DB-->>OS: CatalogItem[]
    OS-->>Web: 200 OK (JSON)
    Web-->>U: Render catalog list
```

---

### Workflow 2: Create or update a quote from the UI
Purpose and trigger:
- Purpose: Create a new quote or update an existing one based on customer and sizing details.
- Trigger: User edits fields on Quotes page and clicks Save.

Communication patterns:
- REST: POST /quotes (create), PUT /quotes/{id} (update)
- Database: MongoDB insert/save in quotes collection

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant Web as Web Client
    participant OS as OrderService
    participant DB as MongoDB

    U->>Web: Enter quote details, click Save
    alt New quote
        Web->>OS: POST /quotes (JSON Quote)
        OS->>DB: insert(Quote)
        DB-->>OS: ack (id)
        OS-->>Web: 201 Created + Location:/quotes/{id} + body
    else Update existing quote
        Web->>OS: PUT /quotes/{id} (JSON Quote)
        OS->>DB: save(QuoteDetails by id)
        DB-->>OS: ack
        OS-->>Web: 200 OK
    end
    Web-->>U: Show saved quote
```

---

### Workflow 3: Create order from a quote (with conflict handling)
Purpose and trigger:
- Purpose: Convert an approved quote into an order; gracefully handle duplicate creations.
- Trigger: User clicks “Deliver”/“Create order” from the Quote detail or Orders page.

Communication patterns:
- REST: POST /orders?fromQuote={quoteId}
- Conflict resolution: If 409, GET /orders/order-{quoteId}, then GET /quotes/{quoteId}
- Database: MongoDB insert/select on orders, quotes

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant Web as Web Client
    participant OS as OrderService
    participant DB as MongoDB

    U->>Web: Create order from quote {quoteId}
    Web->>OS: POST /orders?fromQuote={quoteId}
    alt Order not yet exists for quote
        OS->>DB: find(QuoteDetails by quoteId)
        DB-->>OS: QuoteDetails
        OS->>DB: insert(OrderDetails) with id "order-{quoteId}"
        DB-->>OS: ack
        OS-->>Web: 201 Created (Order)
    else Order already exists
        OS-->>Web: 409 Conflict
        Web->>OS: GET /orders/order-{quoteId}
        OS->>DB: find(OrderDetails by orderId)
        DB-->>OS: OrderDetails
        OS-->>Web: 200 OK (Order)
        Web->>OS: GET /quotes/{quoteId}
        OS->>DB: find(QuoteDetails by quoteId)
        DB-->>OS: QuoteDetails
        OS-->>Web: 200 OK (Quote)
    end
    Web-->>U: Show order detail (with quote)
```

---

### Workflow 4: Create a shipment and update order status from Deliveries UI
Purpose and trigger:
- Purpose: Confirm delivery details and persist shipment; sync order status changes.
- Trigger: User opens Deliveries page, creates/edits a shipment and saves.

Communication patterns:
- REST: POST /shipments (create), PUT /orders/{orderId} (status and event), PUT /shipments/{orderId} (update)
- Database: MongoDB insert/save in shipments and orders collections

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant Web as Web Client (Deliveries)
    participant OS as OrderService
    participant DB as MongoDB

    U->>Web: Enter shipment details, click Save
    alt New shipment
        Web->>OS: POST /shipments (ShipmentRecord JSON)
        OS->>DB: insert(ShipmentDetails)
        DB-->>OS: ack
        OS-->>Web: 201 Created
    else Update existing shipment
        Web->>OS: PUT /shipments/{orderId} (ShipmentRecord JSON)
        OS->>DB: save(ShipmentDetails by orderId)
        DB-->>OS: ack
        OS-->>Web: 200 OK
    end

    opt Order status changed
        Web->>OS: PUT /orders/{orderId}/status (OrderUpdateInfo)
        OS->>DB: save(OrderDetails status + add event)
        DB-->>OS: ack
        OS-->>Web: 200 OK
    end

    Web-->>U: Show updated delivery and order status
```

---

### Workflow 5: Deliveries page initial load (composite read)
Purpose and trigger:
- Purpose: Load shipments and hydrate each list item with linked order and quote for display.
- Trigger: User navigates to Deliveries page.

Communication patterns:
- REST: GET /shipments, then for each item GET /orders/{orderId}, then GET /quotes/{quoteId}
- Database: MongoDB reads in shipments, orders, and quotes collections

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant Web as Web Client (Deliveries)
    participant OS as OrderService
    participant DB as MongoDB

    U->>Web: Open Deliveries page
    Web->>OS: GET /shipments
    OS->>DB: find(Shipments)
    DB-->>OS: ShipmentDetails[]
    OS-->>Web: 200 OK (shipments)

    loop For each shipment item
        Web->>OS: GET /orders/{orderId}
        OS->>DB: find(OrderDetails by orderId)
        DB-->>OS: OrderDetails
        OS-->>Web: 200 OK (order)

        Web->>OS: GET /quotes/{quoteId}
        OS->>DB: find(QuoteDetails by quoteId)
        DB-->>OS: QuoteDetails
        OS-->>Web: 200 OK (quote)
    end

    Web-->>U: Render deliveries list with enriched details
```

---

### Workflow 6: Event-driven order ingestion via IntegrationService (Azure Storage Queue -> MRP)
Purpose and trigger:
- Purpose: Integrate website orders asynchronously into MRP (quote → order → shipment).
- Trigger: Website posts OrderMessage to Azure Storage Queue “orders”; IntegrationService scheduled task runs every 30 seconds.

Communication patterns:
- Asynchronous: Azure Storage Queue (orders)
- REST: POST /quotes, POST /orders?fromQuote=..., POST /shipments
- Queue ops: getQueueMessage, deleteMessage
- Database: MongoDB inserts in quotes, orders, shipments

```mermaid
sequenceDiagram
    autonumber
    participant WS as Website (external)
    participant Q as Azure Queue (orders)
    participant IS as IntegrationService (Scheduled 30s)
    participant MRP as MrpConnectService
    participant OS as OrderService
    participant DB as MongoDB

    WS->>Q: enqueue(OrderMessage JSON)
    IS->>Q: getQueueMessage()
    alt Deserialization OK
        IS->>MRP: createNewOrder(OrderMessage)
        MRP->>OS: POST /quotes (Quote from message)
        OS->>DB: insert(QuoteDetails)
        DB-->>OS: ack
        OS-->>MRP: 201 (Quote with quoteId)

        MRP->>OS: POST /orders?fromQuote={quoteId}
        OS->>DB: insert(OrderDetails order-{quoteId})
        DB-->>OS: ack
        OS-->>MRP: 201 (Order with orderId)

        MRP->>OS: POST /shipments (ShipmentRecord with orderId)
        OS->>DB: insert(ShipmentDetails)
        DB-->>OS: ack
        OS-->>MRP: 201

        IS->>Q: deleteMessage()
    else Deserialization fails
        IS-->>IS: log error
        IS->>Q: deleteMessage()  %% poison-avoid cleanup %%
    end
```

---

### Workflow 7: Event-driven product inventory publishing (IntegrationService -> Website)
Purpose and trigger:
- Purpose: Periodically publish current inventory/lead-time to the website.
- Trigger: Scheduled IntegrationService task (every 30 seconds).

Communication patterns:
- REST: GET /catalog
- Asynchronous: Azure Storage Queue (product)
- Database: MongoDB catalog read

```mermaid
sequenceDiagram
    autonumber
    participant IS as IntegrationService (Scheduled 30s)
    participant MRP as MrpConnectService
    participant OS as OrderService
    participant DB as MongoDB
    participant Q as Azure Queue (product)

    IS->>MRP: getCatalogItems()
    MRP->>OS: GET /catalog
    OS->>DB: findAll(Catalog)
    DB-->>OS: CatalogItem[]
    OS-->>MRP: 200 OK (catalog)
    alt Non-empty list
        IS-->>Q: addQueueMessage(ProductMessage with ProductItem list)
    else Empty list
        IS-->>IS: noop
    end
```

---

### Workflow 8: Error handling and recovery
Purpose and trigger:
- Purpose: Demonstrate resilience patterns for transient MongoDB issues and poisoned queue messages.
- Trigger: Transient DB timeouts; malformed queue messages.

Communication patterns:
- Retry on transient DB error (SocketTimeout) within MongoOperationsWithRetry
- Delete malformed queue messages to avoid poison

```mermaid
sequenceDiagram
    autonumber
    participant Web as Web Client
    participant OS as OrderService
    participant Repo as Repositories (Mongo*)
    participant MOps as MongoOperationsWithRetry
    participant DB as MongoDB
    participant AI as AppInsights

    Web->>OS: REST request (e.g., GET /orders/{id})
    OS->>Repo: getOrder(id)
    Repo->>MOps: findOne(query, OrderDetails)
    alt Success on first attempt
        MOps->>DB: findOne
        DB-->>MOps: doc
        MOps-->>Repo: doc
    else SocketTimeoutException
        MOps-->>OS: catch DataAccessResourceFailureException
        MOps->>DB: findOne (retry)
        DB-->>MOps: doc
        MOps-->>Repo: doc
    end
    Repo-->>OS: Order
    OS-->>Web: 200 OK
    par Telemetry
        OS->>AI: track Request (AppInsightsFilter)
        MOps->>AI: track Dependency (Mongo op + duration + success)
    end
```

- Queue deserialization error handling is shown in Workflow 6 (branch “Deserialization fails”: deleteMessage).

---

### Notes on ports and deployment
- Web client is served by Tomcat (default moved to 9080 in scripts), OrderService runs on 8080.
- Client-side baseAddress points to http://{hostname}:8080 for REST calls.
- Docker example maps:
  - db (MongoDB): 27017
  - order (OrderService): 8080
  - web (Tomcat serving static UI): 80
- Initial data seeding to MongoDB via MongoRecords.js included in deployment scripts.

---

### Summary of communication patterns
- Synchronous:
  - REST (JSON over HTTP): CRUD on /catalog, /dealers, /quotes, /orders, /shipments
  - Database transactions: MongoDB inserts/updates/queries via Spring Data MongoTemplate with retry wrapper
  - Telemetry: AppInsights Request and Dependency telemetry per request/DB op

- Asynchronous:
  - Azure Storage Queues:
    - orders: Website → IntegrationService (OrderMessage); IntegrationService deletes after successful processing or on deserialization error
    - product: IntegrationService → Website (ProductMessage)
  - Scheduled tasks: IntegrationService runs every 30 seconds for both order ingestion and inventory publishing

- Error handling:
  - OrderService controllers return 400/409/404/500 with clear semantics
  - Mongo operations retry once on SocketTimeoutException
  - Malformed queue messages are logged and deleted to prevent poison messages
  - CORS enabled for cross-origin UI calls