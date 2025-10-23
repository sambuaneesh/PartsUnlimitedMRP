### **1. User Creates a New Sales Quote**

-   **Workflow Purpose & Trigger:** A user, likely a sales representative, creates a new sales quote for a customer. The process is initiated by filling out a form in the Web UI and saving it. A key behavior is the idempotent creation of a new dealer if the one specified in the quote does not already exist.
-   **Communication Patterns:**
    -   **Web UI to Order Service:** Synchronous REST call (`POST /quotes`) using `WinJS.xhr`.
    -   **Order Service to Database:** Synchronous database writes using the Repository pattern. This involves an "application-side join" where the `QuoteRepository` first queries the `DealersRepository`.

```mermaid
sequenceDiagram
    actor User
    participant WebUI as Web UI (SPA)
    participant OrderSvc as Order Service
    participant QuoteRepo as QuoteRepository
    participant DealerRepo as DealersRepository
    participant DBLayer as DB Driver (MongoOperationsWithRetry)
    participant MongoDB

    User->>+WebUI: Fills out Quote form (Customer, Dealer, Items) and Clicks "Save"
    WebUI->>+OrderSvc: POST /quotes (with Quote JSON payload)
    Note over WebUI,OrderSvc: Synchronous API call via WinJS.xhr

    OrderSvc->>+QuoteRepo: createQuote(quoteData)
    QuoteRepo->>+DealerRepo: findByName(quoteData.dealerName)
    Note right of QuoteRepo: Check if Dealer exists before creating Quote.
    DealerRepo->>+DBLayer: findOne("dealers", {name: "..."})
    DBLayer-->>-DealerRepo: Dealer not found (null)

    opt If Dealer does not exist
        DealerRepo->>+DBLayer: insert("dealers", newDealer)
        Note right of DealerRepo: Idempotent dealer creation.
        DBLayer-->>-DealerRepo: New Dealer record
    end
    QuoteRepo-->>-OrderSvc: (Dealer is now guaranteed to exist)

    OrderSvc->>+QuoteRepo: (proceeds with quote creation)
    QuoteRepo->>+DBLayer: insert("quotes", quoteDetails)
    DBLayer->>+MongoDB: write(quoteDetails)
    DBLayer->>AppInsights: TrackDependency("MongoDB", "insert")
    MongoDB-->>-DBLayer: Success
    DBLayer-->>-QuoteRepo: Saved Quote
    QuoteRepo-->>-OrderSvc: Saved Quote with ID
    OrderSvc-->>-WebUI: 201 Created (with new Quote object)
    WebUI-->>-User: Displays success message and new Quote ID
```

### **2. User Converts a Quote to an Order**

-   **Workflow Purpose & Trigger:** After a quote is accepted, a user converts it into a firm order. This is a critical business transaction initiated from the quote details page in the Web UI. The system must ensure that a single quote can only be converted into an order once.
-   **Communication Patterns:**
    -   **Web UI to Order Service:** Synchronous REST call (`POST /orders`).
    -   **Order Service to Database:** Synchronous database reads and writes. This workflow demonstrates internal data coupling, where the `OrderRepository` must validate data with the `QuoteRepository`.
    -   **Error Handling:** A specific business exception (`ConflictingRequestException`) is thrown for rule violations.

```mermaid
sequenceDiagram
    actor User
    participant WebUI as Web UI (SPA)
    participant OrderSvc as Order Service
    participant OrderRepo as OrderRepository
    participant QuoteRepo as QuoteRepository
    participant DBLayer as DB Driver (MongoOperationsWithRetry)
    participant MongoDB

    User->>+WebUI: Opens existing Quote and Clicks "Create Order"
    WebUI->>+OrderSvc: POST /orders?fromQuote={quoteId}
    Note over WebUI,OrderSvc: Synchronous API call

    OrderSvc->>+OrderRepo: createOrderFromQuote(quoteId)

    OrderRepo->>+QuoteRepo: findById(quoteId)
    QuoteRepo->>+DBLayer: findOne("quotes", {quoteId: "..."})
    DBLayer-->>-QuoteRepo: Quote Details
    QuoteRepo-->>-OrderRepo: Returns valid Quote

    OrderRepo->>+DBLayer: findOne("orders", {quoteId: "..."})
    Note right of OrderRepo: Validate that quote has not been used before.
    DBLayer-->>-OrderRepo: Query result (order found or not)

    alt Quote already used to create an Order
        OrderRepo-->>-OrderSvc: Throws ConflictingRequestException
        OrderSvc-->>-WebUI: 409 Conflict
        WebUI-->>-User: Displays "Quote already used" error message
    else Quote is valid and unused
        OrderRepo->>+DBLayer: insert("orders", newOrderDetails)
        DBLayer->>+MongoDB: write(newOrderDetails)
        MongoDB-->>-DBLayer: Success
        DBLayer-->>-OrderRepo: Saved Order
        OrderRepo-->>-OrderSvc: Saved Order with ID
        OrderSvc-->>-WebUI: 201 Created (with new Order object)
        WebUI-->>-User: Displays success and navigates to the new Order
    end
```

### **3. Chatty UI Data Loading for Order List**

-   **Workflow Purpose & Trigger:** This diagram illustrates a potential performance issue where the UI makes multiple, sequential API calls to render a single list of items. It is triggered when a user navigates to the "Orders" page. This "N+1" query pattern is common in systems that haven't been optimized for list views.
-   **Communication Patterns:**
    -   **Web UI to Order Service:** Multiple synchronous REST calls (`GET /orders`, `GET /quotes/{id}`, `GET /dealers/{name}`).

```mermaid
sequenceDiagram
    actor User
    participant WebUI as Web UI (SPA)
    participant OrderSvc as Order Service

    User->>+WebUI: Navigates to the Orders page
    WebUI->>+OrderSvc: GET /orders
    Note over WebUI,OrderSvc: Initial call to get the list of orders
    OrderSvc-->>-WebUI: [Order1, Order2, Order3, ...]

    loop For each order in the list
        WebUI->>+OrderSvc: GET /quotes/{order.quoteId}
        Note right of WebUI: Fetches associated quote to display its details
        OrderSvc-->>-WebUI: QuoteDetails

        WebUI->>+OrderSvc: GET /dealers/{quote.dealerName}
        Note right of WebUI: Fetches associated dealer to display name/address
        OrderSvc-->>-WebUI: DealerDetails
    end

    WebUI-->>-User: Renders the complete list of orders with all details
```

### **4. Asynchronous Order Ingestion via Integration Service**

-   **Workflow Purpose & Trigger:** The system automatically ingests new orders from an external system (e.g., a public website) without user intervention. The `IntegrationService` polls an Azure Storage Queue for new order messages on a fixed schedule.
-   **Communication Patterns:**
    -   **External System to Queue:** Asynchronous message publishing (not shown, but implied).
    -   **Integration Service to Queue:** Asynchronous message polling.
    -   **Integration Service to Order Service:** A series of synchronous REST calls to create the corresponding entities (Quote, Order, Shipment) in the MRP system.

```mermaid
sequenceDiagram
    participant Scheduler
    participant IntSvc as Integration Service
    participant AzureQueue as Azure Storage Queue (orders)
    participant OrderSvc as Order Service
    participant MongoDB

    Scheduler->>+IntSvc: Triggers CreateOrderProcessTask (every 30s)
    IntSvc->>+AzureQueue: GetMessages()
    AzureQueue-->>-IntSvc: [OrderMessage, ...]

    loop For each OrderMessage received
        IntSvc->>IntSvc: Transform OrderMessage to QuoteData
        Note right of IntSvc: Anti-Corruption Layer: Maps external<br>model to internal MRP model.

        IntSvc->>+OrderSvc: POST /quotes (QuoteData)
        OrderSvc->>+MongoDB: (Creates Quote)
        MongoDB-->>-OrderSvc: Success
        OrderSvc-->>-IntSvc: 201 Created (with new QuoteID)

        IntSvc->>IntSvc: Transform OrderMessage to OrderData
        IntSvc->>+OrderSvc: POST /orders?fromQuote={QuoteID}
        OrderSvc->>+MongoDB: (Creates Order)
        MongoDB-->>-OrderSvc: Success
        OrderSvc-->>-IntSvc: 201 Created (with new OrderID)

        IntSvc->>IntSvc: Transform OrderMessage to ShipmentData
        IntSvc->>+OrderSvc: POST /shipments (ShipmentData for new OrderID)
        OrderSvc->>+MongoDB: (Creates Shipment)
        MongoDB-->>-OrderSvc: Success
        OrderSvc-->>-IntSvc: 201 Created

        IntSvc->>+AzureQueue: DeleteMessage(OrderMessage.id)
        AzureQueue-->>-IntSvc: Success
    end
```

### **5. Asynchronous Inventory Synchronization to External System**

-   **Workflow Purpose & Trigger:** The system periodically sends inventory updates to an external system. The `IntegrationService` polls the `OrderService` for all catalog items and pushes transformed messages to an outbound Azure Storage Queue.
-   **Communication Patterns:**
    -   **Integration Service to Order Service:** Synchronous REST call (`GET /catalog`).
    -   **Integration Service to Queue:** Asynchronous message publishing.

```mermaid
sequenceDiagram
    participant Scheduler
    participant IntSvc as Integration Service
    participant OrderSvc as Order Service
    participant AzureQueue as Azure Storage Queue (inventory)

    Scheduler->>+IntSvc: Triggers UpdateProductProcessTask (every 30s)

    IntSvc->>+OrderSvc: GET /catalog
    Note over IntSvc, OrderSvc: Fetches all catalog items from the MRP system.
    OrderSvc-->>-IntSvc: [CatalogItem1, CatalogItem2, ...]

    loop For each CatalogItem
        IntSvc->>IntSvc: Transform CatalogItem to ProductMessage
        Note right of IntSvc: Maps internal MRP model to<br>the external system's message format.
        IntSvc->>+AzureQueue: SendMessage(ProductMessage)
        AzureQueue-->>-IntSvc: Success
    end

    IntSvc-->>-Scheduler: Task complete
```

### **6. Database Operation with Retry and Telemetry**

-   **Workflow Purpose & Trigger:** This diagram shows the internal resilience and observability pattern for database interactions within the `OrderService`. Any repository operation triggers this flow, which uses a Decorator pattern (`MongoOperationsWithRetry`) to handle transient network errors.
-   **Communication Patterns:**
    -   **Repository to DB Driver:** Internal Java method call.
    -   **DB Driver to Application Insights:** Out-of-band telemetry call.

```mermaid
sequenceDiagram
    participant Repo as Repository (e.g., OrderRepository)
    participant RetryWrapper as MongoOperationsWithRetry
    participant SpringMongo as Spring MongoOperations
    participant MongoDB
    participant AppInsights as Azure Application Insights

    Repo->>+RetryWrapper: find(...)
    Note right of Repo: A repository method is called<br>by the business logic.

    RetryWrapper->>+SpringMongo: find(...)
    Note right of RetryWrapper: First attempt.
    SpringMongo->>+MongoDB: network request
    MongoDB-->>-SpringMongo: Throws SocketTimeoutException
    SpringMongo-->>-RetryWrapper: Exception

    alt On SocketTimeoutException (and first attempt)
        RetryWrapper->>RetryWrapper: Log "Retrying operation..."
        RetryWrapper->>+SpringMongo: find(...)
        Note right of RetryWrapper: Second attempt.
        SpringMongo->>+MongoDB: network request
        MongoDB-->>-SpringMongo: Success (query results)
        SpringMongo-->>-RetryWrapper: Success (query results)
    end

    RetryWrapper->>+AppInsights: TrackDependency("MongoDB", "find", duration, success=true)
    Note right of RetryWrapper: Telemetry is sent for every<br>DB operation, regardless of retry.
    AppInsights-->>-RetryWrapper: ack

    RetryWrapper-->>-Repo: Success (query results)
```