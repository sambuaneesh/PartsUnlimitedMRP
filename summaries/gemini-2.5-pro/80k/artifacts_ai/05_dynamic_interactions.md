### Workflow 1: User Creates a Sales Quote

This workflow describes a user creating a new sales quote. The process involves selecting a dealer, adding items from the catalog, and saving the quote. This highlights the frontend's role in orchestrating data from multiple API endpoints to build a single user view.

**Communication Patterns:**
*   **Client-Server:** Synchronous REST/HTTP (AJAX) calls.
*   **Data Access:** Direct synchronous reads/writes to MongoDB.
*   **Orchestration:** The Web Frontend acts as an orchestrator, fetching data from `/dealers` and `/catalog` to provide choices for the user before submitting the final quote to `/quotes`.

```mermaid
sequenceDiagram
    participant User
    participant Web Frontend (SPA)
    participant OrderService (Monolith)
    participant MongoDB

    User->>Web Frontend (SPA): Navigates to create a new quote
    Web Frontend (SPA)->>OrderService (Monolith): GET /dealers
    OrderService (Monolith)->>MongoDB: find(dealers, {})
    MongoDB-->>OrderService (Monolith): Returns list of dealers
    OrderService (Monolith)-->>Web Frontend (SPA): 200 OK [Dealer List]
    Web Frontend (SPA)->>User: Renders quote form with dealer dropdown

    User->>Web Frontend (SPA): Fills customer name, selects dealer
    User->>Web Frontend (SPA): Clicks "Add Items"
    
    activate Web Frontend (SPA)
    Note over Web Frontend (SPA): "Extras" popup component is shown.
    Web Frontend (SPA)->>OrderService (Monolith): GET /catalog
    OrderService (Monolith)->>MongoDB: find(catalog, {})
    MongoDB-->>OrderService (Monolith): Returns list of catalog items
    OrderService (Monolith)-->>Web Frontend (SPA): 200 OK [Catalog List]
    Web Frontend (SPA)->>User: Displays catalog items for selection
    User->>Web Frontend (SPA): Selects items and quantities
    User->>Web Frontend (SPA): Clicks "Save" on popup
    deactivate Web Frontend (SPA)
    
    Web Frontend (SPA)->>User: Renders selected items in quote form
    User->>Web Frontend (SPA): Clicks "Save Quote"
    Web Frontend (SPA)->>OrderService (Monolith): POST /quotes [QuoteData]
    
    activate OrderService (Monolith)
    OrderService (Monolith)->>OrderService (Monolith): model.validate()
    OrderService (Monolith)->>MongoDB: save(quotes, newQuote)
    MongoDB-->>OrderService (Monolith): Returns saved document with _id
    OrderService (Monolith)-->>Web Frontend (SPA): 201 Created [New Quote with ID]
    deactivate OrderService (Monolith)
    
    Web Frontend (SPA)->>User: Displays success message and new quote
```

### Workflow 2: User Converts a Quote to an Order

This workflow shows the critical business transition from a non-binding quote to a committed order. The frontend navigates the user through the process, triggering a specific backend endpoint that encapsulates the creation logic.

**Communication Patterns:**
*   **Client-Server:** Synchronous REST/HTTP calls.
*   **Data Access:** The `OrderService` reads from the `quotes` collection and writes to the `orders` collection in a single business transaction.
*   **State Transition:** A `POST` to `/orders?fromQuote={quoteId}` represents a key state transition in the system's core workflow.

```mermaid
sequenceDiagram
    participant User
    participant Web Frontend (SPA)
    participant OrderService (Monolith)
    participant MongoDB

    User->>Web Frontend (SPA): Selects an existing quote to view
    Web Frontend (SPA)->>OrderService (Monolith): GET /quotes/{quoteId}
    OrderService (Monolith)-->>Web Frontend (SPA): 200 OK [QuoteDetails]
    Web Frontend (SPA)->>User: Displays quote details

    User->>Web Frontend (SPA): Clicks "Convert to Order"
    Note over Web Frontend (SPA): SPA navigates to order page, retaining quote ID.
    
    Web Frontend (SPA)->>OrderService (Monolith): POST /orders?fromQuote={quoteId}
    activate OrderService (Monolith)
    
    OrderService (Monolith)->>MongoDB: findOne(quotes, {quoteId: quoteId})
    MongoDB-->>OrderService (Monolith): Returns QuoteDetails document
    
    alt Quote found and valid
        OrderService (Monolith)->>OrderService (Monolith): Create new OrderDetails from QuoteDetails
        Note right of OrderService (Monolith): Business logic to check if quote was already used.
        OrderService (Monolith)->>MongoDB: save(orders, newOrder)
        MongoDB-->>OrderService (Monolith): Returns saved Order document
        OrderService (Monolith)-->>Web Frontend (SPA): 201 Created [New Order with ID]
    else Quote not found or already used
        OrderService (Monolith)-->>Web Frontend (SPA): 4xx Error (e.g., 404 Not Found or 409 Conflict)
    end
    
    deactivate OrderService (Monolith)
    Web Frontend (SPA)->>User: Displays newly created order or an error message
```

### Workflow 3: User Ships an Order (Creates a Delivery)

This workflow captures the final stage of fulfillment. Creating a shipment not only adds a new record but also implicitly changes the state of the associated order. The backend service should handle this state change atomically.

**Communication Patterns:**
*   **Client-Server:** Synchronous REST/HTTP calls.
*   **Data Access:** Multiple database writes (`shipments` and `orders` collections) that should be treated as a single logical transaction.
*   **Error Handling:** If updating the order status fails after creating the shipment, the system could be left in an inconsistent state, highlighting a weakness of the monolithic, non-transactional approach across operations.

```mermaid
sequenceDiagram
    participant User
    participant Web Frontend (SPA)
    participant OrderService (Monolith)
    participant MongoDB

    User->>Web Frontend (SPA): Views an existing order and clicks "Deliver"
    Note over Web Frontend (SPA): User is navigated to the delivery creation form.
    User->>Web Frontend (SPA): Fills in delivery address and contact info
    User->>Web Frontend (SPA): Clicks "Save Delivery"
    
    Web Frontend (SPA)->>OrderService (Monolith): POST /shipments [ShipmentData with orderId]
    activate OrderService (Monolith)
    
    OrderService (Monolith)->>OrderService (Monolith): Create new ShipmentDetails object
    OrderService (Monolith)->>MongoDB: save(shipments, newShipment)
    MongoDB-->>OrderService (Monolith): Acknowledges save
    
    Note right of OrderService (Monolith): The service now updates the related order's status.
    OrderService (Monolith)->>MongoDB: findOneAndUpdate(orders, {orderId: orderId}, {$set: {status: 'Shipped'}})
    MongoDB-->>OrderService (Monolith): Acknowledges update
    
    OrderService (Monolith)-->>Web Frontend (SPA): 201 Created [New Shipment]
    deactivate OrderService (Monolith)
    
    Web Frontend (SPA)->>User: Displays success and navigates to the delivery record
```

### Workflow 4: Viewing a Composite Delivery Record (Client-Side Join)

This read-only workflow is critical for understanding the system's performance and data coupling characteristics. It demonstrates a "chatty" communication pattern where the client is responsible for making multiple, sequential API calls and joining the data, a common issue in monolithic APIs consumed by SPAs.

**Communication Patterns:**
*   **Client-Server:** Multiple, sequential synchronous REST/HTTP calls. This is an inefficient, "chatty" pattern.
*   **Data Aggregation:** The **Web Frontend (SPA)** performs a "client-side join" of data from three separate entities (Shipment, Order, Quote).
*   **Design Flaw:** This pattern increases latency and couples the client tightly to the backend's data model. A microservices architecture would address this with an API Gateway or BFF.

```mermaid
sequenceDiagram
    participant User
    participant Web Frontend (SPA)
    participant OrderService (Monolith)
    participant MongoDB

    User->>Web Frontend (SPA): Selects a delivery record to view
    
    activate Web Frontend (SPA)
    Note over Web Frontend (SPA): Begin chained API calls for data aggregation.

    Web Frontend (SPA)->>OrderService (Monolith): 1. GET /shipments/{orderId}
    OrderService (Monolith)->>MongoDB: findOne(shipments, {orderId: orderId})
    MongoDB-->>OrderService (Monolith): ShipmentDetails
    OrderService (Monolith)-->>Web Frontend (SPA): 200 OK [ShipmentDetails]
    
    Note over Web Frontend (SPA): SPA extracts orderId from shipment response.

    Web Frontend (SPA)->>OrderService (Monolith): 2. GET /orders/{orderId}
    OrderService (Monolith)->>MongoDB: findOne(orders, {orderId: orderId})
    MongoDB-->>OrderService (Monolith): OrderDetails
    OrderService (Monolith)-->>Web Frontend (SPA): 200 OK [OrderDetails]
    
    Note over Web Frontend (SPA): SPA extracts quoteId from order response.
    
    Web Frontend (SPA)->>OrderService (Monolith): 3. GET /quotes/{quoteId}
    OrderService (Monolith)->>MongoDB: findOne(quotes, {quoteId: quoteId})
    MongoDB-->>OrderService (Monolith): QuoteDetails
    OrderService (Monolith)-->>Web Frontend (SPA): 200 OK [QuoteDetails]
    
    Note over Web Frontend (SPA): SPA now has all data. It performs a client-side join.
    Web Frontend (SPA)->>User: Renders the complete, composite delivery view.
    deactivate Web Frontend (SPA)
```

### Workflow 5: Asynchronous Order Ingestion from External System

This workflow demonstrates the system's asynchronous integration capabilities. The `IntegrationService` acts as an adapter, polling a message queue and orchestrating a series of REST calls to the `OrderService` to create a complete sales record.

**Communication Patterns:**
*   **Asynchronous Polling:** The `IntegrationService` uses a scheduled task to poll an Azure Storage Queue. This decouples it from the message producer.
*   **Adapter/Orchestration:** The `IntegrationService` orchestrates synchronous REST calls (`POST /quotes`, `POST /orders`, `POST /shipments`) to translate the asynchronous event into a transactional workflow within the `OrderService`.
*   **Event-Driven:** The workflow is triggered by an external event (a message appearing on the queue).

```mermaid
sequenceDiagram
    participant External System
    participant Azure Storage Queue ("orders" queue)
    participant IntegrationService
    participant OrderService (Monolith)
    participant MongoDB

    External System->>Azure Storage Queue: Enqueues NewOrderMessage
    
    loop Every 30 seconds
        IntegrationService->>Azure Storage Queue: Dequeue Message
    end

    Azure Storage Queue-->>IntegrationService: Returns NewOrderMessage
    activate IntegrationService
    
    Note over IntegrationService: Transform message into MRP entities. Begin orchestration.
    
    IntegrationService->>OrderService (Monolith): 1. POST /quotes [QuoteData]
    OrderService (Monolith)->>MongoDB: save(quotes, newQuote)
    OrderService (Monolith)-->>IntegrationService: 201 Created [Quote with ID]
    
    IntegrationService->>OrderService (Monolith): 2. POST /orders?fromQuote={quoteId}
    OrderService (Monolith)->>MongoDB: save(orders, newOrder)
    OrderService (Monolith)-->>IntegrationService: 201 Created [Order with ID]
    
    IntegrationService->>OrderService (Monolith): 3. POST /shipments [ShipmentData]
    OrderService (Monolith)->>MongoDB: save(shipments, newShipment)
    OrderService (Monolith)-->>IntegrationService: 201 Created [Shipment]
    
    alt All steps successful
        IntegrationService->>Azure Storage Queue: Delete Message
    else A step failed
        Note over IntegrationService: Message remains on queue for retry. Error is logged.
    end
    
    deactivate IntegrationService
```

### Workflow 6: Asynchronous Catalog Update to External System

This workflow shows the outbound integration path. The `IntegrationService` polls the `OrderService` for data and pushes updates to an external system via a message queue, acting as an event publisher.

**Communication Patterns:**
*   **Asynchronous Publishing:** The `IntegrationService` pushes messages to an Azure Storage Queue, decoupling it from consumers.
*   **Scheduled Polling:** A scheduled task periodically fetches data, turning a state-based system (the catalog in the DB) into an event-based one (messages on a queue). This is a common pattern for integrating with legacy systems.
*   **Data Source:** The `OrderService`'s REST API serves as the single source of truth for the `IntegrationService`.

```mermaid
sequenceDiagram
    participant IntegrationService
    participant OrderService (Monolith)
    participant MongoDB
    participant Azure Storage Queue ("product" queue)
    participant External System

    loop Every 30 seconds
        IntegrationService->>OrderService (Monolith): GET /catalog
        OrderService (Monolith)->>MongoDB: find(catalog, {})
        MongoDB-->>OrderService (Monolith): CatalogItem list
        OrderService (Monolith)-->>IntegrationService: 200 OK [Catalog List]
        
        activate IntegrationService
        Note over IntegrationService: Compares with previous state to find updates (not detailed in summary).
        
        loop For each updated product
            IntegrationService->>Azure Storage Queue: Enqueue ProductUpdateMessage
        end
        deactivate IntegrationService
    end
    
    Note over External System: External system independently polls the queue.
    External System->>Azure Storage Queue: Dequeue Message
    Azure Storage Queue-->>External System: ProductUpdateMessage
```