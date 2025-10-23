```mermaid
graph TB
    subgraph "External Systems"
        ExternalWebsite["External Parts Unlimited Website"]
        GoogleMaps["Google Maps Places API"]
        AppInsights["Azure Application Insights"]
    end

    subgraph "Parts Unlimited MRP System"
        WebUI["Web UI (SPA - WinJS)"]
        OrderService["Order Service (Monolithic Core - Spring)"]
        IntegrationService["Integration Service (Adapter - Spring)"]
    end

    subgraph "Data & Messaging Infrastructure"
        Database["Database (MongoDB / PostgreSQL)"]
        OrderQueue["Azure Storage Queue (Orders In)"]
        InventoryQueue["Azure Storage Queue (Inventory Out)"]
    end

    %% --- User & Synchronous Interactions ---
    WebUI -- "Sync REST API (Chatty)" --> OrderService
    WebUI -- "Address Autocomplete" --> GoogleMaps

    %% --- Backend Service Interactions ---
    IntegrationService -- "Sync REST API (CRUD Operations)" --> OrderService
    OrderService -- "CRUD & Telemetry" --> Database
    OrderService -- "Monitoring & Tracing" --> AppInsights

    %% --- Asynchronous Integration Flow ---
    ExternalWebsite -- "Writes OrderMessage" --> OrderQueue
    OrderQueue -- "Polls for Messages" --> IntegrationService
    IntegrationService -- "Publishes ProductMessage" --> InventoryQueue
    InventoryQueue -- "Consumed by" --> ExternalWebsite
```

The architecture is a logically-decomposed monolith centered around the `Order Service`, which holds all core business logic and state. The `Web UI` and `Integration Service` act as distinct clients, communicating with the core via synchronous REST APIs. Asynchronous integration with external systems is achieved through the `Integration Service` acting as an adapter, which polls and publishes messages to Azure Storage Queues, decoupling the core system from external data formats and availability.