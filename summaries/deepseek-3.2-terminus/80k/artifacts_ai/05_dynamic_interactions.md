```markdown
# Parts Unlimited MRP - Dynamic Interaction Flows

## 1. Quote Creation and Management Workflow

### Workflow Description
**Purpose**: Create and manage customer quotes for refrigeration/freezer units with pricing calculations and validation.
**Triggers**: User initiates quote creation from web client, enters customer and product details.
**Communication Patterns**: Synchronous REST calls, Google Maps API integration, database transactions.

```mermaid
sequenceDiagram
    participant User as Web Client User
    participant UI as Quote Management UI
    participant Data as Data Service Layer
    participant OrderService as OrderService API
    participant Google as Google Maps API
    participant DB as MongoDB

    User->>UI: Navigate to Quote Management
    UI->>Data: quotesGet()
    Data->>OrderService: GET /quotes
    OrderService->>DB: Query quotes collection
    DB-->>OrderService: Return quotes list
    OrderService-->>Data: 200 OK with quotes
    Data-->>UI: Render quotes list

    User->>UI: Click "Create New Quote"
    UI->>Data: quoteCreate()
    Data-->>UI: Return empty quote template
    UI->>User: Display quote form

    User->>UI: Enter customer details & address
    UI->>Google: Address autocomplete request
    Google-->>UI: Return validated address
    UI->>User: Display address suggestions

    User->>UI: Select product type (Refrigerator/Freezer)
    UI->>Data: catalogGet()
    Data->>OrderService: GET /catalog
    OrderService->>DB: Query catalog collection
    DB-->>OrderService: Return catalog items
    OrderService-->>Data: 200 OK with catalog
    Data-->>UI: Render product options

    User->>UI: Configure unit specs & add extras
    UI->>UI: Calculate total cost with discount
    User->>UI: Submit quote

    UI->>Data: quoteSave(quote)
    Data->>OrderService: POST /quotes
    OrderService->>OrderService: Validate quote data
    OrderService->>DB: Insert quote document
    DB-->>OrderService: Quote created successfully
    OrderService-->>Data: 201 Created with quoteId
    Data-->>UI: Confirmation with quote ID
    UI->>User: Display success message
```

## 2. Order Creation from Quote Workflow

### Workflow Description
**Purpose**: Convert an approved quote into a formal order with status tracking and inventory validation.
**Triggers**: User selects quote and initiates order creation, system validates inventory availability.
**Communication Patterns**: Synchronous REST calls, database transactions with validation, inventory checks.

```mermaid
sequenceDiagram
    participant User as Web Client User
    participant UI as Order Management UI
    participant Data as Data Service Layer
    participant OrderService as OrderService API
    participant DB as MongoDB

    User->>UI: Navigate to Quotes list
    UI->>Data: quotesGet()
    Data->>OrderService: GET /quotes
    OrderService->>DB: Query quotes collection
    DB-->>OrderService: Return approved quotes
    OrderService-->>Data: 200 OK with quotes
    Data-->>UI: Render quotes with "Create Order" buttons

    User->>UI: Select quote & click "Create Order"
    UI->>Data: orderCreateFromQuote(quoteId)
    Data->>OrderService: POST /orders?fromQuote={quoteId}
    
    OrderService->>DB: Find quote by ID
    DB-->>OrderService: Return quote details
    OrderService->>OrderService: Validate quote exists & not converted
    OrderService->>DB: Check for existing order with quoteId
    DB-->>OrderService: No existing order found
    
    OrderService->>DB: Query catalog for inventory check
    DB-->>OrderService: Return inventory levels
    OrderService->>OrderService: Calculate lead time based on inventory
    
    OrderService->>OrderService: Generate orderId: order-{quoteId}
    OrderService->>DB: Insert order document with status="Created"
    DB-->>OrderService: Order created successfully
    
    OrderService->>DB: Create initial order event
    DB-->>OrderService: Event recorded
    OrderService-->>Data: 201 Created with order details
    Data-->>UI: Order creation confirmed
    UI->>User: Display order details with status "Created"
```

## 3. External Order Processing via Integration Service

### Workflow Description
**Purpose**: Process orders from Parts Unlimited website through Azure queues with background processing.
**Triggers**: Website places order in Azure queue, IntegrationService polls queue every 30 seconds.
**Communication Patterns**: Asynchronous queue processing, scheduled tasks, REST API calls between services.

```mermaid
sequenceDiagram
    participant Website as Parts Unlimited Website
    participant AzureQueue as Azure Storage Queue
    participant Integration as IntegrationService
    participant OrderService as OrderService API
    participant DB as MongoDB

    Website->>AzureQueue: Place order message in "orders" queue
    Note over Integration,AzureQueue: Every 30 seconds...
    
    Integration->>AzureQueue: Poll "orders" queue for messages
    AzureQueue-->>Integration: Return order message if available
    
    alt Message Available
        Integration->>Integration: Parse order message
        Integration->>OrderService: POST /quotes (create quote from website order)
        OrderService->>DB: Insert quote document
        DB-->>OrderService: Quote created
        OrderService-->>Integration: 201 Created with quoteId
        
        Integration->>OrderService: POST /orders?fromQuote={quoteId}
        OrderService->>DB: Create order from quote
        DB-->>OrderService: Order created
        OrderService-->>Integration: 201 Created with orderId
        
        Integration->>AzureQueue: Delete processed message
        AzureQueue-->>Integration: Message deleted
        
        Integration->>Integration: Schedule inventory update
        Integration->>OrderService: GET /catalog
        OrderService->>DB: Query catalog collection
        DB-->>OrderService: Return current inventory
        OrderService-->>Integration: 200 OK with catalog
        Integration->>AzureQueue: Place inventory update in "product" queue
    else No Message
        Integration->>Integration: Wait for next polling interval
    end
```

## 4. Order to Delivery Conversion Workflow

### Workflow Description
**Purpose**: Convert confirmed orders into scheduled deliveries with contact coordination and address management.
**Triggers**: Order reaches "Confirmed" status, user initiates delivery creation for order fulfillment.
**Communication Patterns**: Synchronous REST calls, embedded data relationships, status workflow enforcement.

```mermaid
sequenceDiagram
    participant User as Web Client User
    participant UI as Delivery Management UI
    participant Data as Data Service Layer
    participant OrderService as OrderService API
    participant DB as MongoDB

    User->>UI: Navigate to Orders with "Confirmed" status
    UI->>Data: ordersGet() with status filter
    Data->>OrderService: GET /orders?status=Confirmed
    OrderService->>DB: Query orders collection
    DB-->>OrderService: Return confirmed orders
    OrderService-->>Data: 200 OK with orders
    Data-->>UI: Render orders with "Create Delivery" buttons

    User->>UI: Select order & click "Create Delivery"
    UI->>Data: deliveryCreateFromOrder(orderId)
    Data->>OrderService: POST /shipments
    
    OrderService->>DB: Find order by ID with embedded quote
    DB-->>OrderService: Return order with full details
    OrderService->>OrderService: Validate order status allows delivery creation
    
    OrderService->>OrderService: Populate delivery from order & quote data
    OrderService->>DB: Insert shipment document with embedded order/quote
    DB-->>OrderService: Delivery created successfully
    
    OrderService->>DB: Update order status to "DeliveryConfirmed"
    DB-->>OrderService: Order status updated
    
    OrderService->>DB: Create delivery event record
    DB-->>OrderService: Event recorded
    OrderService-->>Data: 201 Created with delivery details
    Data-->>UI: Delivery creation confirmed
    UI->>User: Display delivery scheduling interface
```

## 5. Order Status Update and Event Tracking Workflow

### Workflow Description
**Purpose**: Update order status through workflow progression and maintain audit trail of all order events.
**Triggers**: User actions (status updates), system events (automatic transitions), delivery milestones.
**Communication Patterns**: Synchronous REST calls, event logging, status validation, workflow enforcement.

```mermaid
sequenceDiagram
    participant User as Web Client User
    participant UI as Order Management UI
    participant Data as Data Service Layer
    participant OrderService as OrderService API
    participant DB as MongoDB

    User->>UI: Navigate to specific order details
    UI->>Data: orderFindById(orderId)
    Data->>OrderService: GET /orders/{orderId}
    OrderService->>DB: Query orders collection with events
    DB-->>OrderService: Return order with events timeline
    OrderService-->>Data: 200 OK with order details
    Data-->>UI: Render order with current status and events

    User->>UI: Select new status from workflow options
    UI->>UI: Validate status transition (Created→Confirmed→Started→Built→etc.)
    
    User->>UI: Add status change comments and submit
    UI->>Data: orderSave(order, original)
    Data->>OrderService: PUT /orders/{orderId}/status
    
    OrderService->>DB: Find current order status
    DB-->>OrderService: Return current status
    OrderService->>OrderService: Validate status transition rules
    
    OrderService->>DB: Update order status field
    DB-->>OrderService: Status updated successfully
    
    OrderService->>OrderService: Create OrderEvent with timestamp and comments
    OrderService->>DB: Insert event into order's events array
    DB-->>OrderService: Event recorded
    
    OrderService-->>Data: 200 OK with updated order
    Data-->>UI: Status update confirmed
    UI->>User: Display updated status and new event in timeline
    
    alt Status requires inventory update
        OrderService->>DB: Query catalog for inventory adjustment
        DB-->>OrderService: Return current inventory
        OrderService->>DB: Update inventory levels
        DB-->>OrderService: Inventory updated
    end
```

## 6. Catalog Inventory Management Workflow

### Workflow Description
**Purpose**: Manage product catalog including SKU maintenance, pricing updates, and inventory tracking.
**Triggers**: Product changes, inventory adjustments, price updates, new product introductions.
**Communication Patterns**: Synchronous REST CRUD operations, inventory validation, lead time calculation.

```mermaid
sequenceDiagram
    participant User as Web Client User
    participant UI as Catalog Management UI
    participant Data as Data Service Layer
    participant OrderService as OrderService API
    participant DB as MongoDB
    participant Integration as IntegrationService

    User->>UI: Navigate to Catalog Management
    UI->>Data: catalogGet()
    Data->>OrderService: GET /catalog
    OrderService->>DB: Query catalog collection
    DB-->>OrderService: Return all catalog items
    OrderService-->>Data: 200 OK with catalog
    Data-->>UI: Render catalog with inventory levels

    User->>UI: Click "Add New Product"
    UI->>Data: catalogCreate()
    Data-->>UI: Return empty catalog template
    UI->>User: Display product form

    User->>UI: Enter SKU, description, price, inventory, lead time
    User->>UI: Submit new product
    UI->>Data: catalogSave(catalogItem)
    Data->>OrderService: POST /catalog
    
    OrderService->>OrderService: Validate catalog item data
    OrderService->>DB: Check for duplicate SKU
    DB-->>OrderService: No duplicate found
    OrderService->>DB: Insert catalog document
    DB-->>OrderService: Product created successfully
    OrderService-->>Data: 201 Created
    Data-->>UI: Product creation confirmed
    
    Integration->>Integration: Scheduled inventory sync (every 30s)
    Integration->>OrderService: GET /catalog
    OrderService->>DB: Query catalog collection
    DB-->>OrderService: Return updated catalog
    OrderService-->>Integration: 200 OK with catalog
    Integration->>AzureQueue: Place inventory updates in "product" queue
```

## 7. Error Handling and Recovery Patterns

### Workflow Description
**Purpose**: Handle system failures, validation errors, and recovery scenarios across all workflows.
**Triggers**: Network failures, validation errors, database constraints, external service unavailability.
**Communication Patterns**: Structured error responses, retry mechanisms, graceful degradation.

```mermaid
sequenceDiagram
    participant User as Web Client User
    participant UI as Web Client
    participant Data as Data Service Layer
    participant OrderService as OrderService API
    participant DB as MongoDB
    participant Ext as External Services

    User->>UI: Perform action (e.g., save order)
    UI->>Data: orderSave(order, original)
    Data->>OrderService: PUT /orders/{orderId}
    
    alt Database Connection Failure
        OrderService->>DB: Update order document
        DB-->>OrderService: Connection timeout
        OrderService->>OrderService: Retry logic (3 attempts)
        OrderService->>DB: Retry update operation
        DB-->>OrderService: Still unavailable
        OrderService-->>Data: 503 Service Unavailable
        Data-->>UI: Display "System temporarily unavailable"
        UI->>User: Show retry option with error message
    end
    
    alt Validation Error
        OrderService->>OrderService: Validate order data
        OrderService->>OrderService: Accumulate validation errors
        OrderService-->>Data: 400 Bad Request with error details
        Data-->>UI: Return structured error object
        UI->>UI: Highlight invalid fields with error messages
        UI->>User: Display validation errors with guidance
    end
    
    alt External Service Failure (Google Maps)
        UI->>Ext: Address validation request
        Ext-->>UI: Service timeout
        UI->>UI: Fallback to manual address entry
        UI->>User: Display "Address service unavailable" warning
        User->>UI: Continue with manual address input
    end
    
    alt Queue Processing Failure
        Integration->>AzureQueue: Poll for messages
        AzureQueue-->>Integration: Service unavailable
        Integration->>Integration: Exponential backoff retry
        Integration->>Integration: Log failure for monitoring
        Integration->>Integration: Continue other operations
    end
```

## Communication Patterns Summary

### Synchronous Patterns
- **REST API Calls**: All client-server communication via HTTP REST endpoints
- **Database Transactions**: MongoDB operations with immediate consistency requirements
- **External API Integration**: Google Maps Places API for address validation

### Asynchronous Patterns
- **Queue Processing**: Azure Storage Queues for website order integration
- **Scheduled Tasks**: Background processing at 30-second intervals
- **Event Logging**: Non-blocking event recording for audit trails

### Data Flow Patterns
- **Request-Response**: Standard RESTful communication between components
- **Embedded Data**: Quotes embedded in orders, orders embedded in deliveries
- **Workflow Chaining**: Sequential quote→order→delivery conversion process
- **Status Propagation**: Order status changes triggering downstream actions
```