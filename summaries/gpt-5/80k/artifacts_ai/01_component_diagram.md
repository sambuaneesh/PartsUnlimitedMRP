```mermaid
graph TB
  %% Top-level actors
  Client[Clients (WinJS SPA)]
  External[External Website]
  AppI[Application Insights]

  %% OrderService monolith
  subgraph OrderService["OrderService (Spring Boot REST monolith)"]
    AppFilter[AppInsightsFilter (HTTP cross-cutting)]
    Api[Domain Controllers: /catalog, /dealers, /quotes, /orders, /shipments, /ping]
    Repos[Repositories (Spring Data Mongo)]
    MongoRetry[MongoOperationsWithRetry]
  end

  %% Data store
  DB[(MongoDB 'ordering')]

  %% IntegrationService worker
  subgraph IntegrationService["IntegrationService (Spring Boot worker)"]
    OrdersIngest[OrdersIngestJob (every 30s)]
    InventoryPub[InventoryPublisherJob (every 30s)]
    MrpConnect[MrpConnectService (REST client)]
    QueueSvc[QueueService<T> + QueueFactory]
  end

  %% Azure queues
  subgraph Azure["Azure Storage Queues"]
    OrdersQ["orders (inbound)"]
    ProductQ["product (outbound)"]
  end

  %% Client-to-service interactions
  Client -->|HTTP/JSON (CORS)| AppFilter
  AppFilter --> Api

  %% IntegrationService-to-OrderService
  OrdersIngest -->|create Quote/Order/Shipment| MrpConnect
  InventoryPub -->|GET catalog| MrpConnect
  MrpConnect -->|REST| Api

  %% OrderService data access and telemetry
  Api -->|Repository pattern| Repos
  Repos --> MongoRetry
  MongoRetry -->|CRUD| DB
  AppFilter -->|Request/Exception telemetry| AppI
  MongoRetry -->|Dependency telemetry (MongoDB)| AppI

  %% Queue-based integration
  External -->|enqueue OrderMessage| OrdersQ
  OrdersQ -->|dequeue JSON| QueueSvc --> OrdersIngest
  InventoryPub --> QueueSvc -->|enqueue ProductMessage| ProductQ
  ProductQ -->|consume inventory feed| External
```

The system has three deployables: a WinJS SPA client, a Spring Boot OrderService monolith encapsulating all domain controllers over a single MongoDB, and a Spring Boot IntegrationService that bridges to an external website via Azure Storage Queues. The client and IntegrationService call OrderService synchronously over REST, while website interactions are asynchronous through queues for order ingestion and product inventory publication; OrderService adds cross-cutting telemetry and Mongo retry around data access.