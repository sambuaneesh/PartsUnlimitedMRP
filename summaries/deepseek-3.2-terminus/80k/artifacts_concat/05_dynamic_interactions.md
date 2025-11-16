# Parts Unlimited MRP - Dynamic Interaction Flows

## 1. Order Creation from Website Integration

### Workflow Description
This workflow handles the asynchronous processing of orders received from the Parts Unlimited website via Azure Storage Queues. The IntegrationService polls for new orders and processes them through the MRP system.

**Triggers**: New order message in Azure Queue
**Communication Patterns**: Asynchronous queue processing, REST API calls, database transactions

```mermaid
sequenceDiagram
    participant Website as Parts Unlimited Website
    participant Queue as Azure Storage Queue
    participant Integration as IntegrationService
    participant Order as OrderService
    participant DB as MongoDB

    Note over Website,DB: Order Creation from Website
    
    Website->>Queue: Place order message (async)
    Integration->>Queue: Poll for messages (every 30s)
    Queue->>Integration: Return order message
    
    Integration->>Order: POST /quotes (create quote)
    Order->>DB: Save quote to MongoDB
    DB->>Order: Quote saved successfully
    Order->>Integration: Return quote ID
    
    Integration->>Order: POST /orders?fromQuote={quoteId}
    Order->>DB: Convert quote to order
    DB->>Order: Order created successfully
    Order->>Integration: Return order confirmation
    
    Integration->>Order: POST /shipments (create shipment)
    Order->>DB: Create shipment record
    DB->>Order: Shipment created
    Order->>Integration: Return shipment details
    
    Integration->>Queue: Delete processed message
    Integration->>Website: Update inventory (async)
```

## 2. Quote-to-Order Conversion Workflow

### Workflow Description
This workflow captures the user-driven process of converting a customer quote into a formal order within the MRP system. It involves data validation, order creation, and status initialization.

**Triggers**: User action in web client to convert quote to order
**Communication Patterns**: Synchronous REST API, database transactions, client-server communication

```mermaid
sequenceDiagram
    participant User as MRP User
    participant Client as Web Client
    participant Order as OrderService
    participant DB as MongoDB

    Note over User,DB: Quote to Order Conversion
    
    User->>Client: Select quote for conversion
    Client->>Order: GET /quotes/{id}
    Order->>DB: Retrieve quote details
    DB->>Order: Return quote data
    Order->>Client: Return quote information
    
    User->>Client: Click "Convert to Order"
    Client->>Order: POST /orders?fromQuote={quoteId}
    
    Order->>DB: Validate quote exists and not converted
    DB->>Order: Quote validation successful
    
    Order->>DB: Create new order with quote data
    DB->>Order: Order created with ID
    
    Order->>DB: Update quote status to "converted"
    DB->>Order: Quote updated
    
    Order->>DB: Initialize order status as "Created"
    DB->>Order: Status initialized
    
    Order->>Client: Return order confirmation with ID
    Client->>User: Display order creation success
```

## 3. Order Status Update Workflow

### Workflow Description
This workflow manages the progression of an order through its lifecycle statuses, from creation to installation. Each status change triggers events and may involve multiple system components.

**Triggers**: User status updates, automated system events
**Communication Patterns**: REST API, database transactions, event logging

```mermaid
sequenceDiagram
    participant User as MRP User
    participant Client as Web Client
    participant Order as OrderService
    participant DB as MongoDB
    participant Shipment as ShipmentService

    Note over User,Shipment: Order Status Update Flow
    
    User->>Client: Navigate to order details
    Client->>Order: GET /orders/{id}
    Order->>DB: Retrieve order data
    DB->>Order: Return order information
    Order->>Client: Return order details
    
    User->>Client: Update order status (e.g., "Confirmed")
    Client->>Order: PUT /orders/{id}/status
    Order->>DB: Validate status transition
    DB->>Order: Status validation passed
    
    Order->>DB: Update order status
    DB->>Order: Status updated
    
    Order->>DB: Add status change event
    DB->>Order: Event recorded
    
    alt Status requires shipment creation
        Order->>Shipment: POST /shipments
        Shipment->>DB: Create shipment record
        DB->>Shipment: Shipment created
        Shipment->>Order: Return shipment ID
    end
    
    Order->>Client: Return update confirmation
    Client->>User: Display updated status and timeline
```

## 4. Inventory Management and Catalog Update

### Workflow Description
This workflow handles inventory updates triggered by order processing and manual catalog management. It ensures inventory levels are maintained and propagated to external systems.

**Triggers**: Order processing, manual catalog updates, scheduled tasks
**Communication Patterns**: REST API, database transactions, asynchronous messaging

```mermaid
sequenceDiagram
    participant User as MRP User
    participant Client as Web Client
    participant Order as OrderService
    participant DB as MongoDB
    participant Integration as IntegrationService
    participant Website as Parts Unlimited Website

    Note over User,Website: Inventory Management Flow
    
    User->>Client: Update catalog item inventory
    Client->>Order: PUT /catalog/{sku}
    Order->>DB: Update inventory level
    DB->>Order: Inventory updated
    
    Order->>Integration: Notify inventory change (async)
    Integration->>Website: Update website inventory
    
    alt Order processing affects inventory
        Order->>DB: Process order items
        DB->>Order: Calculate inventory impact
        
        loop For each ordered item
            Order->>DB: Decrement inventory count
            DB->>Order: Inventory updated
            Order->>DB: Check lead time requirements
            DB->>Order: Lead time calculated
        end
        
        Order->>Integration: Notify inventory changes
        Integration->>Website: Propagate inventory updates
    end
    
    Order->>Client: Return operation result
    Client->>User: Display updated inventory status
```

## 5. Shipment Tracking and Delivery Management

### Workflow Description
This workflow manages the complete shipment lifecycle from creation through delivery confirmation. It includes event tracking, status updates, and coordination with order management.

**Triggers**: Order status changes, manual shipment creation, delivery events
**Communication Patterns**: REST API, database transactions, event-driven updates

```mermaid
sequenceDiagram
    participant User as MRP User
    participant Client as Web Client
    participant Order as OrderService
    participant DB as MongoDB
    participant Shipment as ShipmentController

    Note over User,Shipment: Shipment Tracking Flow
    
    User->>Client: Create shipment from order
    Client->>Shipment: POST /shipments
    Shipment->>DB: Create shipment with order data
    DB->>Shipment: Shipment created
    
    Shipment->>Order: Update order status to "Shipped"
    Order->>DB: Update order status
    DB->>Order: Status updated
    
    Shipment->>Client: Return shipment confirmation
    
    loop During shipment progress
        User->>Client: Add shipment event
        Client->>Shipment: POST /shipments/{id}/events
        Shipment->>DB: Add shipment event
        DB->>Shipment: Event recorded
        
        Shipment->>Order: Notify shipment progress
        Order->>DB: Update order timeline
        DB->>Order: Timeline updated
    end
    
    User->>Client: Mark delivery as complete
    Client->>Shipment: PUT /shipments/{id} (status=Delivered)
    Shipment->>DB: Update shipment status
    DB->>Shipment: Status updated
    
    Shipment->>Order: Update order status to "Delivered"
    Order->>DB: Finalize order status
    DB->>Order: Order completed
    
    Shipment->>Client: Return delivery confirmation
    Client->>User: Display delivery completion
```

## 6. Error Handling and Recovery Patterns

### Workflow Description
This workflow demonstrates the system's error handling and recovery mechanisms across different failure scenarios, including database failures, service unavailability, and data validation errors.

**Triggers**: System failures, validation errors, timeout scenarios
**Communication Patterns**: Retry patterns, fallback mechanisms, error propagation

```mermaid
sequenceDiagram
    participant Client as Web Client
    participant Order as OrderService
    participant DB as MongoDB
    participant Integration as IntegrationService
    participant Queue as Azure Queue

    Note over Client,Queue: Error Handling and Recovery
    
    Client->>Order: POST /orders (create order)
    
    alt Database connection failure
        Order->>DB: Save order data
        DB-->>Order: Connection timeout
        Order->>Order: Retry logic (3 attempts)
        Order->>DB: Retry database operation
        DB-->>Order: Still unavailable
        
        Order->>Client: Return 503 Service Unavailable
        Client->>Client: Show retry prompt to user
    end
    
    alt Data validation error
        Order->>Order: Validate order data
        Order-->>Order: Validation failed (missing fields)
        Order->>Client: Return 400 Bad Request with error details
        Client->>Client: Display validation errors to user
    end
    
    alt Integration service failure
        Integration->>Queue: Poll for messages
        Queue->>Integration: Return order message
        Integration->>Order: POST /quotes
        Order-->>Integration: Service unavailable
        
        Integration->>Integration: Exponential backoff retry
        Integration->>Queue: Re-queue message for later processing
        Integration->>Integration: Log failure for monitoring
    end
    
    alt Partial failure in workflow
        Order->>DB: Begin transaction
        Order->>DB: Save order
        DB->>Order: Order saved
        Order->>DB: Update inventory
        DB-->>Order: Inventory update failed
        
        Order->>DB: Rollback transaction
        DB->>Order: Transaction rolled back
        Order->>Client: Return 500 Internal Server Error
        Client->>Client: Show error and suggest retry
    end
```

## Communication Patterns Summary

### Synchronous Communication
- **REST API Calls**: Web Client ↔ OrderService, IntegrationService ↔ OrderService
- **Database Operations**: All services ↔ MongoDB
- **Health Checks**: Ping endpoints for service monitoring

### Asynchronous Communication
- **Queue Processing**: Website → Azure Queue → IntegrationService
- **Event Propagation**: Order status changes → Shipment updates
- **Background Tasks**: Scheduled inventory synchronization

### Data Flow Patterns
- **Request-Response**: Client-initiated operations with immediate feedback
- **Event Sourcing**: Order and shipment event timelines
- **Batch Processing**: Scheduled queue polling and inventory updates

### Error Handling Patterns
- **Retry Logic**: Database operations and external service calls
- **Circuit Breaker**: Service-to-service communication
- **Compensation**: Transaction rollback for partial failures
- **Dead Letter Queues**: Failed message handling in async workflows