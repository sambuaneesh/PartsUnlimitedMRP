
```mermaid
graph TB
    subgraph "Client Layer"
        Website[Website Frontend]
    end

    subgraph "Ordering Service (smpl.ordering)"
        direction TB
        API[Ordering API<br/>Controllers]
        Repo[Data Access Layer<br/>Repositories]
        DB[(MongoDB)]
        CatalogConsumer[Catalog Updater<br/>(Queue Consumer)]
        API -- Manages Orders/Quotes --> Repo
        Repo -- Persists/Retrieves --> DB
    end

    subgraph "Integration Service (integration)"
        direction TB
        OrderTask[Scheduled Task:<br/>Create Order]
        ProductTask[Scheduled Task:<br/>Update Product]
        MrpService[MrpConnectService]
        QueueService[Queue Service]
    end

    subgraph "External Systems"
        ExtMRP[External MRP System]
        AzureQueue[Azure Queue Service]
    end

    %% Interactions
    Website -- "1. Submits Order (REST)" --> API
    API -- "2. Places Order Message" --> AzureQueue
    OrderTask -- "3. Polls for Order Messages" --> AzureQueue
    OrderTask -- "4. Calls MRP API" --> MrpService
    MrpService -- "5. Creates Order in" --> ExtMRP

    ProductTask -- "6. Polls MRP for Products" --> MrpService
    MrpService -- "7. Fetches Product Data" --> ExtMRP
    ProductTask -- "8. Publishes Product Updates" --> QueueService
    QueueService -- "Manages Queues" --> AzureQueue
    AzureQueue -- "9. Notifies of Product Updates" --> CatalogConsumer
    CatalogConsumer -- "10. Updates Local Catalog" --> Repo

    %% Styling
    classDef client fill:#ECEFF1,stroke:#37474F,stroke-width:2px;
    classDef ordering fill:#C5E1A5,stroke:#33691E,stroke-width:2px;
    classDef integration fill:#FFCCBC,stroke:#BF360C,stroke-width:2px;
    classDef external fill:#B39DDB,stroke:#4527A0,stroke-width:2px;
    class Website client;
    class API,Repo,DB,CatalogConsumer ordering;
    class OrderTask,ProductTask,MrpService,QueueService integration;
    class ExtMRP,AzureQueue external;
```

The system is cleanly decomposed into two primary services: a synchronous `ordering` service for user-facing transactions and an asynchronous `integration` service for background processing and external system communication. This separation allows the fast, interactive API operations in the `ordering` service to remain responsive, while the `integration` service reliably handles slower, I/O-bound tasks like order ingestion and product synchronization. Communication between the services and with external systems is predominantly asynchronous, mediated by a message queue which decouples components, enhances resilience against failures, and supports the event-driven nature of the supply chain workflow.