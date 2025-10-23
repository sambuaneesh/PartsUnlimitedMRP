```markdown
```mermaid
graph TB
    subgraph "User & External Systems"
        User(["User"])
        ExternalWebsite["External Parts Unlimited Website"]
        GoogleMapsAPI["Google Maps API"]
        AppInsights["Application Insights"]
    end

    subgraph "MRP Application"
        Frontend_SPA["Frontend SPA (WinJS)<br>Client-side logic & UI"]
        OrderService["OrderService Monolith (Spring Boot)<br>Core Business Logic & REST API"]
        IntegrationService["IntegrationService (Spring Boot)<br>Asynchronous Adapter"]
        MongoDB[(MongoDB<br>Primary Data Store)]
        AzureOrdersQueue["Azure Queue (orders)"]
        AzureProductQueue["Azure Queue (product)"]
    end

    %% Client to Backend Communication
    User --> Frontend_SPA
    Frontend_SPA -- "REST API Calls (AJAX)" --> OrderService
    Frontend_SPA -- "Address Autocomplete" --> GoogleMapsAPI

    %% Backend Monolith Dependencies
    OrderService -- "Reads/Writes Data" --> MongoDB
    OrderService -- "Sends Telemetry" --> AppInsights

    %% Integration Service Flow
    IntegrationService -- "REST API Calls<br>(Polls Catalog, Creates Orders)" --> OrderService
    ExternalWebsite -- "Writes new orders" --> AzureOrdersQueue
    AzureOrdersQueue -- "Polls for new orders" --> IntegrationService
    IntegrationService -- "Pushes inventory updates" --> AzureProductQueue
    AzureProductQueue -- "Reads inventory updates" --> ExternalWebsite
```

The architecture is a hybrid three-tier model centered on the `OrderService` monolith, which encapsulates all core business logic and exposes it via a synchronous REST API. The `Frontend SPA` acts as a rich client, communicating directly with the monolith, while the `IntegrationService` serves as an asynchronous adapter, decoupling the core system from an external website using message queues for communication.
```