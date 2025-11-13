```mermaid
graph TB
  %% Users and Client
  U[End User Browser]
  C[Web Frontend (Tomcat WAR)\nWinJS SPA\nTomcat:8080 (exposed 9080/80)]
  U --> C

  %% OrderService (internals as components)
  subgraph OS_SG["OrderService (Spring Boot) :8080"]
    CTRL[REST Controllers\n/catalog /dealers /quotes /orders /shipments /ping]
    REPO[Repository Layer\n(MongoDB or memory)\nMongoOperationsWithRetry]
    CORS[SimpleCORSFilter (CORS)]
    AIF[AppInsightsFilter]
    CTRL --> REPO
    CORS -.cross-cutting.-> CTRL
    AIF -.cross-cutting.-> CTRL
  end

  %% IntegrationService internals
  subgraph IS_SG["IntegrationService (Spring Boot)\n(no HTTP API)"]
    TASKS[Scheduled Tasks\nCreateOrderProcessTask\nUpdateProductProcessTask]
    MRPAPI[MrpConnectService (RestTemplate)]
    TASKS --> MRPAPI
  end

  %% Data store and telemetry
  DB[(MongoDB\n"ordering" database\n:27017)]
  AI[(Application Insights)]

  %% Queues and external website
  subgraph ASQ["Azure Storage Queues"]
    Q1["orders queue (in)"]
    Q2["product queue (out)"]
  end
  SITE[Parts Unlimited Website]

  %% Client to OrderService
  C -->|AJAX HTTP/JSON (CORS)\nserverconfig.js base URL| CTRL

  %% IntegrationService <-> OrderService
  MRPAPI -->|POST /quotes\nPOST /orders?fromQuote\nPOST /shipments\nGET /catalog| CTRL

  %% OrderService -> MongoDB
  REPO -->|Mongo driver w/ retry| DB

  %% Telemetry
  AIF -.telemetry.-> AI
  REPO -.Mongo telemetry.-> AI

  %% Website <-> Queues
  SITE -->|enqueue OrderMessage| Q1
  TASKS -->|dequeue OrderMessage| Q1
  TASKS -->|enqueue ProductMessage| Q2
  Q2 -->|consume ProductMessage| SITE
```

Rationale:
- The architecture comprises a static WinJS web frontend served by Tomcat that directly consumes the OrderService REST API, a headless IntegrationService that bridges the external website via Azure Storage Queues and synchronously invokes OrderService, and MongoDB as the persistence store. 
- OrderService consolidates multiple domain controllers behind a repository layer with Mongo retry/telemetry; CORS enables browser calls, and Application Insights captures HTTP and Mongo dependencies. Communication is HTTP/JSON for service calls and Azure Queues for asynchronous website integration.