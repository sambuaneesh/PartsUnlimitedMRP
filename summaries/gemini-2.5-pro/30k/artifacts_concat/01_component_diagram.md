```markdown
graph TB
    subgraph "External Systems"
        ExternalWebsite["External Parts Unlimited Website"]
        MessageQueue["Message Queue (Azure)"]
    end

    subgraph "Parts Unlimited MRP System"
        User[("User")]
        WebApp["Frontend Web App (SPA)"]
        OrderService["Order Service (Java/Spring)"]
        IntegrationService["Integration Service (Java/Spring)"]
        MongoDB[("Monolithic MongoDB")]
    end

    %% Main User Flow
    User -- "Interacts via Browser (HTTP)" --> WebApp
    WebApp -- "REST API Calls (HTTP/JSON)" --> OrderService

    %% Backend Persistence
    OrderService -- "Reads/Writes Data" --> MongoDB

    %% Asynchronous Integration Flow
    ExternalWebsite -- "Publishes Order Messages" --> MessageQueue
    MessageQueue -- "Consumes Orders" --> IntegrationService
    IntegrationService -- "Publishes Inventory Updates" --> MessageQueue
    IntegrationService -- "Creates Orders via REST API" --> OrderService
```

The architecture consists of a user-facing Single Page Application (SPA), a core `OrderService` handling all business logic via a REST API, and a separate `IntegrationService` that acts as an anti-corruption layer to an external system. The SPA communicates synchronously with the `OrderService`, while the `IntegrationService` uses a message queue for asynchronous external integration and REST for internal communication, with both backend services sharing a single MongoDB database for persistence.