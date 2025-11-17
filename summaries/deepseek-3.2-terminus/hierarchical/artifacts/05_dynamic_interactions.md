```markdown
# Dynamic Interaction Flows for PartsUnlimitedMRP

## 1. Order Processing Workflow

### Workflow Description
**Purpose**: Complete order lifecycle management from quote creation through final shipment and installation
**Triggers**: Customer places order through website or external system
**Communication Patterns**: REST APIs, Azure Queue messaging, synchronous database transactions, event-driven status updates

```mermaid
sequenceDiagram
    participant Customer
    participant Website as Website Frontend
    participant OrderQueue as Azure Order Queue
    participant CreateOrderTask as CreateOrderProcessTask
    participant OrderController
    participant OrderRepository
    participant QuoteController
    participant ShipmentController
    participant MongoDB

    Note over Customer, MongoDB: Order Creation Phase
    Customer->>Website: Submit Order
    Website->>OrderQueue: Publish OrderMessage
    
    Note over CreateOrderTask, MongoDB: Background Processing
    loop Every 30 seconds
        CreateOrderTask->>OrderQueue: Poll for messages
        OrderQueue->>CreateOrderTask: Return OrderMessage
        CreateOrderTask->>OrderController: POST /orders (create from message)
        OrderController->>OrderRepository: createOrder(order)
        OrderRepository->>MongoDB: Insert OrderDetails
        MongoDB->>OrderRepository: Return saved order
        OrderRepository->>OrderController: Return Order
        OrderController->>CreateOrderTask: 201 Created
        CreateOrderTask->>OrderQueue: Delete processed message
    end

    Note over Customer, MongoDB: Order Fulfillment Phase
    OrderController->>OrderRepository: updateOrderStatus(orderId, CONFIRMED)
    OrderRepository->>MongoDB: Update OrderDetails status
    MongoDB->>OrderRepository: Return updated order
    
    OrderController->>OrderRepository: addOrderEvent(orderId, event)
    OrderRepository->>MongoDB: Push event to events array
    MongoDB->>OrderRepository: Return success

    OrderController->>ShipmentController: POST /shipments (create shipment)
    ShipmentController->>ShipmentRepository: createShipment(shipmentRecord)
    ShipmentRepository->>MongoDB: Insert ShipmentDetails
    MongoDB->>ShipmentRepository: Return saved shipment

    Note over Customer, MongoDB: Status Progression
    OrderController->>OrderRepository: updateOrderStatus(orderId, STARTED)
    OrderController->>OrderRepository: updateOrderStatus(orderId, BUILT)
    OrderController->>OrderRepository: updateOrderStatus(orderId, SHIPPED)
    
    ShipmentController->>ShipmentRepository: addShipmentEvent(shipmentId, event)
    ShipmentRepository->>MongoDB: Push event to shipment events
    MongoDB->>ShipmentRepository: Return success

    OrderController->>OrderRepository: updateOrderStatus(orderId, DELIVERED)
    OrderController->>OrderRepository: updateOrderStatus(orderId, INSTALLED)
```

## 2. Catalog Synchronization Workflow

### Workflow Description
**Purpose**: Real-time synchronization of product catalog data between MRP system and external inventory systems
**Triggers**: Scheduled task execution or manual catalog updates
**Communication Patterns**: Scheduled background tasks, Azure Queue messaging, REST API calls

```mermaid
sequenceDiagram
    participant Scheduler as Spring Scheduler
    participant UpdateProductTask as UpdateProductProcessTask
    participant MrpConnectService
    participant CatalogController
    participant CatalogRepository
    participant MongoDB
    participant ProductQueue as Azure Product Queue
    participant ExternalSystem as External Inventory System

    Note over Scheduler, ExternalSystem: Catalog Extraction Phase
    Scheduler->>UpdateProductTask: Execute scheduled task (every 5 min)
    UpdateProductTask->>MrpConnectService: getCatalogItems()
    MrpConnectService->>CatalogController: GET /catalog
    CatalogController->>CatalogRepository: getCatalogItems()
    CatalogRepository->>MongoDB: Find all CatalogItems
    MongoDB->>CatalogRepository: Return catalog items
    CatalogRepository->>CatalogController: Return List<CatalogItem>
    CatalogController->>MrpConnectService: Return catalog items
    MrpConnectService->>UpdateProductTask: Return catalog items

    Note over UpdateProductTask, ExternalSystem: Message Publishing Phase
    UpdateProductTask->>UpdateProductTask: Transform CatalogItem to ProductMessage
    UpdateProductTask->>ProductQueue: Publish ProductMessage
    ProductQueue->>ExternalSystem: Deliver ProductMessage
    
    Note over ExternalSystem, MongoDB: External System Processing
    ExternalSystem->>ExternalSystem: Process catalog updates
    ExternalSystem->>MrpConnectService: POST /orders (external orders)
    MrpConnectService->>OrderController: createOrder(order)
    OrderController->>OrderRepository: createOrder(order)
    OrderRepository->>MongoDB: Insert OrderDetails
```

## 3. Quote-to-Order Conversion Workflow

### Workflow Description
**Purpose**: Convert customer quotes into formal orders with pricing validation and dealer coordination
**Triggers**: Customer accepts quote or sales representative converts quote
**Communication Patterns**: REST APIs, synchronous database transactions, dealer validation

```mermaid
sequenceDiagram
    participant SalesRep as Sales Representative
    participant QuoteController
    participant QuoteRepository
    participant DealerRepository
    participant OrderController
    participant OrderRepository
    participant MongoDB

    SalesRep->>QuoteController: POST /quotes (create new quote)
    QuoteController->>QuoteRepository: createQuote(quote)
    QuoteRepository->>DealerRepository: getDealer(dealerName)
    DealerRepository->>MongoDB: Find Dealer by name
    MongoDB->>DealerRepository: Return dealer
    DealerRepository->>QuoteRepository: Return dealer info
    QuoteRepository->>MongoDB: Insert QuoteDetails
    MongoDB->>QuoteRepository: Return saved quote
    QuoteRepository->>QuoteController: Return Quote
    QuoteController->>SalesRep: 201 Created with quote ID

    Note over SalesRep, MongoDB: Quote Acceptance & Conversion
    SalesRep->>QuoteController: PUT /quotes/{id} (update with acceptance)
    QuoteController->>QuoteRepository: updateQuote(quote)
    QuoteRepository->>MongoDB: Update QuoteDetails
    MongoDB->>QuoteRepository: Return updated quote
    
    SalesRep->>OrderController: POST /orders (convert quote to order)
    OrderController->>QuoteController: GET /quotes/{quoteId}
    QuoteController->>QuoteRepository: getQuote(quoteId)
    QuoteRepository->>MongoDB: Find QuoteDetails
    MongoDB->>QuoteRepository: Return quote
    QuoteRepository->>QuoteController: Return Quote
    QuoteController->>OrderController: Return Quote
    
    OrderController->>OrderRepository: createOrderFromQuote(quote)
    OrderRepository->>MongoDB: Insert OrderDetails
    MongoDB->>OrderRepository: Return saved order
    OrderRepository->>OrderController: Return Order
    OrderController->>SalesRep: 201 Created with order ID
```

## 4. Shipment Tracking Workflow

### Workflow Description
**Purpose**: Coordinate and track physical shipments from warehouse to customer installation
**Triggers**: Order status changes to "Built" or manual shipment creation
**Communication Patterns**: REST APIs, event-driven updates, status coordination between orders and shipments

```mermaid
sequenceDiagram
    participant Logistics as Logistics User
    participant ShipmentController
    participant ShipmentRepository
    participant OrderController
    participant OrderRepository
    participant MongoDB

    Logistics->>ShipmentController: POST /shipments (create shipment for order)
    ShipmentController->>OrderController: GET /orders/{orderId}
    OrderController->>OrderRepository: getOrder(orderId)
    OrderRepository->>MongoDB: Find OrderDetails
    MongoDB->>OrderRepository: Return order
    OrderRepository->>OrderController: Return Order
    OrderController->>ShipmentController: Return Order
    
    ShipmentController->>ShipmentRepository: createShipment(shipmentRecord)
    ShipmentRepository->>MongoDB: Insert ShipmentDetails
    MongoDB->>ShipmentRepository: Return saved shipment
    ShipmentRepository->>ShipmentController: Return ShipmentRecord
    ShipmentController->>Logistics: 201 Created

    Note over Logistics, MongoDB: Shipment Event Tracking
    Logistics->>ShipmentController: POST /shipments/{id}/events (add pickup event)
    ShipmentController->>ShipmentRepository: addShipmentEvent(shipmentId, event)
    ShipmentRepository->>MongoDB: Push event to events array
    MongoDB->>ShipmentRepository: Return success
    ShipmentRepository->>ShipmentController: Return updated shipment
    
    ShipmentController->>OrderController: PUT /orders/{orderId}/status (update to SHIPPED)
    OrderController->>OrderRepository: updateOrderStatus(orderId, SHIPPED)
    OrderRepository->>MongoDB: Update OrderDetails status
    MongoDB->>OrderRepository: Return updated order

    Note over Logistics, MongoDB: Delivery Confirmation
    Logistics->>ShipmentController: POST /shipments/{id}/events (add delivery event)
    ShipmentController->>ShipmentRepository: addShipmentEvent(shipmentId, event)
    ShipmentRepository->>MongoDB: Push delivery event
    MongoDB->>ShipmentRepository: Return success
    
    ShipmentController->>OrderController: PUT /orders/{orderId}/status (update to DELIVERED)
    OrderController->>OrderRepository: updateOrderStatus(orderId, DELIVERED)
    OrderRepository->>MongoDB: Update OrderDetails
```

## 5. Error Handling and Recovery Workflow

### Workflow Description
**Purpose**: Handle system failures, data conflicts, and ensure data consistency across distributed operations
**Triggers**: Database conflicts, network failures, invalid requests
**Communication Patterns**: Exception handling, retry mechanisms, optimistic concurrency control

```mermaid
sequenceDiagram
    participant Client
    participant Controller
    participant Repository
    participant MongoDB
    participant Telemetry as AppInsights

    Client->>Controller: PUT /orders/{id} (update order)
    Controller->>Repository: updateOrder(order, etag)
    
    alt ETag Conflict
        Repository->>Repository: Detect ETag mismatch
        Repository->>Controller: Throw ConflictingRequestException
        Controller->>Telemetry: Track exception
        Controller->>Client: 409 Conflict with current ETag
        Client->>Controller: GET /orders/{id} (retrieve current state)
        Controller->>Repository: getOrder(orderId)
        Repository->>MongoDB: Find OrderDetails
        MongoDB->>Repository: Return current order
        Repository->>Controller: Return Order with new ETag
        Controller->>Client: 200 OK with current state
    end

    Note over Client, Telemetry: Retry Mechanism for Database Operations
    Controller->>Repository: createOrder(order)
    Repository->>MongoOperationsWithRetry: executeWithRetry(insertOperation)
    loop Retry on SocketTimeoutException
        MongoOperationsWithRetry->>MongoDB: Insert operation
        alt Socket Timeout
            MongoDB->>MongoOperationsWithRetry: SocketTimeoutException
            MongoOperationsWithRetry->>MongoOperationsWithRetry: Wait and retry (max 3 attempts)
        else Success
            MongoDB->>MongoOperationsWithRetry: Return success
        end
    end
    
    Note over Client, Telemetry: Validation Error Handling
    Client->>Controller: POST /orders (with invalid data)
    Controller->>Utility: validateStringFields(order)
    Utility->>Controller: Return validation errors
    Controller->>Telemetry: Track BadRequestException
    Controller->>Client: 400 Bad Request with error details
    
    Note over Client, Telemetry: System Failure Recovery
    Controller->>Repository: getOrder(nonExistentId)
    Repository->>MongoDB: Find OrderDetails
    MongoDB->>Repository: Return null
    Repository->>Controller: Return null
    Controller->>Client: 404 Not Found
```

## 6. Integration Service Background Processing

### Workflow Description
**Purpose**: Automated background synchronization between external systems and MRP core services
**Triggers**: Scheduled task execution at fixed intervals
**Communication Patterns**: Scheduled tasks, Azure Queue consumption, batch processing

```mermaid
sequenceDiagram
    participant Scheduler as Spring Scheduler
    participant OrderTask as CreateOrderProcessTask
    participant ProductTask as UpdateProductProcessTask
    participant QueueService
    participant OrderQueue as Azure Order Queue
    participant ProductQueue as Azure Product Queue
    participant MrpConnectService
    participant ExternalSystem

    Note over Scheduler, ExternalSystem: Order Processing Cycle (every 30s)
    Scheduler->>OrderTask: Execute order processing
    OrderTask->>QueueService: getMessages(OrderQueue)
    QueueService->>OrderQueue: Retrieve OrderMessages
    OrderQueue->>QueueService: Return OrderMessages
    QueueService->>OrderTask: Return deserialized messages
    
    loop For each OrderMessage
        OrderTask->>MrpConnectService: createOrderFromMessage(message)
        MrpConnectService->>OrderController: createOrder(order)
        OrderController->>OrderRepository: createOrder(order)
        OrderRepository->>MongoDB: Insert OrderDetails
        MongoDB->>OrderRepository: Return saved order
        OrderRepository->>OrderController: Return Order
        OrderController->>MrpConnectService: Return Order
        MrpConnectService->>OrderTask: Return success
        OrderTask->>QueueService: deleteMessage(OrderQueue, message)
        QueueService->>OrderQueue: Delete processed message
    end

    Note over Scheduler, ExternalSystem: Product Synchronization Cycle (every 5 min)
    Scheduler->>ProductTask: Execute product update
    ProductTask->>MrpConnectService: getCatalogItems()
    MrpConnectService->>CatalogController: GET /catalog
    CatalogController->>CatalogRepository: getCatalogItems()
    CatalogRepository->>MongoDB: Find all CatalogItems
    MongoDB->>CatalogRepository: Return catalog items
    CatalogRepository->>CatalogController: Return List<CatalogItem>
    CatalogController->>MrpConnectService: Return catalog items
    MrpConnectService->>ProductTask: Return catalog items
    
    ProductTask->>ProductTask: Transform to ProductMessage
    ProductTask->>QueueService: addMessage(ProductQueue, message)
    QueueService->>ProductQueue: Publish ProductMessage
    ProductQueue->>ExternalSystem: Deliver ProductMessage
```