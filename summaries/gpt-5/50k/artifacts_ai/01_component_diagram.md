```mermaid
graph TB
  %% Topology
  subgraph Client
    BROWSER[Browser SPA]
  end

  subgraph Web
    WEB[Web Front End (Tomcat)\nStatic HTML/JS/CSS\nPorts: 8080 (VM: 9080, Docker: 80→8080)]
  end

  subgraph Backend
    OS[OrderService (Spring Boot)\nREST API\nHTTP: 8080\nActuator: 8081 (loopback)]
    IS[IntegrationService (Spring Boot Worker)\n@Scheduled tasks (30s)]
  end

  subgraph Data Stores
    DB[(MongoDB\n27017)]
  end

  subgraph Messaging (Azure Storage Queues)
    QORD((orders queue))
    QPROD((product queue))
  end

  subgraph External
    EXTWEB[External Website]
  end

  %% Flows
  BROWSER -- "HTTP GET static" --> WEB
  BROWSER -- "AJAX HTTP/JSON (8080)\nCORS: *" --> OS
  OS --- DB
  IS -- "POST /quotes, /orders?fromQuote, /shipments\nGET /catalog (REST/JSON)" --> OS
  IS -- "Publish ProductMessage" --> QPROD
  IS -- "Consume OrderMessage" --> QORD
  EXTWEB -- "Enqueue OrderMessage" --> QORD
  QPROD -- "Consume ProductMessage" --> EXTWEB
```

The system centers on a single backend component (OrderService) that owns all core domains and persists to MongoDB, a headless IntegrationService that bridges to an external website via Azure Storage Queues, and a static Web Front End that serves the SPA. The browser loads static assets from Tomcat but communicates directly with OrderService over REST (enabled by permissive CORS), while IntegrationService performs synchronous REST calls to OrderService and asynchronous queue operations to decouple website order ingestion and product updates.