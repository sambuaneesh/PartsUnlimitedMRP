
# Key Workflow Sequence Diagrams for Parts Unlimited MRP System

## 1. Quote Creation Workflow

```mermaid
sequenceDiagram
    participant User as Browser User
    participant Frontend as Frontend Web Service (9080)
    participant OrderAPI as Order Service API (8080)
    participant CatalogRepo as Catalog Repository
    participant DealerRepo as Dealer Repository
    participant QuoteRepo as Quote Repository
    participant MongoDB as MongoDB

    User->>Frontend: Navigate to Quotes Page
    Frontend->>Frontend: Load quotes.html
    Frontend->>OrderAPI: GET /dealers
    OrderAPI->>DealerRepo: findAll()
    DealerRepo->>MongoDB: Query dealers collection
    MongoDB-->>DealerRepo: Return dealers list
    DealerRepo-->>OrderAPI: Dealer list
    OrderAPI-->>Frontend: JSON response
    Frontend-->>User: Display dealer selection

    User->>Frontend: Select dealer & enter quote details
    User->>Frontend: Add catalog items to quote
    Frontend->>OrderAPI: GET /catalog
    OrderAPI->>CatalogRepo: findAll()
    CatalogRepo->>MongoDB: Query catalog collection
    MongoDB-->>CatalogRepo: Return catalog items
    CatalogRepo-->>OrderAPI: Catalog list
    OrderAPI-->>Frontend: JSON response
    Frontend-->>User: Display product catalog

    User->>Frontend: Select products & enter quantities
    User->>Frontend: Calculate totals & discount
    User->>Frontend: Click "Create Quote"
    Frontend->>OrderAPI: POST /quotes
    Note over Frontend,OrderAPI: Quote data: customerName, dealerName, items[], totalCost, discount
    OrderAPI->>QuoteRepo: save(quote)
    QuoteRepo->>MongoDB: Insert into quotes collection
    MongoDB-->>QuoteRepo: Confirmation with quoteId
    QuoteRepo-->>OrderAPI: Saved quote
    OrderAPI-->>Frontend: 201 Created with quoteId
    Frontend-->>User: Quote created successfully
```

**Purpose**: Enables dealers to create customer quotes with product selections and pricing calculations
**Trigger**: User navigates to Quotes module and fills quote form
**Communication Pattern**: Synchronous REST API calls, database transactions

---

## 2. Order Creation from Quote Workflow

```mermaid
sequenceDiagram
    participant User as Dealer User
    participant Frontend as Frontend Web Service (9080)
    participant OrderAPI as Order Service API (8080)
    participant QuoteRepo as Quote Repository
    participant OrderRepo as Order Repository
    participant MongoDB as MongoDB
    participant QueueService as Azure Queue Service

    User->>Frontend: Navigate to Orders Page
    Frontend->>OrderAPI: GET /quotes
    OrderAPI->>QuoteRepo: findAll()
    QuoteRepo->>MongoDB: Query quotes collection
    MongoDB-->>QuoteRepo: Quotes list
    QuoteRepo-->>OrderAPI: Quote list
    OrderAPI-->>Frontend: JSON response
    Frontend-->>User: Display available quotes

    User->>Frontend: Select quote and click "Create Order"
    Frontend->>OrderAPI: POST /orders?fromQuote={quoteId}
    OrderAPI->>QuoteRepo: findById(quoteId)
    QuoteRepo->>MongoDB: Find quote document
    MongoDB-->>QuoteRepo: Quote data
    QuoteRepo-->>OrderAPI: Quote details

    OrderAPI->>OrderRepo: createOrderFromQuote(quote)
    Note over OrderAPI,OrderRepo: Create order with status=Created
    OrderRepo->>MongoDB: Insert into orders collection
    MongoDB-->>OrderRepo: Order created with orderId
    OrderRepo-->>OrderAPI: Order confirmation

    OrderAPI->>QueueService: Enqueue order for external MRP
    Note over OrderAPI,QueueService: Async processing by Integration Service
    QueueService-->>OrderAPI: Queued successfully

    OrderAPI-->>Frontend: 201 Created with orderId
    Frontend-->>User: Order created from quote
```

**Purpose**: Converts approved quotes into executable orders
**Trigger**: Dealer selects quote and initiates order creation
**Communication Pattern**: Synchronous API call + asynchronous queue message

---

## 3. Order Fulfillment & Status Update Workflow

```mermaid
sequenceDiagram
    participant Operator as Operations Staff
    participant Frontend as Frontend Web Service (9080)
    participant OrderAPI as Order Service API (8080)
    participant OrderRepo as Order Repository
    participant EventRepo as Events Repository
    participant MongoDB as MongoDB
    participant Integration as Integration Service
    participant MRPExt as External MRP System

    Operator->>Frontend: Navigate to Orders Page
    Frontend->>OrderAPI: GET /orders
    OrderAPI->>OrderRepo: findAll()
    OrderRepo->>MongoDB: Query orders collection
    MongoDB-->>OrderRepo: Orders list with status
    OrderRepo-->>OrderAPI: Order list
    OrderAPI-->>Frontend: JSON response
    Frontend-->>Operator: Display order dashboard

    Operator->>Frontend: Select order & update status
    Operator->>Frontend: Click "Update Status"
    Frontend->>OrderAPI: PUT /orders/{orderId}/status
    Note over Frontend,OrderAPI: New status (e.g., Confirmed, Started, Built)
    
    OrderAPI->>OrderRepo: updateOrderStatus(orderId, status)
    OrderRepo->>MongoDB: Update order document status
    MongoDB-->>OrderRepo: Update confirmed
    OrderRepo-->>OrderAPI: Status updated

    alt If status change requires external sync
        OrderAPI->>Integration: Sync status to external MRP
        Integration->>MRPExt: REST POST /orders/{id}/status
        MRPExt-->>Integration: Confirmation
        Integration-->>OrderAPI: Sync complete
    end

    OrderAPI->>EventRepo: addOrderEvent(orderId, statusChange)
    EventRepo->>MongoDB: Add event to order events array
    MongoDB-->>EventRepo: Event logged
    EventRepo-->>OrderAPI: Event recorded

    OrderAPI-->>Frontend: 200 OK
    Frontend-->>Operator: Status updated successfully
```

**Purpose**: Manages order lifecycle through production stages
**Trigger**: Operations staff updates order status
**Communication Pattern**: Synchronous REST updates, optional external MRP sync

---

## 4. Shipment & Delivery Tracking Workflow

```mermaid
sequenceDiagram
    participant Shipper as Shipping Staff
    participant Frontend as Frontend Web Service (9080)
    participant OrderAPI as Order Service API (8080)
    participant ShipmentRepo as Shipment Repository
    participant OrderRepo as Order Repository
    participant MongoDB as MongoDB

    Shipper->>Frontend: Navigate to Deliveries Page
    Frontend->>OrderAPI: GET /shipments?status=pending
    OrderAPI->>ShipmentRepo: findByStatus("pending")
    ShipmentRepo->>MongoDB: Query shipments collection
    MongoDB-->>ShipmentRepo: Pending shipments
    ShipmentRepo-->>OrderAPI: Shipments list
    OrderAPI-->>Frontend: JSON response
    Frontend-->>Shipper: Display pending deliveries

    Shipper->>Frontend: Create shipment for order
    Shipper->>Frontend: Enter delivery details & contacts
    Frontend->>OrderAPI: POST /shipments
    Note over Frontend,OrderAPI: orderId, deliveryAddress, contacts
    
    OrderAPI->>OrderRepo: findById(orderId)
    OrderRepo->>MongoDB: Find order document
    MongoDB-->>OrderRepo: Order details
    OrderRepo-->>OrderAPI: Order data

    OrderAPI->>ShipmentRepo: createShipment(order, deliveryInfo)
    ShipmentRepo->>MongoDB: Insert into shipments collection
    MongoDB-->>ShipmentRepo: Shipment created with shipmentId
    ShipmentRepo-->>OrderAPI: Shipment confirmation

    OrderAPI-->>Frontend: 201 Created with shipmentId
    Frontend-->>Shipper: Shipment created

    Shipper->>Frontend: Track delivery progress
    Shipper->>Frontend: Add delivery events
    Frontend->>OrderAPI: POST /shipments/{shipmentId}/events
    Note over Frontend,OrderAPI: Event: date, comments
    OrderAPI->>ShipmentRepo: addShipmentEvent(shipmentId, event)
    ShipmentRepo->>MongoDB: Push event to events array
    MongoDB-->>ShipmentRepo: Event added
    ShipmentRepo-->>OrderAPI: Event logged
    OrderAPI-->>Frontend: 200 OK
    Frontend-->>Shipper: Delivery event recorded
```

**Purpose**: Tracks shipments and delivery events for orders
**Trigger**: Shipping staff creates shipments and updates delivery status
**Communication Pattern**: Synchronous REST API calls with event logging

---

## 5. External MRP Integration Workflow

```mermaid
sequenceDiagram
    participant Scheduler as Spring Scheduler
    participant Integration as Integration Service
    participant QueueService as Azure Queue Service
    participant OrderAPI as Order Service API (8080)
    participant OrderRepo as Order Repository
    participant MRPExt as External MRP System
    participant MongoDB as MongoDB

    Note over Scheduler: Every 30 seconds
    Scheduler->>Integration: execute CreateOrderProcessTask()
    Integration->>QueueService: Dequeue from orders queue
    QueueService-->>Integration: Order message (orderId)
    
    Integration->>OrderAPI: GET /orders/{orderId}
    OrderAPI->>OrderRepo: findById(orderId)
    OrderRepo->>MongoDB: Find order document
    MongoDB-->>OrderRepo: Order details
    OrderRepo-->>OrderAPI: Order data
    OrderAPI-->>Integration: Order JSON

    Integration->>MRPExt: POST /external/orders
    Note over Integration,MRPExt: Transform order to external format
    MRPExt-->>Integration: External order created

    alt If external order created successfully
        Integration->>OrderAPI: PUT /orders/{orderId}/status
        Note over Integration,OrderAPI: Update to external system status
        OrderAPI->>OrderRepo: updateOrderStatus(orderId, status)
        OrderRepo->>MongoDB: Update status field
        MongoDB-->>OrderRepo: Confirmed
        OrderRepo-->>OrderAPI: Updated
        OrderAPI-->>Integration: Status synchronized
    else If external system error
        Integration->>QueueService: Re-enqueue order with delay
    end

    Note over Scheduler: Every 30 seconds
    Scheduler->>Integration: execute UpdateProductProcessTask()
    Integration->>MRPExt: GET /external/products/updates
    MRPExt-->>Integration: Product updates

    loop For each product update
        Integration->>OrderAPI: PUT /catalog/{sku}
        Note over Integration,OrderAPI: Update inventory/pricing
        OrderAPI->>MongoDB: Update catalog item
        MongoDB-->>OrderAPI: Updated
        OrderAPI-->>Integration: Confirmation
    end
```

**Purpose**: Synchronizes orders and catalog with external MRP system
**Trigger**: Scheduled tasks every 30 seconds
**Communication Pattern**: Asynchronous queue processing, REST API integration

---

## 6. Catalog Management Workflow

```mermaid
sequenceDiagram
    participant Manager as Inventory Manager
    participant Frontend as Frontend Web Service (9080)
    participant OrderAPI as Order Service API (8080)
    participant CatalogRepo as Catalog Repository
    participant MongoDB as MongoDB

    Manager->>Frontend: Navigate to Catalog Page
    Frontend->>OrderAPI: GET /catalog
    OrderAPI->>CatalogRepo: findAll()
    CatalogRepo->>MongoDB: Query catalog collection
    MongoDB-->>CatalogRepo: All catalog items
    CatalogRepo-->>OrderAPI: Catalog list
    OrderAPI-->>Frontend: JSON response
    Frontend-->>Manager: Display product catalog

    Manager->>Frontend: Add new product or update existing
    Manager->>Frontend: Enter product details
    Note over Manager,Frontend: SKU, description, price, inventory, leadTime
    Manager->>Frontend: Click Save

    alt Adding new product
        Frontend->>OrderAPI: POST /catalog
        OrderAPI->>CatalogRepo: findBySku(sku)
        CatalogRepo->>MongoDB: Check if SKU exists
        MongoDB-->>CatalogRepo: Not found
        CatalogRepo-->>OrderAPI: SKU available
        OrderAPI->>CatalogRepo: save(catalogItem)
        CatalogRepo->>MongoDB: Insert into catalog collection
        MongoDB-->>CatalogRepo: Created with _id
        CatalogRepo-->>OrderAPI: Saved
    else Updating existing product
        Frontend->>OrderAPI: PUT /catalog/{sku}
        OrderAPI->>CatalogRepo: findBySku(sku)
        CatalogRepo->>MongoDB: Find existing item
        MongoDB-->>CatalogRepo: Catalog item
        CatalogRepo-->>OrderAPI: Found
        OrderAPI->>CatalogRepo: update(catalogItem)
        CatalogRepo->>MongoDB: Update document
        MongoDB-->>CatalogRepo: Updated
        CatalogRepo-->>OrderAPI: Saved
    end

    OrderAPI-->>Frontend: 201 Created / 200 OK
    Frontend-->>Manager: Product saved successfully
```

**Purpose**: Manages product catalog, inventory levels, and pricing
**Trigger**: Inventory manager adds or updates products
**Communication Pattern**: Synchronous REST API calls with database validation

---

## 7. Error Handling & Recovery Pattern

```mermaid
sequenceDiagram
    participant Client as Frontend Client
    participant OrderAPI as Order Service API (8080)
    participant Filter as CORS/AppInsights Filter
    participant Controller as REST Controller
    participant Service as Business Service
    participant Repository as Repository Layer
    participant MongoDB as MongoDB

    Client->>OrderAPI: REST Request
    Filter->>Filter: Log request metrics
    Filter->>Controller: Forward request
    
    Controller->>Service: Call business logic
    
    alt Database connection error
        Service->>Repository: data access operation
        Repository->>MongoDB: Connect/Query
        MongoDB-->>Repository: Connection timeout
        Repository-->>Service: RepositoryException
        Service-->>Controller: ServiceException with retry logic
        Controller->>Filter: Error response
        Filter->>Filter: Log error with AppInsights
        Filter-->>Client: 500 Internal Server Error
        
        Note over Service: Implement retry mechanism
        Service->>Repository: Retry operation (3 attempts)
        Repository->>MongoDB: Reconnect attempt
        alt Recovery successful
            MongoDB-->>Repository: Success
            Repository-->>Service: Data returned
            Service-->>Controller: Success response
            Controller-->>Client: 200 OK
        else Retry exhausted
            Repository-->>Service: Failure
            Service-->>Controller: Fallback to mock data
            Controller-->>Client: 200 OK with cached data
        end
    else Business validation error
        Service->>Service: Validate business rules
        Service-->>Controller: BadRequestException
        Controller-->>Filter: 400 Bad Request
        Filter-->>Client: Error details in response
    else Concurrent modification
        Service->>Repository: update operation
        Repository->>MongoDB: Update with version check
        MongoDB-->>Repository: Version conflict
        Repository-->>Service: ConflictingRequestException
        Service-->>Controller: 409 Conflict
        Controller-->>Client: Conflict response with current state
    end
```

**Purpose**: Handles errors gracefully with retry mechanisms and fallback strategies
**Trigger**: Any operation failure (database errors, validation failures, conflicts)
**Communication Pattern**: Error propagation through layers, monitoring integration, retry logic