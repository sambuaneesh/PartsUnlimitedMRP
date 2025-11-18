
## 1. Quote Creation Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as WinJS Frontend (9080)
    participant OrderService as Order Service (8080)
    participant MongoDB
    participant Telemetry as Application Insights

    User->>Frontend: Navigate to Catalog Page
    Frontend->>Frontend: Load catalog view
    Frontend->>OrderService: GET /catalog
    OrderService->>MongoDB: Find catalog items
    MongoDB-->>OrderService: Return catalog data
    OrderService-->>Frontend: JSON response with items
    Frontend-->>User: Display catalog with items
    
    User->>Frontend: Select items and create quote
    Frontend->>Frontend: Calculate quote totals
    Frontend->>OrderService: POST /quotes
    Note over Frontend,OrderService: Request body: customerName, dealerName, items[]
    
    OrderService->>OrderService: Validate quote data
    OrderService->>OrderService: Set quoteId and validUntil date
    OrderService->>MongoDB: Insert quote document
    MongoDB-->>OrderService: Success confirmation
    OrderService->>Telemetry: Track quote creation event
    OrderService-->>Frontend: 201 Created + quote data
    Frontend-->>User: Show quote confirmation
```

## 2. Order Creation from Quote Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as WinJS Frontend
    participant OrderService as Order Service (8080)
    participant QuoteRepo as QuoteRepository
    participant OrderRepo as OrderRepository
    participant MongoDB
    participant Telemetry

    User->>Frontend: Select quote to convert to order
    Frontend->>OrderService: GET /quotes/{id}
    OrderService->>MongoDB: Find quote by ID
    MongoDB-->>OrderService: Return quote details
    OrderService-->>Frontend: Quote data
    
    User->>Frontend: Confirm order creation
    Frontend->>OrderService: POST /orders?fromQuote={quoteId}
    
    OrderService->>QuoteRepo: Find quote by ID
    QuoteRepo->>MongoDB: Query quotes collection
    MongoDB-->>QuoteRepo: Return quote
    QuoteRepo-->>OrderService: Quote object
    
    alt Quote valid and not expired
        OrderService->>OrderService: Create order from quote data
        OrderService->>OrderService: Set orderId = "order-{quoteId}"
        OrderService->>OrderService: Set initial status = "Created"
        OrderService->>OrderRepo: Save new order
        OrderRepo->>MongoDB: Insert into orders collection
        MongoDB-->>OrderRepo: Success
        
        OrderService->>Telemetry: Track order creation
        OrderService-->>Frontend: 201 Created + order data
        Frontend-->>User: Display order confirmation
    else Quote not found or expired
        OrderService-->>Frontend: 404 Not Found or 400 Bad Request
        Frontend-->>User: Show error message
    end
```

## 3. Shipment Tracking and Events Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as WinJS Frontend
    participant OrderService as Order Service
    participant OrderRepo as OrderRepository
    participant ShipmentRepo as ShipmentRepository
    participant MongoDB
    participant Telemetry

    User->>Frontend: Navigate to Deliveries page
    Frontend->>OrderService: GET /shipments
    OrderService->>MongoDB: Find all shipments
    MongoDB-->>OrderService: Shipments array
    OrderService-->>Frontend: Shipments data
    Frontend-->>User: Display shipment list
    
    User->>Frontend: Click on specific shipment
    Frontend->>OrderService: GET /shipments/{id}
    OrderService->>MongoDB: Find shipment by ID
    MongoDB-->>OrderService: Shipment details with events
    OrderService-->>Frontend: Shipment data
    Frontend-->>User: Show shipment details and events
    
    Note over User,Telemetry: External system triggers shipment update
    participant ExternalSystem as External Tracking System
    ExternalSystem->>OrderService: POST /shipments/{id}/events
    Note over ExternalSystem,OrderService: Event: "In Transit", "Out for Delivery", etc.
    
    OrderService->>ShipmentRepo: Validate shipment exists
    ShipmentRepo->>MongoDB: Query shipments collection
    MongoDB-->>ShipmentRepo: Shipment found
    ShipmentRepo-->>OrderService: Shipment validation passed
    
    OrderService->>OrderService: Validate event data
    OrderService->>MongoDB: Push event to events array
    MongoDB-->>OrderService: Update confirmed
    
    OrderService->>Telemetry: Track shipment event
    OrderService-->>ExternalSystem: 200 OK
    
    Note over Frontend: Frontend polls or receives update
    Frontend->>OrderService: GET /shipments/{id}/events
    OrderService->>MongoDB: Query shipment events
    MongoDB-->>OrderService: Events array
    OrderService-->>Frontend: Updated events
    Frontend-->>User: Real-time event update
```

## 4. Integration Service - Inventory Update Workflow

```mermaid
sequenceDiagram
    participant ExternalWebsite as Parts Unlimited Website
    participant AzureQueue as Azure Queue Service
    participant IntegrationService as Integration Service
    participant ScheduledTask as UpdateProductProcessTask
    participant OrderService as Order Service (8080)
    participant MongoDB
    participant Telemetry

    ExternalWebsite->>AzureQueue: Send ProductMessage (inventory update)
    Note over ExternalWebsite,AzureQueue: Message contains: skuNumber, newInventory, price
    
    IntegrationService->>AzureQueue: Poll queue for messages
    AzureQueue-->>IntegrationService: ProductMessage
    
    IntegrationService->>IntegrationService: Transform message to CatalogItem
    IntegrationService->>OrderService: PUT /catalog/{skuNumber}
    Note over IntegrationService,OrderService: Request body: updated inventory and price
    
    OrderService->>MongoDB: Update catalog item
    MongoDB-->>OrderService: Update confirmation
    OrderService-->>IntegrationService: 200 OK
    
    IntegrationService->>Telemetry: Track inventory update
    
    Note over ScheduledTask: Scheduled task runs periodically
    loop Every 5 minutes
        ScheduledTask->>IntegrationService: Trigger product sync
        IntegrationService->>OrderService: GET /catalog
        OrderService->>MongoDB: Fetch all catalog items
        MongoDB-->>OrderService: Catalog data
        OrderService-->>IntegrationService: Catalog array
        IntegrationService->>IntegrationService: Sync with external systems
    end
```

## 5. Error Handling and Recovery Patterns

```mermaid
sequenceDiagram
    participant Client as Frontend Client
    participant OrderService as Order Service
    participant MongoDB
    participant RetryLogic as MongoOperationsWithRetry
    participant Telemetry as Application Insights

    Client->>OrderService: POST /orders
    OrderService->>MongoDB: Insert order document
    
    alt MongoDB unavailable (first attempt)
        MongoDB-->>RetryLogic: Connection timeout
        RetryLogic->>Telemetry: Log exception
        RetryLogic->>RetryLogic: Wait (exponential backoff)
        RetryLogic->>MongoDB: Retry operation (attempt 2)
    end
    
    alt Still failing after retries
        MongoDB-->>RetryLogic: Connection refused
        RetryLogic->>Telemetry: Log failure after 3 attempts
        RetryLogic-->>OrderService: Throw exception
        OrderService-->>Client: 503 Service Unavailable
    else Success on retry
        MongoDB-->>RetryLogic: Success
        RetryLogic-->>OrderService: Result
        OrderService-->>Client: 201 Created
    end
    
    Note over Client,Telemetry: Validation error scenario
    Client->>OrderService: POST /quotes (invalid data)
    OrderService->>OrderService: Validate quote model
    OrderService->>Telemetry: Track validation errors
    OrderService-->>Client: 400 Bad Request + error details
```

## 6. Order Status Progression Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as WinJS Frontend
    participant OrderService as Order Service
    participant MongoDB
    participant NotificationService as Notification Service
    participant Telemetry

    User->>Frontend: View order details
    Frontend->>OrderService: GET /orders/{id}
    OrderService->>MongoDB: Find order by ID
    MongoDB-->>OrderService: Order with current status
    OrderService-->>Frontend: Order data
    Frontend-->>User: Display order with status
    
    Note over User,Telemetry: Order status update (manual trigger)
    User->>Frontend: Update order status
    Frontend->>OrderService: PUT /orders/{id}/status
    Note over Frontend,OrderService: Body: {status: "Confirmed"}
    
    OrderService->>OrderService: Validate status transition
    Note right of OrderService: Created->Confirmed->Started->Built->Shipped->Delivered
    
    alt Valid transition
        OrderService->>MongoDB: Update order status
        OrderService->>MongoDB: Push event with timestamp
        MongoDB-->>OrderService: Update confirmed
        
        OrderService->>NotificationService: Send status update notification
        OrderService->>Telemetry: Track status change
        
        OrderService-->>Frontend: 200 OK
        Frontend-->>User: Show updated status
    else Invalid transition
        OrderService-->>Frontend: 400 Bad Request
        Frontend-->>User: Display error message
    end
    
    Note over Frontend: Real-time status poll
    loop Every 30 seconds
        Frontend->>OrderService: GET /orders/{id}/events
        OrderService->>MongoDB: Query order events
        MongoDB-->>OrderService: Events array
        OrderService-->>Frontend: Event data
        Frontend->>Frontend: Update UI with new events
    end
```