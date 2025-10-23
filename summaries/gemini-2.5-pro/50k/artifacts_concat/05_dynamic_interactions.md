### 1. User Workflow: Create Quote and Convert to Order

This workflow describes the primary business process where a user, likely a sales representative, creates a price quote for a customer and then converts that approved quote into a formal order.

-   **Trigger:** User action in the Web UI.
-   **Communication Patterns:**
    -   Synchronous RESTful API calls (HTTP GET, POST) from the `Web UI` to the `OrderService`.
    -   Internal database transactions within the `OrderService` to ensure atomicity when writing to `MongoDB`.

```mermaid
sequenceDiagram
    participant User
    participant Web UI (SPA)
    participant OrderService
    participant MongoDB

    User->>Web UI (SPA): Navigates to create a new quote.
    Web UI (SPA)->>OrderService: GET /dealers
    OrderService->>MongoDB: db.dealers.find()
    MongoDB-->>OrderService: Dealer list
    OrderService-->>Web UI (SPA): 200 OK (Dealer list)

    Web UI (SPA)->>OrderService: GET /catalog
    OrderService->>MongoDB: db.catalog.find()
    MongoDB-->>OrderService: Catalog list
    OrderService-->>Web UI (SPA): 200 OK (Catalog list)

    User->>Web UI (SPA): Fills quote details and adds items from catalog.
    User->>Web UI (SPA): Clicks "Save Quote".
    Web UI (SPA)->>OrderService: POST /quotes (Quote JSON payload)

    activate OrderService
    OrderService->>OrderService: Validate quote data.
    alt If dealer in quote does not exist
        OrderService->>MongoDB: db.dealers.findAndModify({ name: ... }, { $setOnInsert: ... }, { upsert: true })
    end
    OrderService->>MongoDB: START_TRANSACTION
    OrderService->>MongoDB: db.quotes.insertOne(quote)
    OrderService->>MongoDB: COMMIT_TRANSACTION
    MongoDB-->>OrderService: New Quote document with _id
    OrderService-->>Web UI (SPA): 201 Created (New Quote with quoteId)
    deactivate OrderService

    Web UI (SPA)->>User: Displays "Quote Created Successfully".

    %% --- Convert to Order ---
    User->>Web UI (SPA): Clicks "Create Order from Quote".
    Web UI (SPA)->>OrderService: POST /orders?fromQuote={quoteId}

    activate OrderService
    OrderService->>OrderService: Validate quoteId exists and is not yet used for an order.
    OrderService->>MongoDB: START_TRANSACTION
    OrderService->>MongoDB: db.orders.insertOne({ quoteId: ..., status: 'Created', ... })
    OrderService->>MongoDB: COMMIT_TRANSACTION
    MongoDB-->>OrderService: New Order document with _id
    OrderService-->>Web UI (SPA): 201 Created (New Order with orderId)
    deactivate OrderService

    Web UI (SPA)->>User: Displays "Order Created Successfully".
```

### 2. User Workflow: View Aggregated Delivery Details

This workflow illustrates how the user views detailed information for a single delivery. It highlights a "chatty" communication pattern where the frontend makes multiple, sequential API calls to aggregate data that is logically linked but stored in separate collections/domains.

-   **Trigger:** User clicks on a delivery record in a list.
-   **Communication Patterns:**
    -   Multiple sequential, synchronous RESTful API calls (HTTP GET) from the `Web UI` to the `OrderService`. This is a classic N+1 query problem at the API layer.

```mermaid
sequenceDiagram
    participant User
    participant Web UI (SPA)
    participant OrderService
    participant MongoDB

    User->>Web UI (SPA): Clicks on a delivery to view its details.
    Web UI (SPA)->>User: Navigates to delivery details page, shows loading spinner.

    activate Web UI (SPA)
    %% Step 1: Get the Shipment/Delivery record
    Web UI (SPA)->>OrderService: GET /shipments/{orderId}
    activate OrderService
    OrderService->>MongoDB: db.shipments.findOne({ orderId: ... })
    MongoDB-->>OrderService: Shipment document
    OrderService-->>Web UI (SPA): 200 OK (Shipment data)
    deactivate OrderService
    Web UI (SPA)->>Web UI (SPA): Renders basic shipment info.

    %% Step 2: Get the associated Order record
    note over Web UI (SPA): Shipment data has orderId, but I need order status and date.
    Web UI (SPA)->>OrderService: GET /orders/{orderId}
    activate OrderService
    OrderService->>MongoDB: db.orders.findOne({ orderId: ... })
    MongoDB-->>OrderService: Order document (contains quoteId)
    OrderService-->>Web UI (SPA): 200 OK (Order data)
    deactivate OrderService
    Web UI (SPA)->>Web UI (SPA): Renders order info (status, date).

    %% Step 3: Get the associated Quote record
    note over Web UI (SPA): Order data has quoteId, but I need customer details.
    Web UI (SPA)->>OrderService: GET /quotes/{quoteId}
    activate OrderService
    OrderService->>MongoDB: db.quotes.findOne({ quoteId: ... })
    MongoDB-->>OrderService: Quote document
    OrderService-->>Web UI (SPA): 200 OK (Quote data)
    deactivate OrderService
    Web UI (SPA)->>Web UI (SPA): Renders quote info (customer name, etc).
    deactivate Web UI (SPA)
    
    Web UI (SPA)->>User: Displays complete, aggregated delivery view.
```

### 3. System Workflow: Process External Order via Message Queue

This workflow describes the asynchronous, backend process for creating an order that originates from an external system (e.g., a public website). The `IntegrationService` acts as an adapter, consuming messages from a queue and using the `OrderService` API to persist the data.

-   **Trigger:** A scheduled task in the `IntegrationService` polls a message queue.
-   **Communication Patterns:**
    -   Asynchronous message queue (polling consumer pattern) between `External System` and `IntegrationService`.
    -   Synchronous RESTful API calls (HTTP POST) from `IntegrationService` to `OrderService`.

```mermaid
sequenceDiagram
    participant External Website
    participant Azure Orders Queue
    participant IntegrationService
    participant OrderService
    participant MongoDB

    External Website->>Azure Orders Queue: Enqueue OrderMessage
    
    loop Scheduled Poll (e.g., every 30s)
        IntegrationService->>Azure Orders Queue: Dequeue Message
        Azure Orders Queue-->>IntegrationService: OrderMessage
        
        opt Message Received
            IntegrationService->>IntegrationService: Transform OrderMessage to Quote model
            IntegrationService->>OrderService: POST /quotes
            activate OrderService
            OrderService->>MongoDB: db.quotes.insertOne(...)
            OrderService-->>IntegrationService: 201 Created (with quoteId)
            deactivate OrderService

            IntegrationService->>IntegrationService: Use quoteId to create Order model
            IntegrationService->>OrderService: POST /orders?fromQuote={quoteId}
            activate OrderService
            OrderService->>MongoDB: db.orders.insertOne(...)
            OrderService-->>IntegrationService: 201 Created (with orderId)
            deactivate OrderService
            
            IntegrationService->>IntegrationService: Use orderId to create Shipment model
            IntegrationService->>OrderService: POST /shipments
            activate OrderService
            OrderService->>MongoDB: db.shipments.insertOne(...)
            OrderService-->>IntegrationService: 201 Created
            deactivate OrderService

            note right of IntegrationService: All API calls successful.
            IntegrationService->>Azure Orders Queue: Delete Message
        end
    end
```

### 4. System Workflow: Synchronize Product Catalog to External System

This workflow shows the `IntegrationService` acting in the reverse direction. It periodically polls the `OrderService` for the current product catalog and pushes updates to a queue for an external system to consume.

-   **Trigger:** A scheduled task in the `IntegrationService`.
-   **Communication Patterns:**
    -   Synchronous RESTful API call (HTTP GET) from `IntegrationService` to `OrderService`.
    -   Asynchronous message queue (producer pattern) from `IntegrationService` to the `External System`.

```mermaid
sequenceDiagram
    participant IntegrationService
    participant OrderService
    participant MongoDB
    participant Azure Product Queue
    participant External Website

    loop Scheduled Poll (e.g., every 30s)
        IntegrationService->>OrderService: GET /catalog
        activate OrderService
        OrderService->>MongoDB: db.catalog.find()
        MongoDB-->>OrderService: List of CatalogItems
        OrderService-->>IntegrationService: 200 OK (CatalogItem list)
        deactivate OrderService

        IntegrationService->>IntegrationService: Transform CatalogItem list into ProductMessage
        IntegrationService->>Azure Product Queue: Enqueue ProductMessage
    end

    %% External system consumes asynchronously
    External Website-->>Azure Product Queue: Dequeue Message
```

### 5. Error Handling: Database Operation with Retry

This sequence diagram details the internal error handling and recovery pattern implemented by the `MongoOperationsWithRetry` decorator within the `OrderService`. It adds resilience to database interactions by automatically retrying on transient network errors.

-   **Trigger:** An internal call to a repository method that performs a database operation.
-   **Communication Patterns:**
    -   Internal method calls within the `OrderService`.
    -   Decorator Pattern wrapping the standard MongoDB client.
    -   Telemetry calls to Application Insights (not shown for brevity).

```mermaid
sequenceDiagram
    participant Controller
    participant Repository
    participant MongoOperationsWithRetry
    participant MongoTemplate
    participant MongoDB

    Controller->>Repository: createOrder(order)
    activate Repository
    Repository->>MongoOperationsWithRetry: save(orderDetails)
    activate MongoOperationsWithRetry

    note over MongoOperationsWithRetry: Start telemetry timer.

    MongoOperationsWithRetry->>MongoTemplate: save(orderDetails)
    activate MongoTemplate
    MongoTemplate->>MongoDB: db.orders.insertOne(...)
    
    %% First attempt fails
    MongoDB-->>MongoTemplate: SocketTimeoutException
    MongoTemplate-->>MongoOperationsWithRetry: throws SocketTimeoutException
    deactivate MongoTemplate
    
    note over MongoOperationsWithRetry: Catch exception, log retry attempt.

    %% Second attempt
    MongoOperationsWithRetry->>MongoTemplate: save(orderDetails)
    activate MongoTemplate
    MongoTemplate->>MongoDB: db.orders.insertOne(...)

    alt Operation Succeeds on Retry
        MongoDB-->>MongoTemplate: Success
        MongoTemplate-->>MongoOperationsWithRetry: return result
        deactivate MongoTemplate
        note over MongoOperationsWithRetry: Stop timer, report success telemetry.
        MongoOperationsWithRetry-->>Repository: return result
        Repository-->>Controller: return orderId
        deactivate Repository
    else Operation Fails Again
        MongoDB-->>MongoTemplate: SocketTimeoutException
        MongoTemplate-->>MongoOperationsWithRetry: throws SocketTimeoutException
        deactivate MongoTemplate
        note over MongoOperationsWithRetry: Stop timer, report failure telemetry.
        MongoOperationsWithRetry-->>Repository: throws DataAccessException
        Repository-->>Controller: throws DataAccessException
        deactivate Repository
        Controller-->>Client: 500 Internal Server Error
    end
    deactivate MongoOperationsWithRetry
```