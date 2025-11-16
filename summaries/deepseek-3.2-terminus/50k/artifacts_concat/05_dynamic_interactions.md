```markdown
# Parts Unlimited MRP - Dynamic Interaction Flows

## Workflow 1: Quote-to-Order Conversion

### Description
This workflow handles the conversion of a customer quote into a formal order, including validation, inventory checks, and status transitions.

### Communication Patterns
- **Synchronous REST**: Frontend to OrderService communication
- **Database Transactions**: Order creation and inventory updates
- **Event-driven**: Order status updates and event logging

```mermaid
sequenceDiagram
    participant User as Customer/Dealer
    participant Frontend as Web Client
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Integration as Integration Service
    participant Queue as Azure Queue

    User->>Frontend: Create Quote
    Frontend->>OrderService: POST /quotes
    OrderService->>MongoDB: Validate dealer & catalog items
    OrderService->>MongoDB: Save quote (quotes collection)
    OrderService-->>Frontend: Quote created with ID
    
    User->>Frontend: Convert Quote to Order
    Frontend->>OrderService: POST /orders?fromQuote={quoteId}
    
    OrderService->>MongoDB: Get quote details
    OrderService->>MongoDB: Check inventory availability
    OrderService->>MongoDB: Create order record (orders collection)
    OrderService->>MongoDB: Update order status to "Created"
    OrderService->>MongoDB: Add order creation event
    
    OrderService->>Queue: Send order message (async)
    Integration->>Queue: Poll for new orders
    Integration->>OrderService: PUT /orders/{orderId}/status (Confirmed)
    
    OrderService-->>Frontend: Order created successfully
    Frontend-->>User: Order confirmation with tracking
```

## Workflow 2: Order Fulfillment and Shipment Processing

### Description
Manages the complete order lifecycle from confirmation through shipment and delivery, including status tracking and event logging.

### Communication Patterns
- **Synchronous REST**: Status updates and event logging
- **Asynchronous Queue**: Integration with external systems
- **Database Transactions**: Multi-document updates

```mermaid
sequenceDiagram
    participant User as Warehouse User
    participant Frontend as Web Client
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Integration as Integration Service

    User->>Frontend: View orders for fulfillment
    Frontend->>OrderService: GET /orders?status=Confirmed
    OrderService->>MongoDB: Query orders collection
    OrderService-->>Frontend: List of confirmed orders
    
    User->>Frontend: Start order fulfillment
    Frontend->>OrderService: PUT /orders/{orderId}/status (Started)
    OrderService->>MongoDB: Update order status
    OrderService->>MongoDB: Add "Started" event
    
    User->>Frontend: Mark order as built
    Frontend->>OrderService: PUT /orders/{orderId}/status (Built)
    OrderService->>MongoDB: Update order status
    OrderService->>MongoDB: Add "Built" event
    
    User->>Frontend: Create shipment record
    Frontend->>OrderService: POST /shipments
    OrderService->>MongoDB: Create shipment (shipments collection)
    OrderService->>MongoDB: Update order status to "Shipped"
    OrderService->>MongoDB: Add shipment creation event
    
    Integration->>OrderService: Process external shipment updates
    OrderService->>MongoDB: Update shipment events
    OrderService->>MongoDB: Update order status to "Delivered"
```

## Workflow 3: Catalog and Inventory Synchronization

### Description
Handles product catalog updates and inventory synchronization between the MRP system and external systems via queue-based messaging.

### Communication Patterns
- **Asynchronous Queue**: Product update messages
- **Scheduled Tasks**: Periodic inventory synchronization
- **REST APIs**: External system communication

```mermaid
sequenceDiagram
    participant External as External System
    participant Queue as Azure Queue (inventory)
    participant Integration as Integration Service
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Frontend as Web Client

    External->>Queue: Send product update message
    Integration->>Queue: Poll for product updates (every 30s)
    Integration->>Integration: Parse ProductMessage
    
    alt Update Existing Product
        Integration->>OrderService: PUT /catalog/{sku}
        OrderService->>MongoDB: Update catalog item
        OrderService->>MongoDB: Adjust inventory levels
    else Create New Product
        Integration->>OrderService: POST /catalog
        OrderService->>MongoDB: Create new catalog item
    end
    
    OrderService-->>Integration: Update confirmation
    Integration->>Queue: Delete processed message
    
    Frontend->>OrderService: GET /catalog
    OrderService->>MongoDB: Query catalog collection
    OrderService-->>Frontend: Updated catalog data
```

## Workflow 4: Dealer Management and Quote Generation

### Description
Manages dealer information and facilitates the quote generation process with validation and pricing calculations.

### Communication Patterns
- **Synchronous REST**: CRUD operations for dealers and quotes
- **Database Transactions**: Complex quote calculations
- **Data Validation**: Multi-field validation

```mermaid
sequenceDiagram
    participant User as Sales User
    participant Frontend as Web Client
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database

    User->>Frontend: Access dealer management
    Frontend->>OrderService: GET /dealers
    OrderService->>MongoDB: Query dealers collection
    OrderService-->>Frontend: List of dealers
    
    User->>Frontend: Create new dealer
    Frontend->>OrderService: POST /dealers
    OrderService->>MongoDB: Validate dealer data
    OrderService->>MongoDB: Save dealer record
    OrderService-->>Frontend: Dealer created
    
    User->>Frontend: Create quote for customer
    Frontend->>OrderService: POST /quotes
    OrderService->>MongoDB: Validate dealer exists
    OrderService->>MongoDB: Calculate quote totals
    OrderService->>MongoDB: Apply business rules
    OrderService->>MongoDB: Save quote with expiration
    OrderService-->>Frontend: Quote with calculated pricing
    
    User->>Frontend: Modify quote items
    Frontend->>OrderService: PUT /quotes/{quoteId}
    OrderService->>MongoDB: Recalculate totals
    OrderService->>MongoDB: Update quote record
    OrderService-->>Frontend: Updated quote
```

## Workflow 5: Error Handling and Recovery Patterns

### Description
Demonstrates the system's error handling and recovery mechanisms across different failure scenarios.

### Communication Patterns
- **Exception Handling**: Structured error responses
- **Retry Mechanisms**: Queue message processing
- **Health Checks**: Service monitoring

```mermaid
sequenceDiagram
    participant Frontend as Web Client
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Integration as Integration Service
    participant Queue as Azure Queue
    participant Insights as Application Insights

    Frontend->>OrderService: POST /orders (with invalid data)
    OrderService->>OrderService: Validate request data
    alt Validation Failure
        OrderService->>Insights: Log validation error
        OrderService-->>Frontend: 400 Bad Request with details
    end
    
    Frontend->>OrderService: PUT /orders/{orderId}/status
    OrderService->>MongoDB: Update order status
    alt Database Connection Failure
        OrderService->>Insights: Log database error
        OrderService-->>Frontend: 503 Service Unavailable
        OrderService->>OrderService: Implement retry logic
    end
    
    Integration->>Queue: Poll for messages
    Queue->>Integration: Return order message
    Integration->>OrderService: Process order update
    alt External System Failure
        OrderService->>Insights: Log integration error
        Integration->>Integration: Implement exponential backoff
        Integration->>Queue: Requeue message for retry
    end
    
    Frontend->>OrderService: GET /ping
    OrderService->>MongoDB: Health check query
    OrderService->>OrderService: Verify service health
    OrderService-->>Frontend: 200 OK with health status
```

## Workflow 6: Order Status Tracking and Event Management

### Description
Provides comprehensive order status tracking with event history and real-time updates for stakeholders.

### Communication Patterns
- **Event Sourcing**: Order event history
- **REST APIs**: Status queries and updates
- **Real-time Updates**: Web client polling

```mermaid
sequenceDiagram
    participant User as Customer/Dealer
    participant Frontend as Web Client
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database

    User->>Frontend: Check order status
    Frontend->>OrderService: GET /orders/{orderId}
    OrderService->>MongoDB: Query orders collection
    OrderService->>MongoDB: Retrieve order events
    OrderService-->>Frontend: Order details with current status
    
    User->>Frontend: View order history
    Frontend->>OrderService: GET /orders/{orderId}/events
    OrderService->>MongoDB: Query order events
    OrderService-->>Frontend: Complete event timeline
    
    Frontend->>OrderService: Add order event (async)
    OrderService->>MongoDB: Validate event data
    OrderService->>MongoDB: Append to events array
    OrderService->>MongoDB: Update order status if needed
    OrderService-->>Frontend: Event recorded
    
    Frontend->>Frontend: Poll for status updates (every 30s)
    Frontend->>OrderService: GET /orders/{orderId}
    OrderService->>MongoDB: Get latest status
    OrderService-->>Frontend: Updated order information
```

## Workflow 7: Integration Service Batch Processing

### Description
Shows the scheduled batch processing performed by the Integration Service for order and product synchronization.

### Communication Patterns
- **Scheduled Tasks**: Cron-based processing
- **Batch Operations**: Bulk data processing
- **Queue Management**: Message lifecycle

```mermaid
sequenceDiagram
    participant Scheduler as Spring Scheduler
    participant Integration as Integration Service
    participant Queue as Azure Queue
    participant OrderService as Order Service
    participant External as External MRP System

    Note over Scheduler,External: Every 30 seconds - Order Processing
    
    Scheduler->>Integration: Trigger CreateOrderProcessTask
    Integration->>Queue: Poll orders queue
    Queue->>Integration: Return batch of order messages
    
    loop For each order message
        Integration->>Integration: Parse OrderMessage
        Integration->>External: Send order to MRP system
        External-->>Integration: Order acceptance
        Integration->>OrderService: Update order status
        Integration->>Queue: Delete processed message
    end
    
    Note over Scheduler,External: Every 30 seconds - Product Updates
    
    Scheduler->>Integration: Trigger UpdateProductProcessTask
    Integration->>External: Request product updates
    External-->>Integration: Return product changes
    
    loop For each product update
        Integration->>Queue: Send ProductMessage
        Integration->>OrderService: Sync inventory levels
    end
    
    Integration->>Integration: Log processing statistics
    Integration->>Integration: Handle failures with retry logic
```