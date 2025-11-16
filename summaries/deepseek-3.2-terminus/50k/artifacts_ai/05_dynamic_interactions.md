```markdown
# Parts Unlimited MRP System - Dynamic Interaction Flows

## Workflow 1: Quote-to-Order Conversion Flow

### Description
This workflow captures the complete process from creating a customer quote to converting it into a manufacturing order. It involves multiple services and demonstrates the state transition from quote to order with validation and business rule enforcement.

**Triggers**: Dealer creates new quote or converts existing quote to order
**Communication Patterns**: REST API calls, database transactions, synchronous service communication

### Sequence Diagram

```mermaid
sequenceDiagram
    participant F as Frontend Client
    participant OS as OrderService
    participant DB as MongoDB
    participant IS as IntegrationService
    participant MQ as Azure Queue

    Note over F,MQ: Quote Creation Phase
    F->>OS: POST /quotes (quote details)
    OS->>DB: Validate dealer exists
    DB-->>OS: Dealer details
    OS->>DB: Validate catalog items
    DB-->>OS: Catalog items with inventory
    OS->>OS: Calculate pricing & discounts
    OS->>DB: Save quote with status "Draft"
    DB-->>OS: Quote saved with quoteId
    OS-->>F: Return created quote

    Note over F,MQ: Quote Validation & Conversion
    F->>OS: PUT /quotes/{quoteId} (finalize)
    OS->>DB: Retrieve quote
    DB-->>OS: Quote details
    OS->>OS: Validate expiration & completeness
    OS->>DB: Update quote status to "Valid"
    DB-->>OS: Quote updated
    
    F->>OS: POST /orders?fromQuote={quoteId}
    OS->>DB: Retrieve validated quote
    DB-->>OS: Quote with validUntil check
    OS->>OS: Convert quote to order
    OS->>DB: Create order with status "Created"
    OS->>DB: Add order creation event
    DB-->>OS: Order created with orderId
    OS-->>F: Return created order

    Note over F,MQ: Order Processing Initiation
    F->>OS: PUT /orders/{orderId}/status (Confirmed)
    OS->>DB: Update order status to "Confirmed"
    OS->>DB: Add status change event
    DB-->>OS: Order updated
    
    OS->>IS: Queue order for MRP processing
    IS->>MQ: Send order message to "orders" queue
    MQ-->>IS: Message queued successfully
    OS-->>F: Order confirmed and queued
```

## Workflow 2: Asynchronous Order Processing Flow

### Description
This workflow shows the background processing of orders from the website queue to the MRP system. It demonstrates asynchronous message processing, scheduled tasks, and integration with external manufacturing systems.

**Triggers**: Scheduled task execution (30-second intervals), new orders in queue
**Communication Patterns**: Queue-based messaging, scheduled batch processing, REST API calls to external systems

### Sequence Diagram

```mermaid
sequenceDiagram
    participant IS as IntegrationService
    participant MQ as Azure Queue (orders)
    participant MRP as External MRP System
    participant OS as OrderService
    participant DB as MongoDB

    Note over IS,DB: Scheduled Order Processing Task
    loop Every 30 seconds
        IS->>MQ: Poll for order messages
        MQ-->>IS: Order message batch
        
        loop For each order message
            IS->>IS: Parse order message
            IS->>MRP: POST /mrp/orders (order data)
            MRP-->>IS: Order accepted by MRP
            
            alt Successful processing
                IS->>OS: GET /orders/{orderId}
                OS-->>IS: Order details
                IS->>OS: PUT /orders/{orderId}/status (Started)
                OS->>DB: Update order status
                DB-->>OS: Status updated
                IS->>MQ: Delete processed message
            else Processing failed
                IS->>IS: Increment retry count
                IS->>MQ: Release message with delay
                IS->>OS: POST /orders/{orderId}/events (error)
            end
        end
    end

    Note over IS,DB: MRP Status Updates
    MRP->>IS: POST /mrp/orders/{id}/status (Built)
    IS->>OS: PUT /orders/{orderId}/status (Built)
    OS->>DB: Update order status to "Built"
    OS->>DB: Add manufacturing complete event
    DB-->>OS: Order updated
    OS-->>IS: Status updated confirmation
```

## Workflow 3: Inventory Synchronization Flow

### Description
This workflow handles the bidirectional synchronization of product inventory between the MRP system and the website. It demonstrates event-driven updates and scheduled polling patterns for maintaining data consistency across systems.

**Triggers**: Inventory changes in MRP system, scheduled synchronization tasks
**Communication Patterns**: Queue-based messaging, scheduled polling, REST API updates

### Sequence Diagram

```mermaid
sequenceDiagram
    participant MRP as External MRP System
    participant IS as IntegrationService
    participant MQ as Azure Queue (inventory)
    participant OS as OrderService
    participant DB as MongoDB

    Note over MRP,DB: MRP → Website Inventory Updates
    MRP->>IS: Product inventory change event
    IS->>IS: Transform to product message
    IS->>MQ: Send product update to "product" queue
    MQ-->>IS: Message queued

    Note over IS,DB: Scheduled Product Processing Task
    loop Every 30 seconds
        IS->>MQ: Poll for product messages
        MQ-->>IS: Product message batch
        
        loop For each product message
            IS->>IS: Parse product update
            IS->>OS: PUT /catalog/{sku} (inventory update)
            OS->>DB: Update catalog item inventory
            DB-->>OS: Catalog updated
            OS-->>IS: Update confirmation
            IS->>MQ: Delete processed message
        end
    end

    Note over IS,DB: Website → MRP Catalog Polling
    IS->>MRP: GET /mrp/catalog (latest items)
    MRP-->>IS: Catalog items with inventory
    IS->>IS: Compare with local catalog
    alt New or updated items found
        IS->>OS: POST /catalog (batch update)
        OS->>DB: Update catalog collection
        DB-->>OS: Catalog items saved
    end
```

## Workflow 4: Order Fulfillment and Shipment Tracking

### Description
This workflow covers the complete order fulfillment process from manufacturing completion through shipment creation, tracking, and final delivery. It demonstrates status state machine transitions and event-driven notifications.

**Triggers**: Order status changes, shipment creation, delivery events
**Communication Patterns**: REST API calls, database transactions, event logging

### Sequence Diagram

```mermaid
sequenceDiagram
    participant F as Frontend Client
    participant OS as OrderService
    participant DB as MongoDB
    participant IS as IntegrationService
    participant MRP as External MRP System

    Note over F,MRP: Shipment Creation
    F->>OS: POST /shipments (for orderId)
    OS->>DB: Validate order exists and is "Built"
    DB-->>OS: Order details
    OS->>DB: Create shipment record
    OS->>DB: Add shipment creation event
    DB-->>OS: Shipment created
    OS-->>F: Return shipment details

    Note over F,MRP: Shipping Preparation
    F->>OS: PUT /orders/{orderId}/status (Shipped)
    OS->>DB: Update order status to "Shipped"
    OS->>DB: Add shipping event
    DB-->>OS: Order updated
    
    OS->>IS: Notify shipment ready
    IS->>MRP: Update shipment status in MRP
    MRP-->>IS: MRP updated
    OS-->>F: Order shipped confirmation

    Note over F,MRP: Delivery Tracking
    F->>OS: POST /shipments/{id}/events (In Transit)
    OS->>DB: Add shipment tracking event
    DB-->>OS: Event logged
    
    F->>OS: POST /shipments/{id}/events (Out for Delivery)
    OS->>DB: Add delivery event
    DB-->>OS: Event logged
    
    F->>OS: PUT /orders/{orderId}/status (Delivered)
    OS->>DB: Update order status to "Delivered"
    OS->>DB: Add delivery completion event
    DB-->>OS: Order fulfillment completed
    OS-->>F: Delivery confirmed
```

## Workflow 5: Error Handling and Recovery Patterns

### Description
This workflow demonstrates the system's resilience patterns including queue message retry logic, error event logging, and recovery mechanisms for failed operations.

**Triggers**: Failed API calls, queue processing errors, timeout scenarios
**Communication Patterns**: Retry patterns, error logging, dead letter queues, health monitoring

### Sequence Diagram

```mermaid
sequenceDiagram
    participant IS as IntegrationService
    participant MQ as Azure Queue
    participant MRP as External MRP System
    participant OS as OrderService
    participant DB as MongoDB
    participant AI as Application Insights

    Note over IS,AI: Queue Processing Error Scenario
    IS->>MQ: Poll for messages
    MQ-->>IS: Order message
    IS->>MRP: POST /mrp/orders (order data)
    
    alt MRP System Unavailable
        MRP-->>IS: HTTP 503 / Timeout
        IS->>IS: Increment retry counter
        IS->>AI: Log processing failure
        alt Retry count < max retries
            IS->>MQ: Release message with visibility delay
            MQ-->>IS: Message requeued
        else Max retries exceeded
            IS->>MQ: Move to dead-letter queue
            IS->>OS: POST /orders/{orderId}/events (processing failed)
            OS->>DB: Log critical error event
            DB-->>OS: Event saved
        end
    end

    Note over IS,AI: Recovery Mechanism
    IS->>OS: GET /health
    OS-->>IS: Service health status
    alt Service degraded
        IS->>IS: Implement circuit breaker
        IS->>MQ: Pause processing temporarily
    end

    Note over IS,AI: Successful Recovery
    IS->>MRP: GET /health
    MRP-->>IS: MRP available
    IS->>IS: Reset circuit breaker
    IS->>MQ: Resume normal processing
    MQ-->>IS: Next message batch
```

## Workflow 6: Catalog Management and Inventory Updates

### Description
This workflow shows the complete lifecycle of catalog item management including creation, updates, inventory adjustments, and synchronization with external systems.

**Triggers**: New product creation, inventory adjustments, price changes
**Communication Patterns**: REST API calls, database transactions, event-driven synchronization

### Sequence Diagram

```mermaid
sequenceDiagram
    participant F as Frontend Client
    participant OS as OrderService
    participant DB as MongoDB
    participant IS as IntegrationService
    participant MQ as Azure Queue

    Note over F,MQ: Catalog Item Creation
    F->>OS: POST /catalog (new item)
    OS->>DB: Validate SKU uniqueness
    DB-->>OS: SKU available
    OS->>DB: Save catalog item
    DB-->>OS: Item saved
    OS-->>F: Return created item
    
    OS->>IS: Notify new catalog item
    IS->>MQ: Send product message to queue
    MQ-->>IS: Message queued

    Note over F,MQ: Inventory Adjustment
    F->>OS: PUT /catalog/{sku} (inventory update)
    OS->>DB: Update inventory levels
    DB-->>OS: Inventory updated
    OS->>DB: Recalculate lead times
    DB-->>OS: Lead times updated
    OS-->>F: Inventory adjustment confirmed

    OS->>IS: Notify inventory change
    IS->>MQ: Send inventory update message
    MQ-->>IS: Message queued

    Note over F,MQ: Price Update Flow
    F->>OS: PUT /catalog/{sku} (price update)
    OS->>DB: Update pricing
    OS->>DB: Recalculate affected quotes
    DB-->>OS: Prices and quotes updated
    OS-->>F: Price update completed
```

## Communication Patterns Summary

### Synchronous Communication
- **Frontend ↔ OrderService**: REST API calls for immediate operations
- **IntegrationService ↔ MRP System**: Direct REST calls for order processing
- **Health Checks**: Service monitoring and readiness verification

### Asynchronous Communication  
- **Queue-based Messaging**: Order and product updates via Azure Storage Queues
- **Scheduled Processing**: 30-second intervals for batch operations
- **Event-driven Updates**: Inventory and status change notifications

### Data Flow Patterns
- **Request-Response**: Immediate operations requiring confirmation
- **Fire-and-Forget**: Non-critical updates and notifications
- **Polling**: Regular synchronization and health monitoring
- **Event Sourcing**: Audit trail through event logging

### Error Handling Patterns
- **Retry with Exponential Backoff**: Queue message processing
- **Circuit Breaker**: Service dependency failure protection
- **Dead Letter Queues**: Failed message handling
- **Health Monitoring**: Proactive system status tracking
```