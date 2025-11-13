```mermaid
graph TB
  subgraph Frontend
    UI[MRP Web UI (Tomcat WAR /mrp :9080)]
  end

  subgraph "Backend Services"
    OS[OrderService (Spring Boot REST :8080)]
    IS[IntegrationService (Spring Boot Scheduler)]
  end

  subgraph "Data Stores"
    MDB[(MongoDB "ordering" :27017)]
    subgraph "Azure Storage Queues"
      QOrders[orders queue]
      QProduct[product queue]
    end
  end

  subgraph "External Systems"
    Website[Parts Unlimited Website]
    AI[(Application Insights)]
  end

  %% Frontend -> Backend
  UI -->|"REST JSON (CORS) /catalog,/dealers,/quotes,/orders,/shipments"| OS

  %% Integration flows
  Website -->|"enqueue OrderMessage"| QOrders
  QOrders -->|"poll (scheduled)"| IS
  IS -->|"REST JSON: POST /quotes → POST /orders?fromQuote → POST /shipments"| OS

  OS -->|"MongoTemplate (retry wrapper)"| MDB

  IS -->|"GET /catalog"| OS
  IS -->|"enqueue ProductMessage"| QProduct
  QProduct -->|"consume inventory/lead-time updates"| Website

  %% Telemetry
  OS -.->|"request/exception telemetry"| AI
  MDB -.->|"dependency telemetry via MongoOperationsWithRetry"| AI
```

The system consists of three runtime components: a Tomcat-hosted SPA (UI), the OrderService REST API, and the IntegrationService that bridges Azure Storage Queues with the API. The UI communicates synchronously with OrderService over REST (CORS-enabled), OrderService persists to MongoDB with retry-backed operations, and IntegrationService asynchronously consumes/produces queue messages while invoking OrderService to materialize quotes, orders, and shipments.