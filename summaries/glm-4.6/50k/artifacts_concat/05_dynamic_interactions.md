
# Parts Unlimited MRP - Dynamic Interaction Flows and Sequence Diagrams

## 1. Quote Generation Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Web Client (WinJS)
    participant OrderService as Order Service (Spring Boot)
    participant MongoDB as MongoDB Database
    participant CatalogRepo as Catalog Repository
    participant DealerRepo as Dealer Repository
    
    User->>Frontend: Navigate to Quotes Page
    Frontend->>Frontend: Initialize quote form
    User->>Frontend: Select dealer & add items
    Frontend->>OrderService: GET /api/dealers
    OrderService->>DealerRepo: Fetch dealer info
    DealerRepo-->>MongoDB: Query dealers collection
    MongoDB-->>DealerRepo: Return dealer data
    DealerRepo-->>OrderService: Dealer info
    OrderService-->>Frontend: 200 OK with dealer list
    
    User->>Frontend: Add product items (SKU, quantity)
    Frontend->>OrderService: GET /api/catalog
    OrderService->>CatalogRepo: Fetch product catalog
    CatalogRepo-->>MongoDB: Query catalog collection
    MongoDB-->>CatalogRepo: Return product data
    CatalogRepo-->>OrderService: Product catalog
    OrderService-->>Frontend: 200 OK with catalog
    
    Frontend->>Frontend: Validate inventory levels locally
    User->>Frontend: Submit quote request
    Frontend->>OrderService: POST /api/quotes {quoteData}
    
    OrderService->>OrderService: Validate dealer exists
    OrderService->>DealerRepo: Check dealer
    DealerRepo-->>MongoDB: Verify dealer
    MongoDB-->>DealerRepo: Dealer validation
    DealerRepo-->>OrderService: Valid dealer
    
    OrderService->>OrderService: Calculate total cost
    OrderService->>OrderService: Apply discount rules
    OrderService->>OrderService: Set expiry date (30 days)
    
    OrderService->>MongoDB: Insert quote document
    MongoDB-->>OrderService: Quote created with quoteId
    OrderService-->>Frontend: 201 Created with quote response
    Frontend-->>User: Display quote confirmation
    
    Note over Frontend,MongoDB: Communication Pattern: Synchronous REST API calls
```

## 2. Order Creation from Quote Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Web Client (WinJS)
    participant OrderService as Order Service (Spring Boot)
    participant QuoteRepo as Quote Repository
    participant OrderRepo as Order Repository
    participant ShipmentRepo as Shipment Repository
    participant CatalogRepo as Catalog Repository
    participant MongoDB as MongoDB Database
    
    User->>Frontend: Select quote to convert
    Frontend->>OrderService: GET /api/quotes/{quoteId}
    OrderService->>QuoteRepo: Fetch quote
    QuoteRepo-->>MongoDB: Query quotes collection
    MongoDB-->>QuoteRepo: Quote document
    QuoteRepo-->>OrderService: Quote data
    OrderService-->>Frontend: 200 OK with quote
    
    Frontend->>Frontend: Display quote details
    User->>Frontend: Click "Create Order"
    Frontend->>OrderService: POST /api/orders {quoteId, orderData}
    
    OrderService->>QuoteRepo: Get quote by ID
    QuoteRepo-->>MongoDB: Query quotes collection
    MongoDB-->>QuoteRepo: Quote document
    QuoteRepo-->>OrderService: Quote
    
    OrderService->>OrderService: Validate quote not expired
    OrderService->>OrderService: Check if order exists for quote
    alt Quote already has order
        OrderService-->>Frontend: 409 Conflict - Order exists
    else Valid quote
        OrderService->>CatalogRepo: Reserve inventory for items
        loop For each quote item
            CatalogRepo->>MongoDB: Update inventory levels
            MongoDB-->>CatalogRepo: Updated inventory
        end
        
        OrderService->>OrderRepo: Create order document
        OrderRepo-->>MongoDB: Insert order with status "Created"
        MongoDB-->>OrderRepo: Order with orderId
        OrderRepo-->>OrderService: Order created
        
        OrderService->>ShipmentRepo: Create shipment record
        ShipmentRepo-->>MongoDB: Insert shipment linked to order
        MongoDB-->>ShipmentRepo: Shipment record
        ShipmentRepo-->>OrderService: Shipment created
        
        OrderService->>OrderRepo: Add OrderEventInfo ("Created")
        OrderRepo-->>MongoDB: Update order with event
        MongoDB-->>OrderRepo: Event logged
        OrderRepo-->>OrderService: Update complete
        
        OrderService-->>Frontend: 201 Created with order response
    end
    
    Frontend-->>User: Display order confirmation
    Frontend->>Frontend: Navigate to Orders page
    
    Note over Frontend,MongoDB: Communication Pattern: Synchronous REST with DB transactions
```

## 3. Integration Processing Workflow (External Website)

```mermaid
sequenceDiagram
    participant Website as Parts Unlimited Website
    participant AzureQueue as Azure Storage Queue
    participant IntegrationService as Integration Service (Spring Boot)
    participant OrderService as Order Service (Spring Boot)
    participant MongoDB as MongoDB Database
    participant Scheduler as Scheduled Task
    
    Note over Website,Scheduler: Continuous processing every 30 seconds
    
    Website->>AzureQueue: Add order message (MRP format)
    
    Scheduler->>IntegrationService: Trigger CreateOrderProcessTask
    IntegrationService->>AzureQueue: Get message from queue
    AzureQueue-->>IntegrationService: Order message
    
    IntegrationService->>IntegrationService: Transform message format
    IntegrationService->>OrderService: POST /api/orders (internal)
    
    OrderService->>OrderService: Process order as internal request
    OrderService->>MongoDB: Create order and related records
    MongoDB-->>OrderService: Order created
    OrderService-->>IntegrationService: 201 Created
    
    IntegrationService->>AzureQueue: Delete processed message
    IntegrationService->>IntegrationService: Log successful processing
    
    Note over IntegrationService,MongoDB: Product Catalog Sync (separate task)
    
    Scheduler->>IntegrationService: Trigger UpdateProductProcessTask
    IntegrationService->>OrderService: GET /api/catalog
    OrderService-->>IntegrationService: Current catalog
    IntegrationService->>IntegrationService: Transform catalog data
    IntegrationService->>AzureQueue: Add catalog update message
    
    Note over Website,AzureQueue: Communication Pattern: Asynchronous queue-based messaging
```

## 4. Shipment Tracking and Status Update Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Web Client (WinJS)
    participant OrderService as Order Service (Spring Boot)
    participant ShipmentRepo as Shipment Repository
    participant OrderRepo as Order Repository
    participant MongoDB as MongoDB Database
    participant ExternalSystem as Shipping System
    
    User->>Frontend: Navigate to Deliveries page
    Frontend->>OrderService: GET /api/shipments/{orderId}
    OrderService->>ShipmentRepo: Fetch shipment by orderId
    ShipmentRepo-->>MongoDB: Query shipments collection
    MongoDB-->>ShipmentRepo: Shipment document
    ShipmentRepo-->>OrderService: Shipment data with events
    OrderService-->>Frontend: 200 OK with shipment info
    
    Frontend-->>User: Display shipment status
    
    Note over ExternalSystem,MongoDB: External status update triggered
    
    ExternalSystem->>OrderService: POST /api/shipments/{id}/events
    Note right of ExternalSystem: Webhook or API call
    
    OrderService->>OrderService: Validate event data
    OrderService->>ShipmentRepo: Add ShipmentEventInfo
    ShipmentRepo-->>MongoDB: Update shipment with new event
    MongoDB-->>ShipmentRepo: Event added
    ShipmentRepo-->>OrderService: Update successful
    
    OrderService->>OrderRepo: Update order status based on shipment
    alt Event is "Delivered"
        OrderService->>OrderRepo: Update order status to "Delivered"
        OrderRepo-->>MongoDB: Update order document
        MongoDB-->>OrderRepo: Status updated
    else Event is "Shipped"
        OrderService->>OrderRepo: Update order status to "Shipped"
        OrderRepo-->>MongoDB: Update order document
        MongoDB-->>OrderRepo: Status updated
    end
    
    OrderService-->>ExternalSystem: 200 OK - Event processed
    
    Note over OrderService,MongoDB: Communication Pattern: Event-driven status updates
```

## 5. Catalog Management with Inventory Check Workflow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Web Client (WinJS)
    participant OrderService as Order Service (Spring Boot)
    participant CatalogRepo as Catalog Repository
    participant MongoDB as MongoDB Database
    participant QuoteService as Quote Service (internal)
    
    User->>Frontend: Navigate to Catalog page
    Frontend->>OrderService: GET /api/catalog
    OrderService->>CatalogRepo: Fetch all catalog items
    CatalogRepo-->>MongoDB: Query catalog collection
    MongoDB-->>CatalogRepo: All products
    CatalogRepo-->>OrderService: Product list
    OrderService-->>Frontend: 200 OK with catalog
    
    Frontend-->>User: Display catalog with inventory levels
    
    User->>Frontend: Click "Edit" on product
    Frontend->>OrderService: GET /api/catalog/{skuNumber}
    OrderService->>CatalogRepo: Get specific product
    CatalogRepo-->>MongoDB: Query by SKU
    MongoDB-->>CatalogRepo: Product details
    CatalogRepo-->>OrderService: Product data
    OrderService-->>Frontend: 200 OK with product
    
    User->>Frontend: Update price/inventory
    Frontend->>OrderService: PUT /api/catalog/{skuNumber} {productData}
    
    OrderService->>OrderService: Validate input data
    OrderService->>CatalogRepo: Check current inventory
    CatalogRepo-->>MongoDB: Get current stock
    MongoDB-->>CatalogRepo: Current inventory level
    
    alt Inventory change requires quote validation
        OrderService->>OrderService: Check active quotes using this SKU
        OrderService->>MongoDB: Query quotes with this SKU
        MongoDB-->>OrderService: Active quotes list
        OrderService->>OrderService: Calculate impact on quotes
    end
    
    OrderService->>CatalogRepo: Update product
    CatalogRepo-->>MongoDB: Update catalog document
    MongoDB-->>CatalogRepo: Update confirmed
    CatalogRepo-->>OrderService: Product updated
    
    OrderService-->>Frontend: 200 OK with updated product
    Frontend-->>User: Display update confirmation
    
    Note over Frontend,MongoDB: Communication Pattern: Synchronous CRUD with validation
```

## 6. Error Handling and Recovery Pattern

```mermaid
sequenceDiagram
    participant Client as Client Application
    participant OrderService as Order Service (Spring Boot)
    participant RetryHandler as Retry Logic
    participant MongoDB as MongoDB Database
    participant CircuitBreaker as Circuit Breaker
    participant Logger as App Insights
    
    Client->>OrderService: POST /api/orders {orderData}
    
    OrderService->>MongoDB: Insert order document
    MongoDB-->>OrderService: Connection timeout
    
    OrderService->>RetryHandler: Handle transient fault
    RetryHandler->>OrderService: Retry operation (attempt 1)
    OrderService->>MongoDB: Retry insert
    MongoDB-->>OrderService: Network error
    
    RetryHandler->>OrderService: Retry operation (attempt 2)
    OrderService->>MongoDB: Retry insert
    MongoDB-->>OrderService: Success
    
    OrderService-->>Client: 201 Created with order
    
    Note over OrderService,CircuitBreaker: Circuit Breaker Activation
    
    Client->>OrderService: GET /api/orders/{id}
    OrderService->>MongoDB: Query order
    MongoDB-->>OrderService: Database down
    
    OrderService->>CircuitBreaker: Check circuit state
    CircuitBreaker-->>OrderService: Circuit open (after threshold)
    
    OrderService->>Logger: Log error with telemetry
    Logger-->>OrderService: Error logged
    
    OrderService-->>Client: 503 Service Unavailable
    
    Note over Client,Logger: Fallback to Mock Repository (if configured)
    
    alt Fallback enabled
        OrderService->>OrderService: Switch to mock repository
        OrderService-->>Client: 200 OK with cached data
    else No fallback
        OrderService-->>Client: 503 Service Unavailable
    end
    
    Note over Client,Logger: Recovery Pattern: Exponential backoff with circuit breaker
```