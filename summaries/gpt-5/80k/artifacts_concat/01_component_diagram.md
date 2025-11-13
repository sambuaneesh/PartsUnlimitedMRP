```mermaid
graph TB
  %% Top-level components
  ClientWeb[Clients (WinJS SPA on Tomcat)]
  ExternalSite[External Website (Parts Unlimited)]
  OrdersQueue[(Azure Queue: orders)]
  ProductQueue[(Azure Queue: product)]
  MongoDB[(MongoDB: ordering)]

  %% OrderService internals
  subgraph OrderService["OrderService (Spring)"]
    OS_Facade[REST API Surface]
    subgraph Controllers["Controllers"]
      C_Catalog[CatalogController]
      C_Dealers[DealerController]
      C_Quotes[QuoteController]
      C_Orders[OrderController]
      C_Ship[ShipmentController]
      C_Ping[PingController]
    end
    RepoFactory[RepositoryFactory]
    MongoRepos[Mongo* Repositories + Retry/Telemetry]
  end

  %% IntegrationService internals
  subgraph IntegrationService["IntegrationService (Spring Boot)"]
    IS_Core[Service Host]
    MrpConnect[MrpConnectService (REST client)]
    QueueSvc[QueueService<T> + QueueFactory]
    OrderJob[Order Ingestion (poll 30s)]
    InvJob[Inventory Publisher (poll 30s)]
  end

  %% Client -> OrderService
  ClientWeb -->|HTTP/JSON REST (/catalog,/dealers,/quotes,/orders,/shipments)| OS_Facade
  OS_Facade --> C_Catalog
  OS_Facade --> C_Dealers
  OS_Facade --> C_Quotes
  OS_Facade --> C_Orders
  OS_Facade --> C_Ship
  OS_Facade --> C_Ping

  %% Controllers -> Persistence
  C_Catalog --> RepoFactory
  C_Dealers --> RepoFactory
  C_Quotes --> RepoFactory
  C_Orders --> RepoFactory
  C_Ship --> RepoFactory
  RepoFactory --> MongoRepos -->|Mongo driver| MongoDB

  %% IntegrationService -> Queues and OrderService
  IS_Core --> OrderJob
  IS_Core --> InvJob
  OrderJob --> QueueSvc
  InvJob --> QueueSvc
  OrderJob -.->|Dequeue OrderMessage| OrdersQueue
  InvJob -.->|Enqueue ProductMessage| ProductQueue
  OrderJob --> MrpConnect
  InvJob --> MrpConnect
  MrpConnect -->|POST /quotes| OS_Facade
  MrpConnect -->|POST /orders?fromQuote| OS_Facade
  MrpConnect -->|POST /shipments| OS_Facade
  MrpConnect -->|GET /catalog| OS_Facade

  %% External website integration via queues
  ExternalSite -.->|Enqueue OrderMessage| OrdersQueue
  ProductQueue -.->|Consume ProductMessage| ExternalSite
```

The system is composed of three deployable components: a static web client, a single backend service (OrderService) owning the ordering MongoDB, and an IntegrationService that bridges an external website via Azure Storage Queues. The client and IntegrationService communicate synchronously with OrderService over REST, while IntegrationService and the external website communicate asynchronously through the orders and product queues, enabling decoupled, eventually consistent integration.