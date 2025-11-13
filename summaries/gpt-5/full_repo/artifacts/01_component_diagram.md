```mermaid
graph TB
  %% Clients
  U[User Browser] -->|HTTP GET :9080/mrp| WEB[Web Frontend (mrp.war on Tomcat)]

  %% Web -> Backend
  WEB -->|REST HTTP :8080<br/>(/catalog, /dealers, /quotes, /orders, /shipments)| OS_CTRL[OrderService REST Controllers]

  %% OrderService internals
  subgraph Order Service (Spring Boot)
    OS_CTRL --> OS_REPO[Repositories]
    OS_REPO --> DB[(MongoDB)]
    OS_CTRL -. telemetry .-> AI[(Application Insights)]
  end

  %% Integration Service
  subgraph Integration Service (Spring Boot, scheduled)
    CO[CreateOrderProcessTask] -->|Dequeue OrderMessage| QO[[Azure Queue: orders]]
    CO -->|Create Quote/Order/Shipment (REST)| OS_CTRL
    UP[UpdateProductProcessTask] -->|GET catalog (REST)| OS_CTRL
    UP -->|Enqueue ProductMessage| QP[[Azure Queue: product]]
  end

  %% External website integrating via queues
  PU[Parts Unlimited Website (external)] -->|Enqueue OrderMessage| QO
  PU <-->|Consume ProductMessage| QP

  %% Optional containerized deployment view
  subgraph Optional Docker Deployment
    D_WEB[Tomcat container :9080]:::c -.serves.-> WEB
    D_OS[OrderService container :8080]:::c -.hosts.-> OS_CTRL
    D_DB[MongoDB container :27017]:::c -.stores.-> DB
  end

  classDef c fill:#f0f8ff,stroke:#77a,stroke-width:1px
```

The static web frontend (mrp.war on Tomcat) serves the SPA and calls the OrderService over REST on port 8080; the OrderService uses repositories backed by MongoDB and emits telemetry to Application Insights. A separate IntegrationService runs scheduled tasks that bridge with the external Parts Unlimited website through Azure Storage queues: it consumes order messages to create quotes/orders/shipments and publishes catalog/inventory updates back to the website. An optional Docker deployment packages the web, service, and database into separate containers exposing ports 9080/8080/27017 respectively.