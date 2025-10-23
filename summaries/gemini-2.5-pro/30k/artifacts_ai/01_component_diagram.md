```markdown
```mermaid
graph TB
    subgraph "User & External Systems"
        User(["fa:fa-user User"])
        ExternalSystem["External Parts Unlimited Website"]
    end

    subgraph "Parts Unlimited MRP System"
        FrontendSPA["Frontend Service (SPA)"]
        OrderService["Order Service (Monolithic Backend)"]
        IntegrationService["Integration Service (Async Bridge)"]
        MongoDB["fa:fa-database MongoDB (Shared)"]
    end

    subgraph "Integration Layer (Azure)"
        OrdersQueue["Azure Queue (New Orders)"]
        ProductsQueue["Azure Queue (Product Updates)"]
    end

    %% -- Main Synchronous User Flow --
    User -- "Interacts via Browser" --> FrontendSPA
    FrontendSPA -- "REST API Calls (HTTP)" --> OrderService
    OrderService -- "Reads/Writes Data" --> MongoDB

    %% -- Asynchronous Integration Flow --
    ExternalSystem -- "1. Pushes new order messages" --> OrdersQueue
    IntegrationService -- "2. Consumes messages" --> OrdersQueue
    IntegrationService -- "3. Calls API to create entities" --> OrderService
    IntegrationService -- "4. Calls API to get catalog data" --> OrderService
    IntegrationService -- "5. Produces product update messages" --> ProductsQueue
    ProductsQueue -- "6. Consumed by" --> ExternalSystem
```

The architecture is a three-tier system composed of a `Frontend SPA`, a monolithic `Order Service` backend, and a shared `MongoDB` database. Communication between the frontend and backend is synchronous via REST APIs, while an `Integration Service` handles asynchronous communication with an external system through message queues, acting as a resilient anti-corruption layer.
```