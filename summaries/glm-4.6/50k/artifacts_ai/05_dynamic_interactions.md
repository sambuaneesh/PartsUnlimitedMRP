
# Workflow 1: Quote-to-Order Conversion

## Purpose
Converts a validated quote into an order, checking inventory and preventing duplicate orders. Triggered when user clicks "Create Order" from the Quotes page.

```mermaid
sequenceDiagram
    participant Frontend as WinJS Frontend
    participant OrderService as OrderService (Spring Boot)
    participant QuoteRepo as Quote Repository
    participant OrderRepo as Order Repository
    participant CatalogRepo as Catalog Repository
    participant MongoDB as MongoDB
    participant Integration as IntegrationService

    Frontend->>OrderService: POST /api/orders (with quoteId)
    OrderService->>QuoteRepo: findByQuoteId(quoteId)
    QuoteRepo->>MongoDB: Find quote document
    MongoDB-->>QuoteRepo: Quote data
    QuoteRepo-->>OrderService: Quote object

    OrderService->>OrderRepo: findByQuoteId(quoteId)
    OrderRepo->>MongoDB: Check existing order
    MongoDB-->>OrderRepo: Order/null
    OrderRepo-->>OrderService: null (no duplicate)

    loop For each quote item
        OrderService->>CatalogRepo: findBySkuNumber(item.sku)
        CatalogRepo->>MongoDB: Find catalog item
        MongoDB-->>CatalogRepo: CatalogItem
        CatalogRepo-->>OrderService: CatalogItem
        OrderService->>OrderService: Check inventory >= item.amount
    end

    OrderService->>OrderService: Calculate totals, set status "Created"
    OrderService->>OrderRepo: save(newOrder)
    OrderRepo->>MongoDB: Insert order document
    MongoDB-->>OrderRepo: Success confirmation
    OrderRepo-->>OrderService: Order with ID

    OrderService->>Integration: notifyOrderCreated(order)
    OrderService-->>Frontend: 201 Created with order data

    Frontend->>Frontend: Navigate to Orders page
    Frontend->>Frontend: Refresh order list
```

## Communication Patterns
- Synchronous REST API calls throughout
- Database transactions for data consistency
- Repository pattern abstraction
- No async messaging in this flow

---

# Workflow 2: Order-to-Delivery Flow

## Purpose
Creates a delivery/shipment record when an order reaches "Completed" status. Triggered automatically or manually when order is ready for delivery.

```mermaid
sequenceDiagram
    participant Frontend as WinJS Frontend
    participant OrderService as OrderService
    participant OrderRepo as Order Repository
    participant ShipmentRepo as Shipment Repository
    participant MongoDB as MongoDB

    Frontend->>OrderService: POST /api/shipments (with orderId)
    OrderService->>OrderRepo: findByOrderId(orderId)
    OrderRepo->>MongoDB: Find order document
    MongoDB-->>OrderRepo: Order data
    OrderRepo-->>OrderService: Order object

    OrderService->>OrderService: Validate order status = "Completed"
    
    OrderService->>ShipmentRepo: findByOrderId(orderId)
    ShipmentRepo->>MongoDB: Check existing shipment
    MongoDB-->>ShipmentRepo: Shipment/null
    ShipmentRepo-->>OrderService: null (no duplicate)

    OrderService->>OrderService: Create ShipmentRecord from order
    Note over OrderService: Map customer info, delivery address
    
    OrderService->>ShipmentRepo: save(shipmentRecord)
    ShipmentRepo->>MongoDB: Insert shipment document
    MongoDB-->>ShipmentRepo: Success with shipmentId
    ShipmentRepo-->>OrderService: ShipmentRecord with ID

    OrderService->>OrderRepo: updateStatus(orderId, "DeliveryConfirmed")
    OrderRepo->>MongoDB: Update order status
    MongoDB-->>OrderRepo: Success
    OrderRepo-->>OrderService: Updated order

    OrderService-->>Frontend: 201 Created with shipment data
    Frontend->>Frontend: Navigate to Deliveries page
```

## Communication Patterns
- Synchronous REST operations
- Database transaction spans multiple collections
- Status update follows state machine pattern
- Atomic operations ensure consistency

---

# Workflow 3: Order Status Progression with Events

## Purpose
Tracks order lifecycle through 8 states with event logging. Events are added at each status change milestone.

```mermaid
sequenceDiagram
    participant Frontend as WinJS Frontend
    participant OrderService as OrderService
    participant OrderRepo as Order Repository
    participant MongoDB as MongoDB

    Frontend->>OrderService: POST /api/orders/{id}/events
    Note over Frontend: User updates order status or system auto-updates

    OrderService->>OrderRepo: findByOrderId(orderId)
    OrderRepo->>MongoDB: Find current order
    MongoDB-->>OrderRepo: Order with current status
    OrderRepo-->>OrderService: Order object

    OrderService->>OrderService: Validate event comment not empty
    OrderService->>OrderService: Validate status transition is valid
    
    Note over OrderService: Status Flow:<br/>Created → Processing → Shipped<br/>→ Delivered → Installed → Completed
    OrderService->>OrderService: Create OrderEventInfo with timestamp

    OrderService->>OrderRepo: addEvent(orderId, event)
    OrderRepo->>MongoDB: $push event to events array
    MongoDB-->>OrderRepo: Success
    
    OrderService->>OrderRepo: updateStatus(orderId, newStatus)
    OrderRepo->>MongoDB: $set status field
    MongoDB-->>OrderRepo: Success
    OrderRepo-->>OrderService: Updated order

    OrderService-->>Frontend: 200 OK with updated order
    Frontend->>Frontend: Update UI with new status and event
    Frontend->>Frontend: Show event in timeline
```

## Communication Patterns
- Event sourcing pattern for order history
- MongoDB array operations for event storage
- State machine validation
- Real-time UI updates after status change

---

# Workflow 4: Address Standardization Flow

## Purpose
Standardizes dealer addresses using Google Maps Places API when creating or updating dealer information. Triggered during dealer CRUD operations.

```mermaid
sequenceDiagram
    participant Frontend as WinJS Frontend
    participant GoogleAPI as Google Maps API
    participant OrderService as OrderService
    participant DealerRepo as Dealer Repository
    participant MongoDB as MongoDB

    Frontend->>Frontend: User enters address in dealer form
    Frontend->>GoogleAPI: Places autocomplete query
    GoogleAPI-->>Frontend: Suggested places list
    
    Frontend->>Frontend: User selects place
    Frontend->>Frontend: getPostCodeFromPlace(place)
    Frontend->>GoogleAPI: Place details request
    GoogleAPI-->>Frontend: Formatted address + postal code

    Frontend->>OrderService: POST /api/dealers (with standardized address)
    OrderService->>DealerRepo: save(dealer)
    DealerRepo->>MongoDB: Insert/update dealer document
    MongoDB-->>DealerRepo: Success with dealerId
    DealerRepo-->>OrderService: Dealer object

    OrderService-->>Frontend: 201 Created with dealer data
    
    Note over Frontend,GoogleAPI: Address validation occurs on edit too
    Frontend->>GoogleAPI: Validate existing address
    GoogleAPI-->>Frontend: Validation response
```

## Communication Patterns
- Third-party API integration (Google Maps)
- Async autocomplete API calls
- Client-side address processing
- Standardized data storage

---

# Workflow 5: Catalog Synchronization Flow

## Purpose
Periodic synchronization of catalog data between MRP system and local database via Azure queues. Runs every 30 minutes.

```mermaid
sequenceDiagram
    participant MRP as Parts Unlimited MRP
    participant Queue as Azure Storage Queue
    participant IntegrationService as IntegrationService
    participant OrderService as OrderService
    participant CatalogRepo as Catalog Repository
    participant MongoDB as MongoDB

    Note over MRP,IntegrationService: Runs every 30 minutes (UpdateProductProcessTask)
    
    IntegrationService->>Queue: Receive product update messages
    Queue-->>IntegrationService: Product update data
    
    loop For each product update
        IntegrationService->>OrderService: POST /api/catalog (product data)
        OrderService->>CatalogRepo: findBySkuNumber(product.sku)
        CatalogRepo->>MongoDB: Find existing catalog item
        MongoDB-->>CatalogRepo: CatalogItem/null
        CatalogRepo-->>OrderService: Existing item or null

        alt Product exists
            OrderService->>OrderService: Update inventory/price
            OrderService->>CatalogRepo: save(updatedItem)
        else New product
            OrderService->>CatalogRepo: save(newItem)
        end
        
        CatalogRepo->>MongoDB: Insert/update catalog document
        MongoDB-->>CatalogRepo: Success
        CatalogRepo-->>OrderService: Updated catalog
        OrderService-->>IntegrationService: 200 OK
    end

    IntegrationService->>Queue: Delete processed messages
```

## Communication Patterns
- Scheduled task execution
- Queue-based messaging (polling)
- Batch processing
- Idempotent update operations

---

# Workflow 6: Azure Queue Order Processing

## Purpose
Processes orders from Azure Storage Queue, forwarding to external MRP system. Scheduled task runs every 30 seconds.

```mermaid
sequenceDiagram
    participant Frontend as WinJS Frontend
    participant OrderService as OrderService
    participant Queue as Azure Storage Queue
    participant IntegrationService as IntegrationService
    participant MRP as Parts Unlimited MRP

    Frontend->>OrderService: POST /api/orders (create order)
    OrderService->>OrderService: Save order locally
    OrderService->>Queue: Add order to queue (async)

    Note over Queue,IntegrationService: Runs every 30 seconds (CreateOrderProcessTask)
    
    IntegrationService->>Queue: Get order messages
    Queue-->>IntegrationService: Order data
    
    IntegrationService->>MRP: POST /external/orders (MrpConnectService)
    Note over IntegrationService: Transform order format for external system
    MRP-->>IntegrationService: 200 OK with order confirmation
    
    IntegrationService->>OrderService: POST /api/orders/{id}/events
    Note over IntegrationService: Add "Sent to MRP" event
    OrderService->>OrderService: Log event, update status
    
    IntegrationService->>Queue: Delete processed message
    
    alt Error from MRP
        MRP-->>IntegrationService: 4xx/5xx error
        IntegrationService->>OrderService: Add error event
        IntegrationService->>Queue: Return message to queue (retry)
    end
```

## Communication Patterns
- Asynchronous queue processing
- External system integration
- Retry mechanism on failure
- Event logging for audit trail

---

# Workflow 7: Frontend CRUD Operations (Generic)

## Purpose
Standard CRUD operations performed by the WinJS frontend for all entities (dealers, quotes, orders, etc.). Shows data binding and REST interaction pattern.

```mermaid
sequenceDiagram
    participant UI as WinJS UI Page
    participant ViewModel as Page ViewModel
    participant DataService as Data Service
    participant OrderService as OrderService (REST)
    participant Repository as Repository Layer
    participant MongoDB as MongoDB

    Note over UI,MongoDB: READ Operation (List View)
    UI->>ViewModel: onready()
    ViewModel->>DataService: getItems()
    DataService->>OrderService: GET /api/resource
    OrderService->>Repository: findAll()
    Repository->>MongoDB: Find all documents
    MongoDB-->>Repository: Resource array
    Repository-->>OrderService: Resource objects
    OrderService-->>DataService: JSON array
    DataService-->>ViewModel: WinJS.Binding.List
    ViewModel->>UI: Update list view

    Note over UI,MongoDB: CREATE Operation
    UI->>ViewModel: saveItem(formData)
    ViewModel->>DataService: saveItem(item)
    DataService->>OrderService: POST /api/resource
    OrderService->>Repository: save(item)
    Repository->>MongoDB: Insert document
    MongoDB-->>Repository: ID
    Repository-->>OrderService: Created object
    OrderService-->>DataService: 201 Created
    DataService-->>ViewModel: Success
    ViewModel->>UI: Navigate to detail/list

    Note over UI,MongoDB: UPDATE Operation
    UI->>ViewModel: saveItem(changedItem)
    ViewModel->>DataService: updateItem(item)
    DataService->>OrderService: PUT/PATCH /api/resource/{id}
    OrderService->>Repository: save(item)
    Repository->>MongoDB: Update document
    MongoDB-->>Repository: Success
    Repository-->>OrderService: Updated object
    OrderService-->>DataService: 200 OK
    DataService-->>ViewModel: Success
    ViewModel->>UI: Update view

    Note over UI,MongoDB: DELETE Operation
    UI->>ViewModel: deleteItem(itemId)
    ViewModel->>DataService: deleteItem(id)
    DataService->>OrderService: DELETE /api/resource/{id}
    OrderService->>Repository: delete(id)
    Repository->>MongoDB: Delete document
    MongoDB-->>Repository: Success
    Repository-->>OrderService: 204 No Content
    OrderService-->>DataService: Success
    DataService-->>ViewModel: Success
    ViewModel->>UI: Remove item from list
```

## Communication Patterns
- MVVM pattern with data binding
- Observable collections (WinJS.Binding.List)
- Two-way data binding for forms
- RESTful resource operations
- Optimistic UI updates

---

# Workflow 8: Error Handling and Recovery Flow

## Purpose
Global error handling across frontend and backend with user notification, logging, and recovery mechanisms.

```mermaid
sequenceDiagram
    participant Frontend as WinJS Frontend
    participant ErrorHandler as Global Error Handler
    participant OrderService as OrderService
    participant ExceptionFilter as Exception Filter
    participant AppInsights as Application Insights
    participant UI as UI Components

    Frontend->>OrderService: API Request
    OrderService->>OrderService: Business logic execution
    
    alt Business Rule Violation
        OrderService-->>ExceptionFilter: throw BadRequestException
        ExceptionFilter->>ExceptionFilter: Log error details
        ExceptionFilter->>AppInsights: TrackException
        ExceptionFilter-->>Frontend: 400 Bad Request + error message
        Frontend->>ErrorHandler: reporterror(message)
        ErrorHandler->>UI: show error dialog
        UI-->>Frontend: User acknowledges
    end

    alt Conflict/Duplicate
        OrderService-->>ExceptionFilter: throw ConflictingRequestException
        ExceptionFilter->>AppInsights: TrackException
        ExceptionFilter-->>Frontend: 409 Conflict + details
        Frontend->>ErrorHandler: reporterror("Duplicate order exists")
        ErrorHandler->>UI: show confirmation dialog
        UI-->>Frontend: User choice (OK/Cancel)
    end

    alt Network/Infrastructure Error
        OrderService-xExceptionFilter: throw RuntimeException
        ExceptionFilter->>AppInsights: TrackException
        ExceptionFilter-->>Frontend: 500 Internal Server Error
        Frontend->>ErrorHandler: reporterror("System error occurred")
        ErrorHandler->>UI: show error with retry option
        alt User chooses Retry
            UI-->>Frontend: Retry request
            Frontend->>OrderService: Retry API Request
        else User cancels
            UI-->>Frontend: Navigate to safe page
        end
    end

    Note over Frontend,AppInsights: Frontend Global Handler
    Frontend->>ErrorHandler: onerror(exception)
    ErrorHandler->>AppInsights: TrackException
    ErrorHandler->>UI: showProgress(false)
    ErrorHandler->>UI: popup("An error occurred")
```

## Communication Patterns
- Global exception filters in Spring
- Centralized error logging
- User-friendly error messages
- Retry mechanisms for transient failures
- Telemetry integration for monitoring