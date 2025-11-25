<think>
Okay, let's see. The user wants me to create dynamic interaction flows and sequence diagrams based on the provided architectural summary. The goal is to capture key workflows, communication patterns, data flow, event-driven interactions, and error handling. 

First, I need to identify the major workflows in the PartsUnlimited MRP system. The summary mentions several services like OrderService, QuoteService, ShipmentService, etc. The main user interactions might involve creating an order from a quote, managing dealer information, and tracking shipments.

For the first workflow, creating an order from a quote makes sense. When a user submits a quote, the system needs to validate it, create an order, and maybe notify other services. The steps would involve the client API sending a request to the QuoteService, which validates the quote and checks inventory. Then, the OrderService creates the order and logs events. Also, the ShipmentService might need to be involved for shipping details. Communication patterns here are likely REST calls. The sequence diagram should show these services interacting step by step.

Next, tracking a shipment is another key workflow. The client would request shipment information, which involves the ShipmentService retrieving data and possibly updating the order status. This might be a synchronous REST call. Also, maybe there's an event-driven aspect if the shipment status is updated via messages (like Kafka or RabbitMQ topics). If the summary mentions event-driven updates, that should be included with message flows via an event bus.

Error handling is important too. For example, if a quote validation fails when creating an order, the system should return an error. The QuoteService might throw a ConflictException or ObjectNotFoundException. The sequence diagram would show error responses using `alt` and `else` in Mermaid to represent branches.

Including different communication patterns: REST for synchronous API calls between services, and messages/events for asynchronous updates. For example, when an order is shipped, the ShipmentService could publish an event that the OrderService subscribes to, updating the order status asynchronously. The diagram should show a message being published to an event bus.

I should also consider events like the "Truck dispatched" or "Delivery method changed" in the OrderEvents, which might be handled via message queues. The sequence diagram for event-driven interactions would show a service sending a message to a queue, another service consuming it, and updating the order status.

Another workflow could be the dealer registration process. The client API interacts with DealersService to create a new dealer, which might involve database transactions and possible notifications. But maybe the main ones are order creation and shipment tracking.

I need to make sure each diagram uses Mermaid syntax. For example, defining participants like Client, API Gateway, OrderService, QuoteService, etc. Then the steps with arrows for messages. Also, using different types for messages (solid for synchronous, dashed or arrows labeled as messages for asynchronous).

I should check the summary for specific details. The OrderService uses MongoDB, ShipmentService might use MongoDB too. Events are stored in OrderEvents. So when an order is created, OrderService creates an event in the database. When a shipment is created, it might log the event via a message to an event bus.

Also, the summary mentions ARM templates for deployment, but that's infrastructure-related and not part of the workflows. However, communication patterns in deployment (like Docker containers) might influence the services' dependencies.

Need to make sure each diagram is labeled correctly with the workflow's purpose, triggers, communication types, and data flow. Each sequence diagram should be a separate Mermaid block in markdown. Also, include the key points in a brief description before each diagram.

Let me structure this step by step for each major workflow. Start with Order Creation from Quote, then Shipment Tracking, then Event-Driven Shipment Update. Also, handle error scenarios in the first workflow.

Make sure the diagrams are not too complex but capture the essential interactions. For asynchronous, maybe use a message queue component. For synchronous, use direct arrows between services.

Check for any other workflows mentioned in the summary. Maybe dealer services or catalog services, but the user didn't ask for every single one, just major ones. Focus on the most critical paths highlighted in the summary.
</think>

# Dynamic Interaction Flows: PartsUnlimited MRP System

---

## **1. Order Creation from Quote Workflow**
**Purpose:** Allow users to convert validated quotes into actual orders.  
**Triggers:** User submits a quote for order creation via frontend.  
**Communication Patterns:** Synchronous REST API requests, MongoDB transactions.

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participantQuoteService as QuoteService
    participantOrderService as OrderService
    participantMongoDB
    participantEventBus
    Client->>APIGateway: POST /api/orders (quoteId, dealerName)
    APIGateway->>QuoteService: GET /api/quotes/{quoteId}
    QuoteService->>MongoDB: Validate quote existence
    alt Quote valid
        QuoteService-->>APIGateway: Return quote details
        APIGateway->>OrderService: POST /api/orders (quote details)
        OrderService->>MongoDB: Insert new order
        OrderService->>MongoDB: Log event (Order Created)
        OrderService->>EventBus: Publish event (Order Created)
        EventBus-->>APIGateway: Acknowledge event
        APIGateway-->>Client: Return new order ID
    else Quote invalid
        QuoteService-->>APIGateway: Return 400 Conflict
        APIGateway-->>Client: Show "Invalid quote reference"
    end
```

---

## **2. Shipment Tracking Workflow**
**Purpose:** Allow warehouse teams to log and view shipment status updates.  
**Triggers:** Warehouse manager updates shipment delivery status.  
**Communication Patterns:** Synchronous REST, MongoDB write operations.

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participantShipmentService as ShipmentService
    participantOrderService as OrderService
    participantMongoDB

    Client->>APIGateway: GET /api/shipments?orderId=123
    APIGateway->>ShipmentService: GET /api/shipments?orderId=123
    ShipmentService->>MongoDB: Query shipments by orderId
    ShipmentService-->>APIGateway: Return shipment details
    APIGateway-->>Client: Display shipment status

    Client->>APIGateway: POST /api/shipments (deliveryDate, deliveryAddress)
    APIGateway->>ShipmentService: POST /api/shipments
    ShipmentService->>MongoDB: Insert shipment record
    ShipmentService->>OrderService: PUT /api/orders/123/status (Shipped)
    OrderService->>MongoDB: Update order status to "Shipped"
    OrderService->>MongoDB: Log event (Order Shipped)
    OrderService-->>APIGateway: Acknowledge status update
    APIGateway-->>Client: Confirm shipment status
```

---

## **3. Event-Driven Order Status Update**
**Purpose:** Propagate shipment events to update order status asynchronously.  
**Triggers:** Delivery service publishes event when truck is dispatched.  
**Communication Patterns:** Event-driven (Kafka/RabbitMQ), MongoDB updates.

```mermaid
sequenceDiagram
    participant EventLogger
    participantEventBus as EventBus
    participantOrderService as OrderService
    participantMongoDB

    EventLogger->>EventBus: Publish event "Order Shipped"
    EventBus->>OrderService: Listen for "Order Shipped" topic
    OrderService->>MongoDB: Update order status to "Shipped"
    OrderService->>MongoDB: Insert event (Order Shipped)
    OrderService-->>EventBus: Acknowledge message
```

---

## **4. Dealer Registration Workflow**
**Purpose:** Allow partners to register new dealer information.  
**Triggers:** Partner submits dealer form on frontend.  
**Communication Patterns:** REST API, MongoDB transaction.

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participantDealerService as DealerService
    participantMongoDB

    Client->>APIGateway: POST /api/dealers (name, address)
    APIGateway->>DealerService: POST /api/dealers
    DealerService->>MongoDB: Insert dealer record
    DealerService-->>APIGateway: Return dealer ID
    APIGateway-->>Client: Confirm registration
```

---

## **5. Error Handling for Duplicate Order Creation**
**Purpose:** Prevent accidental duplicate orders using the same quote.  
**Triggers:** User attempts to reuse a quote ID (e.g., spam submission).  
**Communication Patterns:** REST error responses.

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participantOrderService as OrderService
    participantMongoDB

    Client->>APIGateway: POST /api/orders (quoteId: "ABC123")
    APIGateway->>OrderService: POST /api/orders
    OrderService->>MongoDB: Query orders by quoteId
    alt Order already exists
        OrderService-->>APIGateway: Return 409 Conflict
        APIGateway-->>Client: Show "Order already exists"
    else No conflict
        OrderService->>MongoDB: Insert new order
        OrderService-->>APIGateway: Return success
        APIGateway-->>Client: Show "Order created"
    end
```

---

## **6. Background Order Validation Task**
**Purpose:** Clean up partial orders every midnight.  
**Triggers:** Cron job at 00:00.  
**Communication Patterns:** Synchronous calls to OrderService via scheduler.

```mermaid
sequenceDiagram
    participant Scheduler
    participantOrderService as OrderService
    participantMongoDB

    Scheduler->>OrderService: GET /api/orders/stale
    OrderService->>MongoDB: Query orders with no status update > 24h
    loop for each stale order
        OrderService->>MongoDB: Delete order
        OrderService->>MongoDB: Log event (Order Expired)
    end
```

---

## **7. Cross-Service Quote-Inventory Check**
**Purpose:** Ensure quote items are valid before order creation.  
**Triggers:** Order creation request.  
**Communication Patterns:** REST calls + MongoDB.

```mermaid
sequenceDiagram
    participantOrderService as OrderService
    participantQuoteService as QuoteService
    participantCatalogService as CatalogService
    participantMongoDB

    OrderService->>QuoteService: GET /api/quotes/{id}
    QuoteService->>MongoDB: Retrieve quote items
    loop for each item in quote
        QuoteService->>CatalogService: GET /api/catalog/{skuNumber}
        CatalogService->>MongoDB: Check inventory and lead time
        alt Inventory available
            CatalogService-->>QuoteService: Confirm valid
        else Low/No stock
            CatalogService-->>QuoteService: Return error
            QuoteService-->>OrderService: Return 400 Conflict
        end
    end
    QuoteService-->>OrderService: Return quote validation result
    OrderService->>MongoDB: Add order record
```

---

## **8. Shipment Address Validation (External API)**
**Purpose:** Ensure all delivery addresses are valid.  
**Triggers:** Shipment request submission.  
**Communication Patterns:** REST to internal services + external HTTP call.

```mermaid
sequenceDiagram
    participantShipmentService as ShipmentService
    participantExternalAPI
    participantMongoDB

    ShipmentService->>ExternalAPI: POST /geocoding (address details)
    ExternalAPI-->>ShipmentService: Return API response
    alt Address valid
        ShipmentService->>MongoDB: Insert shipment record
    else Address invalid
        ShipmentService->>MongoDB: Log error event
        ShipmentService-->>Client: Return 400 Address Invalid
    end
```

---

## **9. Order Event Audit Trail Update**
**Purpose:** Record all changes to an order for accountability.  
**Triggers:** Status update or manual event logging.  
**Communication Patterns:** Synchronous REST endpoint.

```mermaid
sequenceDiagram
    participantOrderService as OrderService
    participantMongoDB

    OrderService->>MongoDB: POST /api/orders/{id}/events (message: "Manual shipment delay request")
    MongoDB-->>OrderService: Acknowledge event write
```

---

## **10. Frontend Dealer Search Workflow**
**Purpose:** Allow users to search dealer information.  
**Triggers:** User types dealer name in search input.  
**Communication Patterns:** REST with pagination and filtering.

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participantDealerService as DealerService
    participantMongoDB

    Client->>APIGateway: GET /api/dealers?query=Electr
    APIGateway->>DealerService: GET /api/dealers
    DealerService->>MongoDB: Query dealers by name partial match
    DealerService-->>APIGateway: Return paginated dealer list
    APIGateway-->>Client: Render dealer results
```

---

## Summary of Communication Patterns
| **Pattern**            | **Use Cases**                                                                   |
|------------------------|---------------------------------------------------------------------------------|
| REST API               | Order/Quote creation, dealer registration, shipment status updates              |
| MongoDB Transactions   | CRUD operations for order, quote, and dealer data                               |
| Event Bus (MQ)         | Async status updates (e.g., shipment events)                                    |
| External HTTP Calls    | Address validation via geocoding APIs                                           |
| Cron Jobs              | Nightly cleanup of stale orders (system task)                                   |