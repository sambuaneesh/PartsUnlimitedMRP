
## 1. Customer Order Processing Workflow

```mermaid
sequenceDiagram
    participant Customer as Customer (Website)
    participant OrderController as OrderController
    participant QuoteController as QuoteController
    participant QuoteRepo as QuoteRepository
    participant OrderRepo as OrderRepository
    participant ShipmentRepo as ShipmentRepository
    participant MongoDB as MongoDB
    participant QueueService as QueueService
    participant IntegrationSvc as Integration Service
    participant ExternalMRP as External MRP System
    participant AppInsights as App Insights

    Note over Customer,ExternalMRP: Customer Order Processing Workflow
    
    Customer->>QuoteController: POST /api/quotes (Create quote)
    QuoteController->>QuoteRepo: createQuote(quote)
    QuoteRepo->>MongoDB: save(QuoteDetails)
    MongoDB-->>QuoteRepo: Quote saved with ID
    QuoteRepo-->>QuoteController: Quote object
    QuoteController-->>Customer: 201 Created (Quote)
    
    Customer->>OrderController: POST /api/orders (Convert quote to order)
    OrderController->>QuoteRepo: getQuote(quoteId)
    QuoteRepo->>MongoDB: find(quoteId)
    MongoDB-->>QuoteRepo: QuoteDetails
    QuoteRepo-->>OrderController: Quote object
    
    OrderController->>OrderRepo: createOrderFromQuote(quote)
    OrderRepo->>MongoDB: save(OrderDetails)
    MongoDB-->>OrderRepo: Order saved with ID
    OrderRepo-->>OrderController: Order object
    
    Note over OrderController,MongoDB: Order status set to "Created"
    
    OrderController-->>Customer: 201 Created (Order)
    
    OrderController->>QueueService: enqueue(OrderMessage)
    QueueService->>QueueFactory: getCloudQueue("orders")
    QueueFactory-->>QueueService: CloudQueue instance
    QueueService->>QueueService: serialize(OrderMessage to JSON)
    QueueService->>CloudQueue: addMessage(OrderMessage JSON)
    CloudQueue-->>QueueService: Message queued
    
    Note over IntegrationSvc,ExternalMRP: Async processing by Integration Service
    
    loop Scheduled Processing
        IntegrationSvc->>QueueService: dequeue("orders")
        QueueService->>CloudQueue: retrieveMessages()
        CloudQueue-->>QueueService: OrderMessage
        QueueService->>QueueService: deserializeJSON()
        QueueService-->>IntegrationSvc: OrderMessage object
        
        IntegrationSvc->>ExternalMRP: POST /api/orders (Create in MRP)
        ExternalMRP-->>IntegrationSvc: 200 OK (Order created)
        
        IntegrationSvc->>QueueService: deleteMessage()
        QueueService->>CloudQueue: deleteMessage(messageId)
        CloudQueue-->>QueueService: Message deleted
    end
    
    Note over IntegrationSvc,AppInsights: Error Handling and Monitoring
    alt Request fails
        IntegrationSvc->>AppInsights: trackException(error)
        IntegrationSvc->>QueueService: addMessageToDeadLetterQueue()
    end
```

### Workflow Description
The customer order processing workflow handles the complete lifecycle from quote generation to order creation and integration with external MRP systems. It starts with a customer requesting a quote, which is converted to an order, then processed asynchronously through message queues.

### Communication Patterns
- **Synchronous REST calls**: Customer -> Controllers, Controllers -> Repositories
- **Asynchronous messaging**: QueueService between ordering and integration services
- **Database transactions**: MongoDB operations via repositories
- **External API calls**: Integration service -> External MRP system

## 2. Product Catalog Synchronization Workflow

```mermaid
sequenceDiagram
    participant Scheduler as Scheduled Task
    participant UpdateProductTask as UpdateProductProcessTask
    participant MrpConnectSvc as MrpConnectService
    participant ExternalMRP as External MRP System
    participant QueueFactory as QueueFactory
    participant QueueService as QueueService
    participant ProductQueue as Azure Product Queue
    participant OrderingSvc as Ordering Service
    participant CatalogController as CatalogController
    participant CatalogRepo as CatalogRepository
    participant MongoDB as MongoDB

    Note over Scheduler,MongoDB: Product Catalog Synchronization Workflow
    
    Note over Scheduler: Triggered by @Scheduled annotation
    Scheduler->>UpdateProductTask: execute()
    
    UpdateProductTask->>MrpConnectSvc: getProductCatalog()
    MrpConnectSvc->>ExternalMRP: GET /api/products
    ExternalMRP-->>MrpConnectSvc: Product catalog data
    MrpConnectSvc-->>UpdateProductTask: List<CatalogItem>
    
    UpdateProductTask->>UpdateProductTask: validateProducts()
    
    loop For each product
        UpdateProductTask->>QueueService: enqueue(ProductMessage)
        QueueService->>QueueFactory: getCloudQueue("products")
        QueueFactory-->>QueueService: CloudQueue instance
        QueueService->>ProductQueue: addMessage(ProductItem JSON)
        ProductQueue-->>QueueService: Message queued
    end
    
    Note over OrderingSvc,MongoDB: Processing in Ordering Service
    
    loop Message Processing
        OrderingSvc->>QueueService: dequeue("products")
        QueueService->>ProductQueue: retrieveMessages()
        ProductQueue-->>QueueService: ProductMessage
        QueueService->>QueueService: deserializeJSON()
        QueueService-->>OrderingSvc: ProductMessage object
        
        OrderingSvc->>CatalogController: updateCatalog(productItems)
        CatalogController->>CatalogRepo: upsertCatalogItem(item)
        CatalogRepo->>MongoDB: saveOrUpdate(CatalogItem)
        MongoDB-->>CatalogRepo: Item saved/updated
        CatalogRepo-->>CatalogController: Success
        CatalogController-->>OrderingSvc: 200 OK
        
        OrderingSvc->>QueueService: deleteMessage()
        QueueService->>ProductQueue: deleteMessage(messageId)
    end
    
    Note over UpdateProductTask,ExternalMRP: Error Handling
    alt API call fails
        UpdateProductTask->>UpdateProductTask: log.error()
        UpdateProductTask->>UpdateProductTask: scheduleRetry()
    end
    
    alt Product validation fails
        UpdateProductTask->>UpdateProductTask: skipInvalidProduct()
        UpdateProductTask->>UpdateProductTask: logValidationFailure()
    end
```

### Workflow Description
This workflow synchronizes product catalog data from the external MRP system to maintain consistency across the manufacturing supply chain. It runs on a scheduled basis, fetching updates and processing them asynchronously.

### Communication Patterns
- **Scheduled polling**: Timer-based task execution
- **Synchronous REST**: External MRP API calls
- **Asynchronous messaging**: Queue-based product updates
- **Database transactions**: MongoDB upsert operations
- **Retry mechanisms**: Built-in retry logic for transient failures

## 3. Shipment Tracking Workflow

```mermaid
sequenceDiagram
    participant Customer as Customer
    participant OrderController as OrderController
    participant OrderRepo as OrderRepository
    participant ShipmentController as ShipmentController
    participant ShipmentRepo as ShipmentRepository
    participant MongoDB as MongoDB
    participant LogisticsAPI as External Logistics API
    participant AppInsights as App Insights
    participant NotificationSvc as Notification Service

    Note over Customer,NotificationSvc: Shipment Tracking Workflow
    
    Note over OrderController,MongoDB: Order Ready for Shipment
    OrderController->>OrderRepo: updateOrderStatus(orderId, "Shipped")
    OrderRepo->>MongoDB: update(OrderDetails)
    MongoDB-->>OrderRepo: Order updated
    OrderRepo-->>OrderController: Success
    
    Note over OrderController,ShipmentRepo: Create Shipment Record
    OrderController->>ShipmentController: createShipment(order)
    ShipmentController->>ShipmentRepo: createShipment(order)
    ShipmentRepo->>MongoDB: save(ShipmentDetails)
    MongoDB-->>ShipmentRepo: Shipment created
    ShipmentRepo-->>ShipmentController: ShipmentRecord
    ShipmentController-->>OrderController: 201 Created
    
    Note over ShipmentController,LogisticsAPI: External Shipment Tracking
    
    ShipmentController->>LogisticsAPI: POST /api/shipments (Create tracking)
    LogisticsAPI-->>ShipmentController: Tracking number
    
    ShipmentController->>ShipmentRepo: updateShipment(trackingNumber)
    ShipmentRepo->>MongoDB: update(ShipmentDetails)
    MongoDB-->>ShipmentRepo: Updated
    
    loop Shipment Events
        Note over LogisticsAPI,ShipmentRepo: Polling or Webhook Events
        LogisticsAPI->>ShipmentController: shipmentEvent(eventData)
        ShipmentController->>ShipmentRepo: addShipmentEvent(shipmentId, event)
        ShipmentRepo->>MongoDB: pushToEventsList(ShipmentEventInfo)
        MongoDB-->>ShipmentRepo: Event added
        ShipmentRepo-->>ShipmentController: Success
        
        ShipmentController->>NotificationSvc: notifyCustomer(shipmentUpdate)
        NotificationSvc-->>Customer: Email/SMS notification
    end
    
    Customer->>ShipmentController: GET /api/shipments/{id}
    ShipmentController->>ShipmentRepo: getShipment(id)
    ShipmentRepo->>MongoDB: find(shipmentId)
    MongoDB-->>ShipmentRepo: ShipmentDetails
    ShipmentRepo-->>ShipmentController: ShipmentRecord
    ShipmentController->>ShipmentRepo: enrichWithOrderData(shipment)
    ShipmentRepo->>OrderRepo: getOrder(orderId)
    OrderRepo-->>ShipmentRepo: Order object
    ShipmentRepo-->>ShipmentController: Enriched shipment
    ShipmentController-->>Customer: 200 OK (Shipment with order details)
    
    Note over ShipmentController,AppInsights: Error Handling
    alt Shipment not found
        ShipmentController->>AppInsights: trackException("ShipmentNotFound")
        ShipmentController-->>Customer: 404 Not Found
    end
    
    alt Logistics API fails
        ShipmentController->>AppInsights: trackDependencyFailure()
        ShipmentController->>ShipmentRepo: updateShipmentStatus("Failed")
    end
```

### Workflow Description
The shipment tracking workflow manages the complete lifecycle of order fulfillment, from shipment creation through delivery tracking. It integrates with external logistics providers and maintains an audit trail of all shipment events.

### Communication Patterns
- **Synchronous REST**: Customer queries, internal API calls
- **Database transactions**: MongoDB operations for persistence
- **External API integration**: Logistics provider APIs
- **Event-driven updates**: Shipment status changes
- **Notification system**: Customer notifications via separate service

## 4. Error Handling and Recovery Pattern

```mermaid
sequenceDiagram
    participant Client as Client/Controller
    participant BusinessLogic as Business Service
    participant Repo as Repository
    participant MongoDB as MongoDB
    participant QueueService as QueueService
    participant DeadLetterQueue as Dead Letter Queue
    participant AppInsights as App Insights
    participant AlertSystem as Alert System

    Note over Client,AlertSystem: Error Handling and Recovery Pattern
    
    Client->>BusinessLogic: processRequest(data)
    
    alt Input Validation Error
        BusinessLogic->>BusinessLogic: validateInput()
        BusinessLogic->>Client: throw BadRequestException("Invalid data")
        Client->>AppInsights: trackException(BadRequestException)
        Client-->>Client: 400 Bad Request
    else Business Logic Error
        BusinessLogic->>Repo: saveData()
        Repo->>MongoDB: execute operation
        MongoDB--xRepo: Database error/timeout
        Repo->>AppInsights: trackException(DatabaseError)
        Repo->>Client: throw ConflictingRequestException("System error")
        Client-->>Client: 409 Conflict
    else Concurrency Conflict
        Repo->>MongoDB: updateWithETag(data)
        MongoDB--xRepo: ETag mismatch
        Repo->>AppInsights: trackException(ETagConflict)
        Repo->>Client: throw ConflictingRequestException("Data modified")
        Client-->>Client: 409 Conflict
    end
    
    Note over QueueService,DeadLetterQueue: Queue Processing Error Recovery
    
    QueueService->>QueueService: processQueuedMessage()
    
    alt Transient Error (Retryable)
        QueueService->>QueueService: operation fails with timeout
        QueueService->>QueueService: retryWithBackoff()
        Note over QueueService: Exponential backoff retry
        alt Retry succeeds
            QueueService->>QueueService: process completes
        else Max retries exceeded
            QueueService->>DeadLetterQueue: moveToDeadLetter(message)
            QueueService->>AppInsights: trackMetric("DeadLetterMessages")
            QueueService->>AlertSystem: sendAlert("Queue processing failed")
        end
    else Non-transient Error (Non-retryable)
        QueueService->>DeadLetterQueue: moveToDeadLetter(message)
        QueueService->>AppInsights: trackException("Permanent failure")
    end
    
    Note over BusinessLogic,MongoDB: Circuit Breaker Pattern
    BusinessLogic->>Repo: databaseOperation()
    
    alt Multiple failures detected
        Repo->>Repo: circuitBreaker.open()
        Repo->>BusinessLogic: CircuitBreakerOpenException
        BusinessLogic->>AppInsights: trackDependencyFailure()
        BusinessLogic->>AlertSystem: sendAlert("Database unavailable")
        BusinessLogic->>BusinessLogic: useFallbackData()
    end
    
    Note over Client,AlertSystem: System Recovery
    AlertSystem->>AlertSystem: monitorSystemHealth()
    AlertSystem->>AlertSystem: detectRecovery()
    AlertSystem->>QueueService: resumeProcessing()
    AlertSystem->>Repo: resetCircuitBreaker()
    
    BusinessLogic->>Repo: databaseOperation()
    Repo->>MongoDB: successful operation
    MongoDB-->>Repo: Success
    Repo-->>BusinessLogic: Result
    BusinessLogic-->>Client: 200 OK
```

### Workflow Description
This workflow demonstrates the comprehensive error handling and recovery patterns implemented throughout the MRP system, including input validation, database error handling, queue processing failures, and circuit breaker patterns for resilience.

### Communication Patterns
- **Exception propagation**: Structured error handling through custom exceptions
- **Telemetry integration**: All errors tracked in Application Insights
- **Dead letter queue**: Failed messages isolated for manual review
- **Circuit breaker**: Prevents cascade failures during outages
- **Alerting system**: Notifications for critical failures
- **Retry mechanisms**: Exponential backoff for transient failures

## 5. End-to-End Order Fulfillment Flow

```mermaid
sequenceDiagram
    participant Customer as Customer
    participant WebFrontend as Web Frontend
    participant OrderController as OrderController
    participant IntegrationSvc as Integration Service
    participant ProductionSystem as Production System
    participant ShipmentController as ShipmentController
    participant Supplier as Supplier Portal
    participant Logistics as Logistics Provider
    participant Monitoring as Monitoring System

    Note over Customer,Monitoring: End-to-End Order Fulfillment Flow
    
    Customer->>WebFrontend: Browse catalog & place order
    WebFrontend->>OrderController: POST /api/orders
    OrderController->>IntegrationSvc: enqueue(OrderMessage)
    
    Note over IntegrationSvc,ProductionSystem: Production Planning
    IntegrationSvc->>ProductionSystem: Submit production order
    ProductionSystem-->>IntegrationSvc: Production schedule
    
    IntegrationSvc->>Supplier: Order raw materials
    Supplier-->>IntegrationSvc: Order confirmation
    
    Note over ProductionSystem,ShipmentController: Manufacturing & Assembly
    ProductionSystem->>ProductionSystem: Manufacture parts
    ProductionSystem->>ProductionSystem: Quality inspection
    ProductionSystem->>ShipmentController: readyForShipment(order)
    
    ShipmentController->>Logistics: Arrange delivery
    Logistics-->>ShipmentController: Tracking number & schedule
    
    loop Shipment Progress
        Logistics->>ShipmentController: shipmentStatusUpdate(event)
        ShipmentController->>WebFrontend: real-time status update
        WebFrontend->>Customer: Notification (Email/SMS)
    end
    
    Logistics->>Customer: Package delivered
    Customer->>WebFrontend: Confirm delivery
    WebFrontend->>ShipmentController: updateDeliveryStatus()
    
    Note over Monitoring: Continuous Monitoring
    Monitoring->>Monitoring: trackOrderMetrics()
    Monitoring->>Monitoring: analyzeSLACompliance()
    Monitoring->>Monitoring: generateReports()
    
    Monitoring->>Monitoring: detectAnomalies()
    alt Anomaly detected
        Monitoring->>OrderController: triggerInvestigation()
        OrderController->>OrderController: manualReviewRequired()
    end
```

### Workflow Description
This comprehensive workflow shows the complete order fulfillment process from customer order to final delivery, including production planning, supplier coordination, manufacturing, logistics, and continuous monitoring.

### Communication Patterns
- **Web interface**: Customer interactions through frontend
- **Service integration**: Multiple system coordination
- **External APIs**: Supplier and logistics provider integration
- **Real-time updates**: Live status notifications
- **Continuous monitoring**: System performance and SLA tracking