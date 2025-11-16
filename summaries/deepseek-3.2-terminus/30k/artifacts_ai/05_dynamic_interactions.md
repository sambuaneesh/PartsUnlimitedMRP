```markdown
# Parts Unlimited MRP System - Dynamic Interaction Flows

## Workflow 1: Quote-to-Order Conversion Process

**Purpose**: Convert a sales quote into a manufacturing order, initiating the production lifecycle.
**Triggers**: Sales representative confirms customer acceptance of quote
**Communication Patterns**: REST API calls, database transactions, event logging

```mermaid
sequenceDiagram
    participant SalesRep as Sales Representative
    participant WebUI as Web Frontend
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Integration as Integration Service
    participant Queue as Azure Storage Queue

    SalesRep->>WebUI: Navigate to quote details
    WebUI->>OrderService: GET /quotes/{quoteId}
    OrderService->>MongoDB: Query quotes collection
    MongoDB-->>OrderService: Return quote data
    OrderService-->>WebUI: Return quote details
    WebUI-->>SalesRep: Display quote information
    
    SalesRep->>WebUI: Click "Create Order" button
    WebUI->>OrderService: POST /orders?fromQuote={quoteId}
    
    OrderService->>MongoDB: Check existing orders for quote
    MongoDB-->>OrderService: No existing order found
    
    OrderService->>OrderService: Validate quote-to-order conversion
    OrderService->>OrderService: Generate order ID: order-{quoteId}
    OrderService->>OrderService: Set initial status: "Created"
    
    OrderService->>MongoDB: Insert order document
    MongoDB-->>OrderService: Order created successfully
    
    OrderService->>MongoDB: Add order event "Order Created"
    OrderService->>Integration: Submit order to processing queue
    Integration->>Queue: Enqueue order message
    
    OrderService-->>WebUI: Return order confirmation
    WebUI-->>SalesRep: Display order created successfully
    
    Integration->>Integration: Process CreateOrderProcessTask
    Integration->>Queue: Dequeue order message
    Integration->>OrderService: Update order status to "Confirmed"
    OrderService->>MongoDB: Update order status
```

## Workflow 2: Order Lifecycle Management

**Purpose**: Track order progression through manufacturing stages from confirmation to installation
**Triggers**: Production team updates order status at each manufacturing milestone
**Communication Patterns**: REST API status updates, event-driven status changes, database transactions

```mermaid
sequenceDiagram
    participant ProdTeam as Production Team
    participant WebUI as Web Frontend
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Shipment as Shipment Service
    participant Integration as Integration Service

    ProdTeam->>WebUI: Access order management dashboard
    WebUI->>OrderService: GET /orders?status=Confirmed
    OrderService->>MongoDB: Query orders collection
    MongoDB-->>OrderService: Return confirmed orders
    OrderService-->>WebUI: Return order list
    WebUI-->>ProdTeam: Display orders ready for production
    
    ProdTeam->>WebUI: Select order to start production
    WebUI->>OrderService: PUT /orders/{orderId}/status (Started)
    OrderService->>OrderService: Validate status transition
    OrderService->>MongoDB: Update order status to "Started"
    OrderService->>MongoDB: Add order event "Production Started"
    OrderService-->>WebUI: Status update confirmed
    
    alt Production Completion
        ProdTeam->>WebUI: Mark order as built
        WebUI->>OrderService: PUT /orders/{orderId}/status (Built)
        OrderService->>MongoDB: Update status to "Built"
        OrderService->>MongoDB: Add event "Manufacturing Complete"
        OrderService->>Shipment: Trigger shipment creation
        Shipment->>MongoDB: Create shipment record
    end
    
    alt Delivery Confirmation
        ProdTeam->>WebUI: Confirm delivery preparation
        WebUI->>OrderService: PUT /orders/{orderId}/status (DeliveryConfirmed)
        OrderService->>MongoDB: Update status to "DeliveryConfirmed"
        OrderService->>Integration: Notify shipping department
    end
    
    alt Installation Complete
        ProdTeam->>WebUI: Mark installation complete
        WebUI->>OrderService: PUT /orders/{orderId}/status (Installed)
        OrderService->>MongoDB: Update status to "Installed"
        OrderService->>MongoDB: Add event "Project Installation Complete"
        OrderService->>Integration: Trigger billing process
    end
```

## Workflow 3: Catalog Management and Inventory Updates

**Purpose**: Manage product catalog items and maintain accurate inventory levels
**Triggers**: Inventory manager adds/updates products or stock levels change
**Communication Patterns**: REST CRUD operations, database transactions, batch updates

```mermaid
sequenceDiagram
    participant InvManager as Inventory Manager
    participant WebUI as Web Frontend
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Integration as Integration Service
    participant Queue as Azure Storage Queue

    InvManager->>WebUI: Navigate to catalog management
    WebUI->>OrderService: GET /catalog
    OrderService->>MongoDB: Query catalog collection
    MongoDB-->>OrderService: Return all catalog items
    OrderService-->>WebUI: Return catalog data
    WebUI-->>InvManager: Display product catalog
    
    alt Add New Product
        InvManager->>WebUI: Click "Add Product"
        WebUI->>InvManager: Display product form
        InvManager->>WebUI: Enter product details (SKU, description, price, inventory)
        WebUI->>OrderService: POST /catalog (new product data)
        OrderService->>OrderService: Validate product data
        OrderService->>MongoDB: Insert catalog document
        MongoDB-->>OrderService: Product created
        OrderService-->>WebUI: Return created product
        WebUI-->>InvManager: Display success message
    end
    
    alt Update Inventory
        InvManager->>WebUI: Update product inventory level
        WebUI->>OrderService: PUT /catalog/{sku} (updated inventory)
        OrderService->>MongoDB: Update catalog document
        MongoDB-->>OrderService: Inventory updated
        OrderService->>OrderService: Recalculate lead time
        OrderService-->>WebUI: Return updated product
    end
    
    alt Batch Product Updates
        Integration->>Integration: Execute UpdateProductProcessTask
        Integration->>Queue: Check for product update messages
        Queue-->>Integration: Return update messages
        Integration->>OrderService: POST /catalog (batch updates)
        OrderService->>MongoDB: Bulk update catalog items
    end
```

## Workflow 4: Shipment Tracking and Delivery Management

**Purpose**: Track order shipments from warehouse to customer site with event logging
**Triggers**: Shipping department creates shipments and updates delivery status
**Communication Patterns**: REST API calls, event logging, status synchronization

```mermaid
sequenceDiagram
    participant ShipDept as Shipping Department
    participant WebUI as Web Frontend
    participant OrderService as Order Service
    participant Shipment as Shipment Controller
    participant MongoDB as MongoDB Database
    participant GoogleAPI as Google Maps API

    ShipDept->>WebUI: Navigate to shipment management
    WebUI->>OrderService: GET /orders?status=Built
    OrderService->>MongoDB: Query orders ready for shipment
    MongoDB-->>OrderService: Return built orders
    OrderService-->>WebUI: Return order list
    WebUI-->>ShipDept: Display orders ready for shipping
    
    ShipDept->>WebUI: Select order to create shipment
    WebUI->>Shipment: GET /shipments/{orderId} (check existing)
    Shipment->>MongoDB: Query shipments collection
    MongoDB-->>Shipment: No existing shipment
    
    ShipDept->>WebUI: Enter delivery address
    WebUI->>GoogleAPI: Validate address via Places API
    GoogleAPI-->>WebUI: Return validated address
    WebUI->>ShipDept: Display validated address
    
    ShipDept->>WebUI: Confirm shipment creation
    WebUI->>Shipment: POST /shipments (shipment data)
    Shipment->>Shipment: Validate shipment data
    Shipment->>MongoDB: Insert shipment document
    MongoDB-->>Shipment: Shipment created
    Shipment->>OrderService: Update order status to "Shipped"
    OrderService->>MongoDB: Update order status
    
    alt Shipment Event Tracking
        ShipDept->>WebUI: Add shipment event (departure, transit, etc.)
        WebUI->>Shipment: POST /shipments/{orderId}/events
        Shipment->>MongoDB: Add event to shipment events array
        Shipment->>OrderService: Sync relevant status updates
    end
    
    alt Delivery Confirmation
        ShipDept->>WebUI: Confirm delivery completion
        WebUI->>Shipment: PUT /shipments/{orderId} (mark delivered)
        Shipment->>MongoDB: Update shipment status
        Shipment->>OrderService: Update order status to "Delivered"
        OrderService->>MongoDB: Update order status and add event
    end
```

## Workflow 5: Dealer and Customer Relationship Management

**Purpose**: Manage dealer information and customer relationships throughout sales process
**Triggers**: Sales team adds new dealers or updates existing dealer information
**Communication Patterns**: REST CRUD operations, data validation, relational data management

```mermaid
sequenceDiagram
    participant SalesTeam as Sales Team
    participant WebUI as Web Frontend
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Quote as Quote Controller

    SalesTeam->>WebUI: Navigate to dealer management
    WebUI->>OrderService: GET /dealers
    OrderService->>MongoDB: Query dealers collection
    MongoDB-->>OrderService: Return all dealers
    OrderService-->>WebUI: Return dealer list
    WebUI-->>SalesTeam: Display dealer directory
    
    alt Add New Dealer
        SalesTeam->>WebUI: Click "Add Dealer"
        WebUI->>SalesTeam: Display dealer form
        SalesTeam->>WebUI: Enter dealer details (name, contact, address, email, phone)
        WebUI->>OrderService: POST /dealers (new dealer data)
        OrderService->>OrderService: Validate dealer data
        OrderService->>MongoDB: Insert dealer document
        MongoDB-->>OrderService: Dealer created
        OrderService-->>WebUI: Return created dealer
        WebUI-->>SalesTeam: Display success message
    end
    
    alt Create Quote for Dealer
        SalesTeam->>WebUI: Navigate to quote creation
        WebUI->>OrderService: GET /dealers (for dropdown)
        OrderService-->>WebUI: Return dealer list
        WebUI->>SalesTeam: Display quote form with dealer selection
        
        SalesTeam->>WebUI: Select dealer, enter customer details
        SalesTeam->>WebUI: Add quote items from catalog
        WebUI->>OrderService: GET /catalog (for product selection)
        OrderService-->>WebUI: Return catalog items
        
        SalesTeam->>WebUI: Submit quote
        WebUI->>Quote: POST /quotes (quote data with dealer reference)
        Quote->>Quote: Calculate total cost with discounts
        Quote->>MongoDB: Insert quote document
        MongoDB-->>Quote: Quote created
        Quote-->>WebUI: Return created quote with ID
        WebUI-->>SalesTeam: Display quote confirmation
    end
    
    alt Update Dealer Information
        SalesTeam->>WebUI: Edit dealer details
        WebUI->>OrderService: PUT /dealers/{name} (updated data)
        OrderService->>MongoDB: Update dealer document
        MongoDB-->>OrderService: Dealer updated
        OrderService-->>WebUI: Return updated dealer
    end
```

## Workflow 6: System Health Monitoring and Error Recovery

**Purpose**: Monitor system health, handle errors, and ensure service availability
**Triggers**: Periodic health checks, system errors, or manual status requests
**Communication Patterns**: Health check endpoints, exception handling, retry mechanisms

```mermaid
sequenceDiagram
    participant Monitor as Monitoring System
    participant LoadBalancer as Load Balancer
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant Integration as Integration Service
    participant Queue as Azure Storage Queue

    Monitor->>LoadBalancer: Periodic health check
    LoadBalancer->>OrderService: HEAD /ping
    OrderService->>OrderService: Check internal health status
    OrderService->>MongoDB: Test database connection
    MongoDB-->>OrderService: Connection successful
    OrderService-->>LoadBalancer: Return 200 OK
    LoadBalancer-->>Monitor: Service healthy
    
    alt Database Connection Failure
        OrderService->>MongoDB: Database operation
        MongoDB-->>OrderService: Connection timeout
        OrderService->>OrderService: Retry logic (3 attempts)
        OrderService->>MongoDB: Retry database operation
        MongoDB-->>OrderService: Operation successful
        OrderService-->>WebUI: Return successful response
    end
    
    alt Queue Processing Error
        Integration->>Integration: Execute scheduled task
        Integration->>Queue: Dequeue message
        Queue-->>Integration: Return message
        Integration->>OrderService: Process message
        OrderService-->>Integration: Service unavailable
        
        Integration->>Integration: Implement exponential backoff
        Integration->>Integration: Log error for manual review
        Integration->>Queue: Re-queue message with delay
        
        Integration->>Integration: Wait and retry
        Integration->>OrderService: Retry operation
        OrderService-->>Integration: Operation successful
    end
    
    alt System Startup and Dependency Check
        OrderService->>OrderService: Application startup
        OrderService->>MongoDB: Verify database connectivity
        MongoDB-->>OrderService: Connection established
        OrderService->>OrderService: Initialize repositories
        OrderService->>Integration: Verify integration service
        Integration-->>OrderService: Service available
        OrderService->>OrderService: Mark service as ready
    end
```

## Workflow 7: Asynchronous Order Processing via Integration Service

**Purpose**: Handle background order processing and external system integration
**Triggers**: Scheduled tasks, queue messages, external system updates
**Communication Patterns**: Queue-based messaging, scheduled batch processing, event-driven updates

```mermaid
sequenceDiagram
    participant Scheduler as Task Scheduler
    participant Integration as Integration Service
    participant Queue as Azure Storage Queue
    participant OrderService as Order Service
    participant MongoDB as MongoDB Database
    participant External as External MRP System

    Scheduler->>Integration: Trigger CreateOrderProcessTask
    Integration->>Queue: Check for new order messages
    Queue-->>Integration: Return queued order messages
    
    loop For each order message
        Integration->>Integration: Parse order message
        Integration->>OrderService: GET /orders/{orderId}
        OrderService->>MongoDB: Query order details
        MongoDB-->>OrderService: Return order data
        OrderService-->>Integration: Return order information
        
        Integration->>Integration: Validate order for processing
        Integration->>External: Submit order to external MRP
        External-->>Integration: Order accepted
        
        Integration->>OrderService: Update order status to "Processing"
        OrderService->>MongoDB: Update order status
        OrderService->>MongoDB: Add processing event
        
        Integration->>Queue: Remove processed message
    end
    
    Scheduler->>Integration: Trigger UpdateProductProcessTask
    Integration->>External: Check for product updates
    External-->>Integration: Return updated products
    
    loop For each product update
        Integration->>Integration: Transform product data
        Integration->>OrderService: PUT /catalog/{sku}
        OrderService->>MongoDB: Update catalog item
        MongoDB-->>OrderService: Update confirmed
        OrderService-->>Integration: Product updated
    end
    
    alt Error in External Integration
        Integration->>External: Submit data
        External-->>Integration: Service timeout
        Integration->>Integration: Implement circuit breaker
        Integration->>Integration: Log integration failure
        Integration->>Queue: Requeue message for retry
    end
```