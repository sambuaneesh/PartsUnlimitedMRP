### 1. User Views Catalog Items

-   **Workflow Purpose and Trigger:** This workflow is triggered when a user navigates to the "Catalog" page in the web UI. Its purpose is to fetch and display the list of all available parts from the system.
-   **Communication Patterns:** This is a synchronous read operation.
    -   **Browser -> Order Service:** A synchronous `GET` REST API call.
    -   **Order Service -> MongoDB:** A synchronous database `find` query.

```mermaid
sequenceDiagram
    participant User
    participant Browser as "Web Client (SPA)"
    participant OrderService as "Order Service"
    participant MongoDB

    User->>Browser: Navigates to Catalog page
    activate Browser
    Browser->>OrderService: GET /catalog
    activate OrderService
    Note right of OrderService: CatalogController.getCatalogItems()
    OrderService->>MongoDB: find("catalog", {})
    activate MongoDB
    MongoDB-->>OrderService: Returns list of CatalogItem documents
    deactivate MongoDB
    OrderService-->>Browser: 200 OK [List of CatalogItems]
    deactivate OrderService
    Browser->>User: Renders list of catalog items
    deactivate Browser
```

### 2. User Creates an Order from a Quote

-   **Workflow Purpose and Trigger:** This workflow converts an existing sales quote into a confirmed order. It's triggered when a user clicks the "Order" button for a selected quote in the UI.
-   **Communication Patterns:** This is a synchronous write operation involving multiple validation steps.
    -   **Browser -> Order Service:** A synchronous `POST` REST API call.
    -   **Order Service -> MongoDB:** Multiple synchronous database `findOne` and `insert` queries to validate the quote and create the new order record.

```mermaid
sequenceDiagram
    participant User
    participant Browser as "Web Client (SPA)"
    participant OrderService as "Order Service"
    participant MongoDB

    User->>Browser: Clicks 'Order' on a selected Quote
    activate Browser
    Browser->>OrderService: POST /orders?fromQuote={quoteId}
    activate OrderService
    Note right of OrderService: OrderController.createOrder()
    
    OrderService->>MongoDB: 1. findOne("quotes", {quoteId})
    activate MongoDB
    MongoDB-->>OrderService: Returns QuoteDetails
    deactivate MongoDB

    alt Quote found and not already ordered
        OrderService->>MongoDB: 2. findOne("orders", {quoteId})
        activate MongoDB
        MongoDB-->>OrderService: Returns null
        deactivate MongoDB

        Note right of OrderService: Create new Order object with status 'Created'
        OrderService->>MongoDB: 3. insert("orders", {orderDetails})
        activate MongoDB
        MongoDB-->>OrderService: Insert Acknowledgment
        deactivate MongoDB

        OrderService-->>Browser: 201 CREATED [Location: /orders/{newId}, Order JSON]
    else Quote not found or already ordered
        OrderService-->>Browser: 400 BAD_REQUEST or 409 CONFLICT
    end
    deactivate OrderService
    Browser->>User: Navigates to new Order or displays error
    deactivate Browser
```

### 3. User Updates an Order's Status

-   **Workflow Purpose and Trigger:** To update the status of an existing order and log an event for the change. This is triggered when a user changes the status dropdown in the Order details view and saves.
-   **Communication Patterns:** This is a synchronous update operation.
    -   **Browser -> Order Service:** A synchronous `PUT` REST API call.
    -   **Order Service -> MongoDB:** A synchronous `findOne` and `save` (update) operation.

```mermaid
sequenceDiagram
    participant User
    participant Browser as "Web Client (SPA)"
    participant OrderService as "Order Service"
    participant MongoDB

    User->>Browser: Changes order status and saves
    activate Browser
    Browser->>OrderService: PUT /orders/{orderId}/status (with status & event info)
    activate OrderService
    Note right of OrderService: OrderController.updateStatus()

    OrderService->>MongoDB: 1. findOne("orders", {orderId})
    activate MongoDB
    MongoDB-->>OrderService: Returns OrderDetails
    deactivate MongoDB

    alt Order exists
        Note right of OrderService: Update status and add new event to the order object
        OrderService->>MongoDB: 2. save("orders", {updatedOrderDetails})
        activate MongoDB
        MongoDB-->>OrderService: Update Acknowledgment
        deactivate MongoDB
        OrderService-->>Browser: 200 OK
    else Order does not exist
        OrderService-->>Browser: 400 BAD_REQUEST
    end
    deactivate OrderService

    Browser->>User: Displays confirmation
    deactivate Browser
```

### 4. Asynchronous Order Creation (External Website Integration)

-   **Workflow Purpose and Trigger:** This workflow processes new orders from an external system (e.g., the main Parts Unlimited website). It is triggered by a periodic, scheduled task in the `IntegrationService` that polls an Azure Storage Queue. This decouples the MRP system from the external website.
-   **Communication Patterns:**
    -   **Integration Service -> Azure Queue:** Asynchronous message polling.
    -   **Integration Service -> Order Service:** A sequence of synchronous `POST` REST API calls to create the quote, order, and shipment.
    -   **Order Service -> MongoDB:** Synchronous database inserts for each REST call.
    -   **Integration Service -> Azure Queue:** Asynchronous message deletion upon success.

```mermaid
sequenceDiagram
    participant Website as "External Website"
    participant AzureQueueOrders as "Azure 'orders' Queue"
    participant IntegrationService
    participant OrderService
    participant MongoDB

    Website->>AzureQueueOrders: Enqueue OrderMessage
    
    loop Periodically (every 30s)
        IntegrationService->>AzureQueueOrders: retrieveMessage()
        activate IntegrationService
        AzureQueueOrders-->>IntegrationService: Returns OrderMessage (if available)

        alt Message found
            Note over IntegrationService: Process new order from Website via MrpConnectService.createNewOrder()
            IntegrationService->>OrderService: 1. POST /quotes (with translated OrderMessage)
            activate OrderService
            OrderService->>MongoDB: insert("quotes", ...)
            MongoDB-->>OrderService: Ack
            OrderService-->>IntegrationService: 201 CREATED [Quote with quoteId]
            deactivate OrderService

            IntegrationService->>OrderService: 2. POST /orders?fromQuote={quoteId}
            activate OrderService
            OrderService->>MongoDB: insert("orders", ...)
            MongoDB-->>OrderService: Ack
            OrderService-->>IntegrationService: 201 CREATED [Order with orderId]
            deactivate OrderService

            IntegrationService->>OrderService: 3. POST /shipments (with shipment record)
            activate OrderService
            OrderService->>MongoDB: insert("shipments", ...)
            MongoDB-->>OrderService: Ack
            OrderService-->>IntegrationService: 201 CREATED
            deactivate OrderService

            Note over IntegrationService: Order processed successfully
            IntegrationService->>AzureQueueOrders: deleteQueueMessage()
        end
        deactivate IntegrationService
    end
```

### 5. Asynchronous Inventory Update (External Website Integration)

-   **Workflow Purpose and Trigger:** This workflow periodically provides inventory level updates to an external system. It's triggered by a scheduled task in the `IntegrationService` that queries the `OrderService` for the latest catalog data and pushes it to an Azure Storage Queue.
-   **Communication Patterns:**
    -   **Integration Service -> Order Service:** A synchronous `GET` REST API call.
    -   **Order Service -> MongoDB:** A synchronous `find` query.
    -   **Integration Service -> Azure Queue:** An asynchronous message push.

```mermaid
sequenceDiagram
    participant IntegrationService
    participant OrderService
    participant MongoDB
    participant AzureQueueInv as "Azure 'inventory' Queue"
    participant Website as "External Website"

    loop Periodically (every 30s)
        activate IntegrationService
        Note over IntegrationService: UpdateProductProcessTask runs
        IntegrationService->>OrderService: GET /catalog
        activate OrderService
        OrderService->>MongoDB: find("catalog", {})
        MongoDB-->>OrderService: Returns list of CatalogItems
        OrderService-->>IntegrationService: 200 OK [List of CatalogItems]
        deactivate OrderService

        alt Catalog items found
            Note over IntegrationService: Transforms data to ProductMessage
            IntegrationService->>AzureQueueInv: Enqueue ProductMessage
        end
        deactivate IntegrationService
    end
    
    Website->>AzureQueueInv: Periodically polls for inventory updates
```

### 6. Error Handling Pattern: Database Operation Retry

-   **Workflow Purpose and Trigger:** To transparently handle transient database connection issues. The `MongoOperationsWithRetry` class wraps database calls from the `OrderService` and implements a retry mechanism specifically for socket timeouts. This enhances system resilience.
-   **Communication Patterns:** Internal synchronous method calls within the `OrderService` and interaction with the database driver.

```mermaid
sequenceDiagram
    participant Repository as "e.g., MongoOrderRepository"
    participant MongoOperationsWithRetry
    participant MongoTemplate
    participant MongoDB

    Repository->>MongoOperationsWithRetry: findOne(query, entity)
    activate MongoOperationsWithRetry

    note right of MongoOperationsWithRetry: First attempt
    MongoOperationsWithRetry->>MongoTemplate: findOne(query, entity)
    activate MongoTemplate
    MongoTemplate->>MongoDB: Execute find query
    
    alt SocketTimeoutException occurs
        MongoDB--xMongoTemplate: Timeout
        MongoTemplate--xMongoOperationsWithRetry: DataAccessResourceFailureException
        deactivate MongoTemplate

        note right of MongoOperationsWithRetry: Catch exception and retry operation
        MongoOperationsWithRetry->>MongoTemplate: findOne(query, entity)
        activate MongoTemplate
        MongoTemplate->>MongoDB: Execute find query (2nd attempt)
        MongoDB-->>MongoTemplate: Returns document
        MongoTemplate-->>MongoOperationsWithRetry: Returns document
        deactivate MongoTemplate
    else Success on first attempt
        MongoDB-->>MongoTemplate: Returns document
        MongoTemplate-->>MongoOperationsWithRetry: Returns document
        deactivate MongoTemplate
    end

    note right of MongoOperationsWithRetry: Track dependency call in Application Insights
    MongoOperationsWithRetry->>MongoOperationsWithRetry: sendTelemetry()

    MongoOperationsWithRetry-->>Repository: Returns document
    deactivate MongoOperationsWithRetry
```