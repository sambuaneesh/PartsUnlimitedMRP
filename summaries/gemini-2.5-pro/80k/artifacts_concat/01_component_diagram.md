```markdown
graph TB
    subgraph "User & External Systems"
        User["fa:fa-user User"]
        ExternalWebsite["External Parts Unlimited Website"]
    end

    subgraph "Parts Unlimited MRP System"
        ClientsWeb["Clients/Web (SPA on Tomcat)"]
        OrderService["OrderService (Monolith API)"]
        IntegrationService["IntegrationService (Scheduled Tasks)"]
        MongoDB["fa:fa-database MongoDB"]
    end

    subgraph "External Dependencies"
        OrdersQueue["fa:fa-envelope Azure Orders Queue"]
        InventoryQueue["fa:fa-envelope Azure Inventory Queue"]
    end

    %% User-facing Synchronous Flow
    User -- "Interacts via Browser" --> ClientsWeb
    ClientsWeb -- "REST API Calls" --> OrderService
    OrderService -- "Reads/Writes Data" --> MongoDB

    %% Asynchronous Integration Flow
    ExternalWebsite -- "Writes new orders" --> OrdersQueue
    IntegrationService -- "1. Polls for new orders" --> OrdersQueue
    IntegrationService -- "2. Creates Quotes/Orders via API" --> OrderService

    IntegrationService -- "3. Polls for catalog changes" --> OrderService
    IntegrationService -- "4. Pushes inventory updates" --> InventoryQueue
    ExternalWebsite -- "Reads inventory updates" --> InventoryQueue

    %% Styling
    classDef service fill:#d6e4ff,stroke:#6b95ff,stroke-width:2px;
    classDef ui fill:#e3f0d9,stroke:#90c574,stroke-width:2px;
    classDef db fill:#ffe6cc,stroke:#ffb366,stroke-width:2px;
    classDef external fill:#f5f5f5,stroke:#999,stroke-width:2px;
    classDef queue fill:#f9d5e5,stroke:#e83e8c,stroke-width:2px;

    class ClientsWeb ui;
    class OrderService,IntegrationService service;
    class MongoDB db;
    class ExternalWebsite,User external;
    class OrdersQueue,InventoryQueue queue;
```

The architecture consists of a core monolithic `OrderService` with a dedicated `Clients/Web` frontend and a single `MongoDB` persistence layer. User interactions follow a synchronous pattern, with the frontend making direct REST API calls to the monolith for all business operations. An independent `IntegrationService` handles asynchronous communication with an external website by using Azure Storage Queues, decoupling the core system from external order ingestion and inventory update processes.