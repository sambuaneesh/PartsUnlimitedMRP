
## 1. Quote Creation Workflow

```mermaid
sequenceDiagram
    participant User as Dealer User
    participant UI as Frontend (WinJS)
    participant API as Order Service
    participant DB as MongoDB
    participant Catalog as Catalog Service
    
    User->>UI: Navigate to Quotes Page
    UI->>UI: Load quote creation form
    UI->>API: GET /dealers
    API->>DB: Fetch dealers list
    DB-->>API: Return dealers
    API-->>UI: Dealers data
    UI->>Catalog: GET /catalog
    Catalog->>DB: Fetch product catalog
    DB-->>Catalog: Catalog items
    Catalog-->>UI: Product data
    User->>UI: Fill quote details (customer, dealer, items)
    UI->>Catalog: GET /catalog/{sku} (for each item)
    Catalog->>DB: Fetch item details
    DB-->>Catalog: Item data
    Catalog-->>UI: Item validation
    UI->>UI: Calculate total cost
    User->>UI: Submit quote
    UI->>API: POST /quotes
    API->>API: Validate quote data
    API->>DB: Save quote to quotes collection
    DB-->>API: Quote saved with ID
    API-->>UI: Quote created response
    UI-->>User: Display success with quote ID
```

**Description**: Dealers create sales quotes for customers by selecting products from the catalog, adding specifications, and calculating pricing. The workflow involves frontend form validation, real-time catalog lookups, and quote persistence.

**Communication Patterns**:
- Synchronous REST calls between frontend and backend
- Database transactions for quote persistence
- Real-time catalog validation through synchronous API calls

## 2. Quote to Order Conversion Workflow

```mermaid
sequenceDiagram
    participant User as Dealer User
    participant UI as Frontend
    participant API as Order Service
    participant DB as MongoDB
    participant Queue as Azure Queue
    participant MRP as External MRP
    
    User->>UI: View quote details
    UI->>API: GET /quotes/{quoteId}
    API->>DB: Fetch quote by ID
    DB-->>API: Quote data
    API-->>UI: Quote details
    User->>UI: Click "Create Order"
    UI->>API: POST /orders?fromQuote={id}
    API->>API: Validate quote exists and unused
    API->>DB: Check quote status
    DB-->>API: Quote is valid
    API->>API: Create order object from quote
    API->>DB: Save order to orders collection
    DB-->>API: Order saved with ID
    API->>Queue: Add message to orders queue
    Queue-->>API: Message queued
    Note over Queue: Async: CreateOrderProcessTask will process
    API-->>UI: Order created response
    UI-->>User: Display order confirmation
    
    rect rgb(240, 240, 240)
    Note over Queue,MRP: Async Processing (30s delay)
    Queue->>MRP: CreateOrderProcessTask triggers
    MRP->>MRP: Process order in external system
    MRP->>API: Update order status callback (optional)
    end
```

**Description**: Converting a quote to an order creates a persistent order record and triggers asynchronous processing in the external MRP system through Azure Queue messaging.

**Communication Patterns**:
- Synchronous REST for order creation
- Asynchronous queue-based messaging for MRP integration
- Database transaction ensuring quote-to-order atomicity
- Scheduled task processing with 30-second intervals

## 3. Order Status Progression Workflow

```mermaid
sequenceDiagram
    participant User as System Admin
    participant UI as Frontend
    participant API as Order Service
    participant DB as MongoDB
    participant Event as Event Logger
    
    User->>UI: Navigate to Orders Page
    UI->>API: GET /orders?status=Created
    API->>DB: Query orders by status
    DB-->>API: Orders list
    API-->>UI: Orders data
    User->>UI: Select order and update status
    UI->>API: PUT /orders/{orderId}/status
    API->>API: Validate status transition
    API->>Event: Create status change event
    Event->>DB: Save event to orders collection (embedded)
    DB-->>Event: Event saved
    API->>DB: Update order status
    DB-->>API: Status updated
    API-->>UI: Success response
    UI-->>User: Display updated order
    
    Note right of API: Status Flow: Created -> Confirmed -> Started -> Built -> DeliveryConfirmed -> Shipped -> Delivered -> Installed
```

**Description**: Order status follows a strict 8-state lifecycle. Each status change creates an event log entry for audit trail and tracking purposes.

**Communication Patterns**:
- Synchronous REST calls for status updates
- Embedded document updates in MongoDB
- Event logging within the same transaction
- Client-side filtering for order lists

## 4. Shipment Creation and Tracking Workflow

```mermaid
sequenceDiagram
    participant User as Logistics User
    participant UI as Frontend
    participant API as Order Service
    participant DB as MongoDB
    participant Event as Event Logger
    
    User->>UI: Navigate to Deliveries Page
    UI->>API: GET /orders?status=DeliveryConfirmed
    API->>DB: Query orders ready for shipment
    DB-->>API: Eligible orders
    API-->>UI: Orders list
    User->>UI: Select order and create shipment
    UI->>API: POST /shipments
    API->>API: Validate order status
    API->>API: Create shipment record
    API->>DB: Save shipment to shipments collection
    DB-->>API: Shipment saved with ID
    API->>DB: Update order status to Shipped
    DB-->>API: Order status updated
    API-->>UI: Shipment created response
    
    User->>UI: Add delivery event
    UI->>API: POST /shipments/{id}/events
    API->>Event: Log shipment event
    Event->>DB: Save event to shipments collection
    DB-->>Event: Event saved
    API-->>UI: Event logged response
    
    UI->>API: GET /shipments/deliveries
    API->>DB: Query confirmed deliveries
    DB-->>API: Delivery records
    API-->>UI: Display deliveries
```

**Description**: Shipment workflow creates delivery records for confirmed orders, tracks shipping events, and maintains delivery status throughout the logistics process.

**Communication Patterns**:
- Synchronous REST for CRUD operations
- Cross-collection database operations (orders -> shipments)
- Event logging for tracking
- Status synchronization between related entities

## 5. Catalog Management Workflow

```mermaid
sequenceDiagram
    participant Admin as Catalog Admin
    participant UI as Frontend
    participant API as Catalog Service
    participant DB as MongoDB
    participant Queue as Azure Queue
    
    Admin->>UI: Navigate to Catalog Page
    UI->>API: GET /catalog
    API->>DB: Fetch all catalog items
    DB-->>API: Catalog data
    API-->>UI: Display catalog items
    
    Admin->>UI: Create new product
    UI->>API: POST /catalog
    API->>API: Validate SKU uniqueness
    API->>DB: Save new catalog item
    DB-->>API: Item saved
    API->>Queue: Add inventory update message
    Queue-->>API: Message queued
    
    Admin->>UI: Update product inventory
    UI->>API: PUT /catalog/{sku}
    API->>DB: Update catalog item
    DB-->>API: Item updated
    API->>Queue: Add inventory update message
    Queue-->>API: Message queued
    
    rect rgb(240, 240, 240)
    Note over Queue: Async Processing
    Queue->>Queue: UpdateProductProcessTask (30s)
    Queue->>MRP: Sync inventory changes
    end
    
    API-->>UI: Success response
    UI-->>Admin: Display updated catalog
```

**Description**: Catalog management handles product CRUD operations, inventory updates, and synchronizes changes with external systems through asynchronous queue processing.

**Communication Patterns**:
- Synchronous REST for catalog operations
- Async queue messaging for external synchronization
- Database transactions for data consistency
- Scheduled batch processing for sync operations

## 6. Queue-based Order Processing Workflow

```mermaid
sequenceDiagram
    participant Queue as Azure Queue Service
    participant Task as CreateOrderProcessTask
    participant MRP as External MRP System
    participant API as Order Service
    participant DB as MongoDB
    participant Monitor as AppInsights
    
    Note over Queue: Scheduled every 30 seconds
    Task->>Queue: Poll messages from orders queue
    Queue-->>Task: Return order messages
    
    loop Process each message
        Task->>Task: Parse order message
        Task->>MRP: POST /api/orders (synchronous)
        MRP-->>Task: Order processing result
        
        alt Success
            Task->>API: Update order status to Confirmed
            API->>DB: Update order in database
            DB-->>API: Update complete
            Task->>Queue: Delete processed message
        else Failure
            Task->>Monitor: Log error with telemetry
            Task->>Queue: Leave message for retry
        end
    end
    
    Monitor->>Monitor: Track processing metrics
```

**Description**: Background task processes queued orders by integrating with external MRP system, updating order status based on processing results, and handling failures with retry logic.

**Communication Patterns**:
- Scheduled task execution (Spring @Scheduled)
- Queue message polling and processing
- Synchronous REST calls to external MRP
- Database updates for status synchronization
- Telemetry and monitoring integration
- Failure handling with message retention

## 7. Inventory Synchronization Workflow

```mermaid
sequenceDiagram
    participant Queue as Azure Queue Service
    participant Task as UpdateProductProcessTask
    participant MRP as External MRP System
    participant Catalog as Catalog Service
    participant DB as MongoDB
    participant Retry as MongoOperationsWithRetry
    
    Note over Queue: Scheduled every 30 seconds
    Task->>Queue: Poll messages from inventory queue
    Queue-->>Task: Return inventory update messages
    
    loop Process each message
        Task->>Task: Parse inventory message
        Task->>MRP: GET /api/inventory/{sku}
        MRP-->>Task: Current inventory level
        
        Task->>Catalog: Update inventory via REST
        Catalog->>Retry: Update MongoDB with retry
        Retry->>DB: Update catalog item inventory
        
        alt Success
            DB-->>Retry: Update confirmed
            Retry-->>Catalog: Success
            Catalog-->>Task: Inventory updated
            Task->>Queue: Delete processed message
        else Database Error
            Retry->>Retry: Exponential backoff retry
            Task->>Monitor: Log failure to AppInsights
        end
    end
```

**Description**: Scheduled task synchronizes inventory levels between local catalog and external MRP system, implementing retry logic for database operations and error tracking.

**Communication Patterns**:
- Scheduled task with fixed rate (30s)
- External API integration for inventory sync
- Database operations with retry mechanism
- Telemetry for monitoring failures
- Queue message management

## 8. Error Handling and Recovery Pattern

```mermaid
sequenceDiagram
    participant Client as Frontend/API Client
    participant Filter as CORS/AppInsights Filter
    participant Controller as REST Controller
    participant Service as Business Service
    participant Repo as Repository
    participant DB as MongoDB
    participant Monitor as AppInsights
    
    Client->>Filter: HTTP Request
    Filter->>Filter: Log request metrics
    Filter->>Controller: Forward request
    
    Controller->>Service: Business logic call
    Service->>Repo: Data access call
    
    alt Database Connection Error
        Repo->>Monitor: Log connection failure
        Repo-->>Service: throws RepositoryException
        Service->>Monitor: Log business error
        Service-->>Controller: throws ServiceException
        Controller->>Monitor: Log controller error
        Controller-->>Client: 500 Internal Server Error
    else Business Validation Error
        Service->>Monitor: Log validation error
        Service-->>Controller: throws BadRequestException
        Controller-->>Client: 400 Bad Request
    else Concurrent Modification
        Repo-->>Service: throws ConflictingRequestException
        Service-->>Controller: Propagate exception
        Controller-->>Client: 409 Conflict
    end
    
    Filter->>Filter: Log response metrics
    Filter-->>Client: HTTP Response
```

**Description**: Comprehensive error handling pattern spanning all application layers, with proper exception propagation, logging, monitoring integration, and appropriate HTTP status codes.

**Communication Patterns**:
- Filter chain for cross-cutting concerns
- Exception hierarchy for different error types
- Centralized logging and monitoring
- HTTP status code mapping
- Transaction rollback on database errors