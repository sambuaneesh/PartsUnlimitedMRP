```markdown
# Parts Unlimited MRP - Dynamic Interaction Flows

## 1. Quote-to-Order Processing Workflow

### Workflow Description
This workflow represents the core business process where a customer quote is converted into an order and progresses through manufacturing and delivery stages. It involves multiple services and status transitions from quote creation through final delivery.

**Triggers**: Customer quote creation, quote acceptance, order confirmation
**Communication Patterns**: Synchronous REST calls, database transactions, event logging

```mermaid
sequenceDiagram
    participant User as Web Client
    participant QuoteSvc as Quote Service
    participant OrderSvc as Order Service
    participant ShipSvc as Shipment Service
    participant DB as MongoDB
    participant IntSvc as Integration Service

    User->>QuoteSvc: POST /quotes (Create Quote)
    QuoteSvc->>DB: Save quote data
    DB-->>QuoteSvc: Quote created
    QuoteSvc-->>User: Quote ID returned
    
    User->>OrderSvc: POST /orders?fromQuote={quoteId}
    OrderSvc->>QuoteSvc: GET /quotes/{quoteId}
    QuoteSvc-->>OrderSvc: Quote details
    OrderSvc->>DB: Create order (Status: Created)
    DB-->>OrderSvc: Order created
    OrderSvc->>OrderSvc: Generate order events
    OrderSvc-->>User: Order ID returned
    
    User->>OrderSvc: PUT /orders/{orderId}/status (Confirmed)
    OrderSvc->>DB: Update order status
    OrderSvc->>IntSvc: Queue order processing
    IntSvc->>OrderSvc: Process order manufacturing
    OrderSvc->>DB: Update status (Started → Built)
    
    User->>ShipSvc: POST /shipments (Create Shipment)
    ShipSvc->>OrderSvc: GET /orders/{orderId}
    OrderSvc-->>ShipSvc: Order details
    ShipSvc->>DB: Create shipment record
    ShipSvc-->>User: Shipment created
    
    loop Delivery Events
        User->>ShipSvc: POST /shipments/{orderId}/events
        ShipSvc->>DB: Add shipment event
        ShipSvc->>OrderSvc: PUT /orders/{orderId}/status
        OrderSvc->>DB: Update order status
    end
    
    Note over OrderSvc,ShipSvc: Status progression:<br/>DeliveryConfirmed → Shipped → Delivered → Installed
```

## 2. Catalog Management and Inventory Updates

### Workflow Description
This workflow handles product catalog management including inventory updates and pricing changes. It demonstrates synchronous data operations and potential integration with external systems for product synchronization.

**Triggers**: Product updates, inventory changes, pricing adjustments
**Communication Patterns**: REST CRUD operations, scheduled batch processing, queue-based updates

```mermaid
sequenceDiagram
    participant Admin as Administrator
    participant Web as Web Client
    participant CatalogSvc as Catalog Service
    participant DB as MongoDB
    participant IntSvc as Integration Service
    participant ExtSys as External System

    Admin->>Web: Navigate to Catalog Management
    Web->>CatalogSvc: GET /catalog
    CatalogSvc->>DB: Query catalog items
    DB-->>CatalogSvc: Catalog data
    CatalogSvc-->>Web: Product list
    
    Admin->>Web: Update product (price/inventory)
    Web->>CatalogSvc: PUT /catalog/{sku}
    CatalogSvc->>DB: Update catalog item
    DB-->>CatalogSvc: Update confirmed
    CatalogSvc-->>Web: Success response
    
    par External Integration
        IntSvc->>IntSvc: Scheduled UpdateProductProcessTask
        IntSvc->>ExtSys: Check for product updates
        ExtSys-->>IntSvc: Product update messages
        IntSvc->>CatalogSvc: POST /catalog (batch update)
        CatalogSvc->>DB: Bulk update catalog
    and Inventory Sync
        CatalogSvc->>IntSvc: Inventory level changes
        IntSvc->>ExtSys: Sync inventory data
    end
```

## 3. Asynchronous Order Processing via Integration Service

### Workflow Description
This workflow shows the asynchronous processing of orders through queue-based messaging, demonstrating event-driven architecture patterns for handling external system integration and batch processing.

**Triggers**: New orders from external systems, scheduled batch processing
**Communication Patterns**: Queue-based messaging, scheduled tasks, REST API calls

```mermaid
sequenceDiagram
    participant ExtSite as Parts Unlimited Website
    participant Queue as Azure Storage Queue
    participant IntSvc as Integration Service
    participant OrderSvc as Order Service
    participant DB as MongoDB

    ExtSite->>Queue: Enqueue OrderMessage
    Note over ExtSite,Queue: External website places order
    
    loop Scheduled Processing
        IntSvc->>IntSvc: CreateOrderProcessTask execution
        IntSvc->>Queue: Dequeue OrderMessage
        Queue-->>IntSvc: Order data
        
        alt Valid Order Message
            IntSvc->>OrderSvc: POST /orders (Create order)
            OrderSvc->>DB: Save order data
            DB-->>OrderSvc: Order created
            OrderSvc-->>IntSvc: Order confirmation
            IntSvc->>IntSvc: Log processing success
        else Invalid Message
            IntSvc->>IntSvc: Log error and retry logic
            IntSvc->>Queue: Requeue or dead-letter
        end
    end
    
    Note over IntSvc: Batch processing with<br/>configurable intervals
```

## 4. Shipment Tracking and Delivery Management

### Workflow Description
This workflow illustrates the complete shipment lifecycle from creation through delivery confirmation, including event tracking and status synchronization with the order system.

**Triggers**: Order ready for shipment, delivery events, status updates
**Communication Patterns**: REST APIs, event logging, status propagation

```mermaid
sequenceDiagram
    participant User as Web Client
    participant ShipSvc as Shipment Service
    participant OrderSvc as Order Service
    participant DB as MongoDB
    participant Driver as Delivery Driver

    User->>ShipSvc: POST /shipments (Create shipment)
    ShipSvc->>OrderSvc: GET /orders/{orderId}
    OrderSvc-->>ShipSvc: Order details
    ShipSvc->>DB: Create shipment record
    ShipSvc-->>User: Shipment created
    
    User->>ShipSvc: POST /shipments/{orderId}/events (Pickup)
    ShipSvc->>DB: Add shipment event
    ShipSvc->>OrderSvc: PUT /orders/{orderId}/status (Shipped)
    OrderSvc->>DB: Update order status
    
    Driver->>ShipSvc: POST /shipments/{orderId}/events (Out for delivery)
    ShipSvc->>DB: Add delivery event
    
    Driver->>ShipSvc: POST /shipments/{orderId}/events (Delivered)
    ShipSvc->>DB: Add delivery confirmation
    ShipSvc->>OrderSvc: PUT /orders/{orderId}/status (Delivered)
    OrderSvc->>DB: Update order status
    
    User->>ShipSvc: POST /shipments/{orderId}/events (Installed)
    ShipSvc->>DB: Add installation event
    ShipSvc->>OrderSvc: PUT /orders/{orderId}/status (Installed)
    OrderSvc->>DB: Final status update
```

## 5. Error Handling and Recovery Patterns

### Workflow Description
This workflow demonstrates the system's error handling and recovery mechanisms, including database connectivity issues, service unavailability, and message processing failures.

**Triggers**: Service failures, network timeouts, invalid data
**Communication Patterns**: Retry logic, fallback mechanisms, health checks, circuit breaker patterns

```mermaid
sequenceDiagram
    participant Client as Web Client
    participant OrderSvc as Order Service
    participant DB as MongoDB
    participant Health as Health Check
    participant Retry as Retry Mechanism

    Client->>OrderSvc: POST /orders (Create order)
    
    alt Database Connection Failure
        OrderSvc->>DB: Save order data
        DB-->>OrderSvc: Connection timeout
        OrderSvc->>Retry: Initiate retry logic
        loop Retry (3 attempts)
            Retry->>DB: Retry database operation
            alt Success on retry
                DB-->>Retry: Operation successful
                Retry-->>OrderSvc: Success
                OrderSvc-->>Client: Order created
            else All retries failed
                Retry-->>OrderSvc: Permanent failure
                OrderSvc-->>Client: Service unavailable error
            end
        end
    else Service Health Issues
        Health->>OrderSvc: Periodic health check
        OrderSvc-->>Health: Unhealthy status
        Note over Health: Circuit breaker opens<br/>temporarily rejects requests
        Client->>OrderSvc: Request (rejected)
        OrderSvc-->>Client: Service unavailable
    end
    
    alt Invalid Data Validation
        Client->>OrderSvc: POST /orders (Invalid data)
        OrderSvc->>OrderSvc: Validate request data
        OrderSvc-->>Client: Validation errors (400 Bad Request)
    end
    
    Note over OrderSvc,DB: Recovery actions:<br/>- Database reconnection<br/>- Service restart<br/>- Queue message retry
```

## 6. System Health Monitoring and Deployment Verification

### Workflow Description
This workflow shows the system monitoring and health verification processes, including ping endpoints, build information checks, and deployment validation.

**Triggers**: Health checks, deployment events, monitoring alerts
**Communication Patterns**: REST health endpoints, configuration validation, telemetry data

```mermaid
sequenceDiagram
    participant Monitor as Monitoring System
    participant LB as Load Balancer
    participant OrderSvc as Order Service
    participant DB as MongoDB
    participant Insights as Application Insights

    LB->>OrderSvc: HEAD /ping (Health check)
    OrderSvc->>DB: Test database connection
    DB-->>OrderSvc: Connection successful
    OrderSvc-->>LB: 200 OK
    
    Monitor->>OrderSvc: GET /ping (Detailed status)
    OrderSvc->>OrderSvc: Check build information
    OrderSvc->>OrderSvc: Validate configuration
    OrderSvc->>Insights: Send telemetry data
    OrderSvc-->>Monitor: Status with build info
    
    alt Service Degradation
        OrderSvc->>DB: Database operation
        DB-->>OrderSvc: Slow response
        OrderSvc->>Insights: Log performance metric
        Insights-->>Monitor: Alert on threshold breach
    end
    
    Note over Monitor,OrderSvc: Continuous monitoring with:<br/>- Database connectivity checks<br/>- Configuration validation<br/>- Performance metrics<br/>- Build version verification
```
```