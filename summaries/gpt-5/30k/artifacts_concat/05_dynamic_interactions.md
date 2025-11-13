# Workflow 1: Catalog browsing and upsert (UI-driven)

- Purpose and trigger:
  - A user views the product catalog and adds or updates a CatalogItem.
  - Triggered by navigating to the Catalog page and submitting a new/edited item.
- Communication patterns:
  - WebUI -> OrderService: REST (JSON) over HTTP (GET /catalog, POST /catalog, PUT /catalog/{sku})
  - OrderService -> MongoDB: synchronous DB calls (find/insert/save) with one-time retry on SocketTimeout via MongoOperationsWithRetry
  - Telemetry: Application Insights for HTTP requests and DB dependencies
- Data flow:
  - CatalogItem { skuNumber, description, price, inventory, leadTime }
  - On GET: list of catalog items from MongoDB
  - On POST/PUT: CatalogItem JSON payload validated and persisted

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant WebUI as WebUI (Tomcat/WinJS)
  participant OrderSvc as OrderService
  participant CatalogCtrl as CatalogController
  participant CatalogRepo
  participant Mongo as MongoDB
  participant AI as AppInsights

  User->>WebUI: Navigate to Catalog page
  WebUI->>OrderSvc: GET /catalog
  OrderSvc->>AI: Track request start
  OrderSvc->>CatalogCtrl: handle GET
  CatalogCtrl->>CatalogRepo: getCatalogItems()
  CatalogRepo->>Mongo: findAll("catalog")
  Mongo-->>CatalogRepo: [CatalogItem...]
  CatalogRepo-->>CatalogCtrl: items
  CatalogCtrl-->>OrderSvc: 200 OK [items]
  OrderSvc->>AI: Track request success
  OrderSvc-->>WebUI: 200 OK [CatalogItem[]]
  WebUI-->>User: Render catalog list

  alt User adds new item
    User->>WebUI: Submit new CatalogItem
    WebUI->>OrderSvc: POST /catalog (CatalogItem JSON)
    OrderSvc->>AI: Track request start
    OrderSvc->>CatalogCtrl: validate + create
    CatalogCtrl->>CatalogRepo: upsertCatalogItem(sku, item)
    CatalogRepo->>Mongo: findOne({skuNumber})
    alt SKU not found
      CatalogRepo->>Mongo: insert(item)
      Mongo-->>CatalogRepo: ack
      CatalogRepo-->>CatalogCtrl: created=false
      CatalogCtrl-->>OrderSvc: 201 Created (Location: /catalog/{sku})
      OrderSvc->>AI: Track success
      OrderSvc-->>WebUI: 201 Created
    else SKU exists (conflict)
      CatalogRepo-->>CatalogCtrl: exists
      CatalogCtrl-->>OrderSvc: 409 Conflict
      OrderSvc->>AI: Track failure
      OrderSvc-->>WebUI: 409 Conflict
      WebUI->>OrderSvc: PUT /catalog/{sku} (update)
      OrderSvc->>AI: Track request start
      OrderSvc->>CatalogCtrl: update
      CatalogCtrl->>CatalogRepo: upsertCatalogItem(sku, item)
      CatalogRepo->>Mongo: save(item)
      Mongo-->>CatalogRepo: ack
      CatalogRepo-->>CatalogCtrl: updated=true
      CatalogCtrl-->>OrderSvc: 200 OK
      OrderSvc->>AI: Track success
      OrderSvc-->>WebUI: 200 OK
    end
  end

  opt Transient DB error
    CatalogRepo->>Mongo: find/insert/save
    Mongo-->>CatalogRepo: SocketTimeoutException
    CatalogRepo->>Mongo: retry once (MongoOperationsWithRetry)
    Mongo-->>CatalogRepo: success or failure
    Note right of CatalogRepo: On second failure, controller returns 500.
  end
```


# Workflow 2: Quote to Order (UI-driven), with dealer upsert and order event/status updates

- Purpose and trigger:
  - A sales user creates a Quote, then converts it to an Order; later adds events and updates status.
  - Triggered by user actions on Quotes and Orders pages.
- Communication patterns:
  - WebUI -> OrderService: REST (JSON) over HTTP (POST /quotes, POST /orders?fromQuote, POST /orders/{id}/events, PUT /orders/{id}/status)
  - Internal repo chaining: QuoteRepo ensures Dealer existence (DealersRepo upsert); OrderRepo validates Quote existence and uniqueness
  - OrderService -> MongoDB: synchronous with retry wrapper
  - Telemetry: Application Insights
- Data flow:
  - Quote { quoteId?, dealerName, customerName, quoteItems[], totalCost, discount, city, postalCode, state }
  - DealerInfo { name, contact, address, email, phone } (auto-upserted if missing)
  - Order { orderId = "order-{quoteId}", quoteId, orderDate, status, events[] }
  - OrderEventInfo { date (server-stamped), comments }

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant WebUI as WebUI (WinJS)
  participant OrderSvc as OrderService
  participant QuoteCtrl as QuoteController
  participant OrderCtrl as OrderController
  participant QuoteRepo
  participant DealersRepo
  participant OrderRepo
  participant Mongo as MongoDB
  participant AI as AppInsights

  User->>WebUI: Create Quote (UI form)
  WebUI->>OrderSvc: POST /quotes (Quote JSON)
  OrderSvc->>AI: Track request start
  OrderSvc->>QuoteCtrl: createQuote()
  QuoteCtrl->>QuoteRepo: createQuote(quote)
  QuoteRepo->>DealersRepo: upsertDealer(quote.dealerName)
  DealersRepo->>Mongo: findOne/save("dealers")
  Mongo-->>DealersRepo: ack
  DealersRepo-->>QuoteRepo: ok
  QuoteRepo->>Mongo: insert("quotes")
  Mongo-->>QuoteRepo: ack (assigned quoteId if missing)
  QuoteRepo-->>QuoteCtrl: Quote (quoteId)
  QuoteCtrl-->>OrderSvc: 201 Created (Location /quotes/{id})
  OrderSvc->>AI: Track success
  OrderSvc-->>WebUI: 201 Created [Quote]

  User->>WebUI: Convert to Order
  WebUI->>OrderSvc: POST /orders?fromQuote={quoteId}
  OrderSvc->>AI: Track request start
  OrderSvc->>OrderCtrl: createOrder()
  OrderCtrl->>OrderRepo: createOrder(quoteId)
  OrderRepo->>QuoteRepo: getQuote(quoteId)
  QuoteRepo->>Mongo: findOne("quotes", quoteId)
  Mongo-->>QuoteRepo: Quote|null
  alt Quote exists and no order yet
    OrderRepo->>Mongo: insert("orders", {orderId:"order-{quoteId}", status:Created, orderDate:now})
    Mongo-->>OrderRepo: ack
    OrderRepo-->>OrderCtrl: Order
    OrderCtrl-->>OrderSvc: 201 Created (Location /orders/{orderId})
    OrderSvc->>AI: Track success
    OrderSvc-->>WebUI: 201 Created [Order]
  else Duplicate order for quote
    OrderRepo-->>OrderCtrl: ConflictingRequest
    OrderCtrl-->>OrderSvc: 409 Conflict
    OrderSvc->>AI: Track failure
    OrderSvc-->>WebUI: 409 Conflict
    WebUI->>OrderSvc: GET /orders/{orderId} (fallback)
    OrderSvc-->>WebUI: 200 OK [Order]
  end

  User->>WebUI: Add order event (comments)
  WebUI->>OrderSvc: POST /orders/{orderId}/events ({comments})
  OrderSvc->>AI: Track request start
  OrderSvc->>OrderCtrl: addEvent()
  OrderCtrl->>OrderRepo: updateOrder(orderId, OrderEventInfo{date=now, comments})
  OrderRepo->>Mongo: save("orders", append events)
  Mongo-->>OrderRepo: ack
  OrderRepo-->>OrderCtrl: ok
  OrderCtrl-->>OrderSvc: 201 Created
  OrderSvc->>AI: Track success
  OrderSvc-->>WebUI: 201 Created

  User->>WebUI: Update order status
  WebUI->>OrderSvc: PUT /orders/{orderId}/status ({status, eventInfo.comments})
  OrderSvc->>OrderCtrl: updateStatus()
  OrderCtrl->>OrderRepo: updateOrder(orderId, OrderUpdateInfo{status, eventInfo{date=now,comments}})
  OrderRepo->>Mongo: save("orders", set status + append event)
  Mongo-->>OrderRepo: ack
  OrderRepo-->>OrderCtrl: ok
  OrderCtrl-->>OrderSvc: 200 OK
  OrderSvc-->>WebUI: 200 OK

  opt Telemetry and DB retry (any DB op)
    OrderSvc->>AI: Dependency: MongoDB.Operation (success/failure, duration)
    Note right of OrderSvc: MongoOperationsWithRetry retries once on SocketTimeoutException.
  end
```


# Workflow 3: Shipment (Delivery) creation from an Order and event management (UI-driven)

- Purpose and trigger:
  - A logistics user creates a ShipmentRecord for an existing Order and manages delivery events.
  - Triggered by navigating to Deliveries and selecting an Order to create/update a shipment.
- Communication patterns:
  - WebUI -> OrderService: REST (JSON) over HTTP (GET/POST/PUT /shipments, POST /shipments/{orderId}/events, GET /shipments?status=)
  - Internal validation: ShipmentRepo validates existence of Order via OrderRepo
  - OrderService -> MongoDB: synchronous with retry wrapper
- Data flow:
  - ShipmentRecord { orderId, deliveryDate, deliveryAddress, contactName, primaryContactPhone, alternateContactPhone, events[] }
  - ShipmentEventInfo { date (server-stamped), comments }
  - Listing by status uses Order.status to filter shipments

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant WebUI as WebUI (WinJS)
  participant OrderSvc as OrderService
  participant ShipCtrl as ShipmentController
  participant ShipRepo as ShipmentRepo
  participant OrderRepo
  participant Mongo as MongoDB

  User->>WebUI: Open Deliveries for Order X
  WebUI->>OrderSvc: GET /shipments/{orderId}
  OrderSvc->>ShipCtrl: getShipmentById(orderId)
  ShipCtrl->>ShipRepo: getShipmentById(orderId)
  ShipRepo->>Mongo: findOne("shipments", orderId)
  Mongo-->>ShipRepo: 404 or Shipment
  alt Shipment not found (first time)
    ShipRepo-->>ShipCtrl: 404
    ShipCtrl-->>OrderSvc: 404 Not Found
    OrderSvc-->>WebUI: 404 Not Found
    WebUI->>OrderSvc: POST /shipments (ShipmentRecord JSON)
    OrderSvc->>ShipCtrl: createShipment()
    ShipCtrl->>ShipRepo: createShipment(record)
    ShipRepo->>OrderRepo: hasOrder(orderId)?
    OrderRepo->>Mongo: exists("orders", orderId)
    Mongo-->>OrderRepo: true/false
    alt Order exists and no shipment yet
      ShipRepo->>Mongo: insert("shipments", record)
      Mongo-->>ShipRepo: ack
      ShipRepo-->>ShipCtrl: created
      ShipCtrl-->>OrderSvc: 201 Created (Location /shipments/{orderId})
      OrderSvc-->>WebUI: 201 Created
    else Order missing or duplicate shipment
      ShipRepo-->>ShipCtrl: BadRequest
      ShipCtrl-->>OrderSvc: 400 Bad Request
      OrderSvc-->>WebUI: 400 Bad Request
    end
  else Shipment exists
    ShipRepo-->>ShipCtrl: Shipment
    ShipCtrl-->>OrderSvc: 200 OK [Shipment]
    OrderSvc-->>WebUI: 200 OK [Shipment]
  end

  User->>WebUI: Add delivery event (comments)
  WebUI->>OrderSvc: POST /shipments/{orderId}/events ({comments})
  OrderSvc->>ShipCtrl: addEvent()
  ShipCtrl->>ShipRepo: addEvent(orderId, ShipmentEventInfo{date=now, comments})
  ShipRepo->>Mongo: save("shipments", append events)
  Mongo-->>ShipRepo: ack
  ShipRepo-->>ShipCtrl: ok
  ShipCtrl-->>OrderSvc: 200 OK
  OrderSvc-->>WebUI: 200 OK

  User->>WebUI: List shipments by status (e.g., Shipped)
  WebUI->>OrderSvc: GET /shipments?status=Shipped
  OrderSvc->>ShipCtrl: getShipments(status=Shipped)
  ShipCtrl->>OrderRepo: getOrdersByStatus(Shipped)
  OrderRepo->>Mongo: find("orders", status=Shipped)
  Mongo-->>OrderRepo: [orderIds]
  ShipCtrl->>ShipRepo: getShipmentsByOrderIds([orderIds])
  ShipRepo->>Mongo: find("shipments", orderId in list)
  Mongo-->>ShipRepo: [ShipmentRecord...]
  ShipRepo-->>ShipCtrl: shipments
  ShipCtrl-->>OrderSvc: 200 OK [shipments]
  OrderSvc-->>WebUI: 200 OK [ShipmentRecord[]]
```


# Workflow 4: External order ingestion via Azure Queue (event-driven, IntegrationService)

- Purpose and trigger:
  - Integrate website orders asynchronously into MRP: consume OrderMessage from Azure Storage Queue, then create Quote, Order, and Shipment in MRP.
  - Triggered by IntegrationService scheduled task every 30s (fixedDelay).
- Communication patterns:
  - IntegrationService <-> Azure Storage Queue: asynchronous dequeue/delete (orders queue)
  - IntegrationService -> OrderService: synchronous REST (POST /quotes, POST /orders?fromQuote, POST /shipments)
  - OrderService -> MongoDB: synchronous persistence with retry
  - Error handling: at-least-once queue semantics; on deserialization error, message deleted; on REST failure, message not deleted (visible after timeout), causing retries and potential duplicates; no idempotency key
- Data flow:
  - OrderMessage { customerName, dealerName, address, phone, city, state, postalCode, totalCost, discount, items[{skuNumber, price}] }
  - Quote(OrderMessage), Order(from Quote), ShipmentRecord(OrderMessage + orderId)

```mermaid
sequenceDiagram
  autonumber
  participant Scheduler as Scheduler (30s)
  participant IntSvc as IntegrationService
  participant OrdersQ as Azure Queue "orders"
  participant OrderSvc as OrderService
  participant QuoteCtrl as QuoteController
  participant OrderCtrl as OrderController
  participant ShipCtrl as ShipmentController

  Scheduler-->>IntSvc: Trigger CreateOrderProcessTask
  loop until queue empty
    IntSvc->>OrdersQ: Dequeue message (visibilityTimeout=300s)
    OrdersQ-->>IntSvc: OrderMessage | null
    alt Message retrieved
      IntSvc->>IntSvc: Deserialize OrderMessage -> Quote + ShipmentRecord templates
      alt Deserialization error
        IntSvc->>OrdersQ: Delete message (poison)
        OrdersQ-->>IntSvc: ack
      else Valid message
        IntSvc->>OrderSvc: POST /quotes (from OrderMessage)
        alt 201 Created
          IntSvc->>OrderSvc: POST /orders?fromQuote={quoteId}
          alt 201 Created
            IntSvc->>OrderSvc: POST /shipments (ShipmentRecord with orderId)
            alt 201 Created
              IntSvc->>OrdersQ: Delete message
              OrdersQ-->>IntSvc: ack
            else 4xx/5xx on /shipments
              IntSvc->>IntSvc: Log error; do NOT delete message
              Note right of IntSvc: Message reappears after visibility timeout (at-least-once)
            end
          else 409 Conflict (order exists) or 4xx/5xx
            IntSvc->>IntSvc: Log error; do NOT delete message
          end
        else 4xx/5xx on /quotes (e.g., duplicate quote ID)
          IntSvc->>IntSvc: Log error; do NOT delete message
        end
      end
    else No message
      IntSvc-->>Scheduler: Sleep until next tick
    end
  end

  Note over IntSvc,OrderSvc: No idempotency keys; duplicate processing can occur on retries.<br/>Order creation conflicts (409) surface as errors here.
```


# Workflow 5: Inventory publish to website via product queue (event-driven, IntegrationService)

- Purpose and trigger:
  - Periodically export current catalog inventory/lead times to the website via Azure Storage Queue “product”.
  - Triggered by IntegrationService scheduled task every 30s (fixedDelay).
- Communication patterns:
  - IntegrationService -> OrderService: synchronous REST (GET /catalog)
  - IntegrationService -> Azure Queue “product”: asynchronous enqueue of ProductMessage
- Data flow:
  - ProductMessage { productList: [{ skuNumber, inventory, leadTime }] } built from OrderService CatalogItem list

```mermaid
sequenceDiagram
  autonumber
  participant Scheduler as Scheduler (30s)
  participant IntSvc as IntegrationService
  participant OrderSvc as OrderService
  participant CatalogCtrl as CatalogController
  participant ProductQ as Azure Queue "product"

  Scheduler-->>IntSvc: Trigger UpdateProductProcessTask
  IntSvc->>OrderSvc: GET /catalog
  OrderSvc->>CatalogCtrl: getCatalogItems()
  CatalogCtrl-->>OrderSvc: 200 OK [CatalogItem[]] or 404
  alt Non-empty catalog
    IntSvc->>IntSvc: Transform to ProductMessage (skuNumber, inventory, leadTime)
    IntSvc->>ProductQ: Enqueue ProductMessage
    ProductQ-->>IntSvc: ack
  else Empty or 404
    IntSvc->>IntSvc: Skip enqueue
  end
```


# Workflow 6: Dealer listing heavy-load path with telemetry and DB retry

- Purpose and trigger:
  - Demonstrate APM/telemetry under load; DealerController intentionally executes 100000 repository calls on GET /dealers.
  - Triggered by a user viewing Dealers.
- Communication patterns:
  - WebUI -> OrderService: REST (GET /dealers)
  - OrderService -> MongoDB: repeated synchronous reads with one-time retry on SocketTimeout per call
  - Telemetry: Application Insights per HTTP request and per DB dependency
- Data flow:
  - DealerInfo[] from MongoDB

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant WebUI as WebUI (WinJS)
  participant OrderSvc as OrderService
  participant DealerCtrl as DealerController
  participant DealersRepo
  participant Mongo as MongoDB
  participant AI as AppInsights

  User->>WebUI: Navigate to Dealers
  WebUI->>OrderSvc: GET /dealers
  OrderSvc->>AI: Track request start
  OrderSvc->>DealerCtrl: getDealers()
  loop 100000 times (intentional)
    DealerCtrl->>DealersRepo: getDealers()
    DealersRepo->>Mongo: findAll("dealers")
    alt SocketTimeoutException
      DealersRepo->>Mongo: retry once
    end
    Mongo-->>DealersRepo: [DealerInfo...]
    DealersRepo-->>DealerCtrl: list
    OrderSvc->>AI: Dependency telemetry (MongoDB.Operation)
  end
  DealerCtrl-->>OrderSvc: 200 OK [DealerInfo[]] or 404 if empty
  OrderSvc->>AI: Track request success/failure
  OrderSvc-->>WebUI: Response
```


# Workflow 7: Health check (operational)

- Purpose and trigger:
  - External monitors check service liveness and build/config info.
  - Triggered by monitoring system or during deployments.
- Communication patterns:
  - Client/Monitor -> OrderService: REST (HEAD/GET /ping)
- Data flow:
  - GET /ping returns pingMessage, validationMessage, build.number, build.timestamp

```mermaid
sequenceDiagram
  autonumber
  participant Monitor
  participant OrderSvc as OrderService
  participant PingCtrl as PingController

  Monitor->>OrderSvc: HEAD /ping
  OrderSvc->>PingCtrl: liveness
  PingCtrl-->>OrderSvc: 200 OK
  OrderSvc-->>Monitor: 200 OK

  Monitor->>OrderSvc: GET /ping
  OrderSvc->>PingCtrl: get health/build info
  PingCtrl-->>OrderSvc: 200 OK { pingMessage, validationMessage, buildinfo }
  OrderSvc-->>Monitor: 200 OK [JSON]
```


# Error handling and recovery patterns (cross-cutting highlights)

- Synchronous REST error semantics:
  - Validation errors: 400 Bad Request (e.g., invalid Dealer, duplicate Quote or Shipment, etc.)
  - Not found: 404 (single entity missing or empty list)
  - Conflict: 409 (duplicate Catalog POST, Order already exists for quote)
  - Success codes: 200 OK (read/update), 201 Created (create/event), 204 No Content (delete)
- Database resilience:
  - MongoOperationsWithRetry retries once on SocketTimeoutException for common operations; emits dependency telemetry for success/failure and duration
- Telemetry:
  - Application Insights filter around each OrderService HTTP request; dependency telemetry around each MongoDB operation
- CORS:
  - SimpleCORSFilter permits browser calls from WebUI to OrderService across ports
- Queue processing (IntegrationService):
  - At-least-once delivery: messages are deleted only on end-to-end success
  - Deserialization errors: messages deleted to avoid poison loops
  - No explicit idempotency/circuit breaker: partial failures may cause duplicates on retry (e.g., quotes, orders, shipments)