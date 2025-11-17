```mermaid
graph TB

subgraph External Channels
  EXT[Website / Partner Channels]
end

subgraph Messaging (Azure Storage Queues)
  QOrder[(Order Queue)]
  QInv[(Inventory Queue)]
end

subgraph Integration Worker (Spring Scheduling)
  IW[Integration Daemon]
  S1[CreateOrderProcessTask]
  S2[UpdateProductProcessTask]
  RESTClient[MrpConnectService (REST Adapter)]
  QueueSvc[QueueService / QueueFactory]
  ConfigInt[ConfigurationManager]
end

subgraph Ordering Service (Spring Boot)
  Controllers[REST Controllers\n(Catalog, Dealers, Quotes, Orders, Shipments, Ping)]
  RepoFactory[RepositoryFactory]
  ConfigOS[OrderingConfiguration\n+ OrderingServiceProperties]
  Domain[(Domain DTOs)]
end

subgraph Persistence
  Repos[Repository Implementations\n(MongoDB or In-Memory)]
  MongoModels[Mongo Document Mappers]
  Mongo[(MongoDB)]
  InMem[(In-Memory Store)]
end

AI[(Application Insights)]

%% External <-> Messaging
EXT -->|enqueue OrderMessage| QOrder
QInv -->|consume ProductMessage| EXT

%% Integration Worker <-> Messaging and REST
IW --> S1
IW --> S2
S1 --> QueueSvc
S2 --> QueueSvc
QueueSvc --> QOrder
QueueSvc --> QInv
S1 -->|create orders| RESTClient
S2 -->|fetch catalog| RESTClient
ConfigInt --> QueueSvc
ConfigInt --> RESTClient

%% Integration Worker -> Ordering Service
RESTClient -->|HTTP| Controllers

%% Ordering Service -> Persistence
Controllers --> RepoFactory
RepoFactory --> Repos
Repos --> MongoModels
MongoModels --> Mongo
Repos --> InMem
Controllers --> Domain

%% Configuration and Observability
ConfigOS --> RepoFactory
ConfigOS --> Controllers
Controllers -. telemetry .-> AI
Repos -. telemetry .-> AI
IW -. telemetry .-> AI
```

Component boundaries separate the API-centric Ordering Service (controllers, domain DTOs, repository abstraction, and MongoDB mappers) from the headless Integration Worker (scheduled adapters using REST and Azure Storage Queues). Communication is hybrid: synchronous REST between the Integration Worker and the Ordering Service, and asynchronous, typed messages via Azure queues for website/partner order intake and product/inventory feeds, enabling decoupling with at-least-once semantics and resilient synchronization. Application Insights instrumentation spans both components for end-to-end observability.