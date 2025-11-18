
```mermaid
graph TB
    subgraph "Client Layer"
        Frontend[Web SPA
        (Tomcat + WinJS)]
    end

    subgraph "Backend Services"
        OrderService[Order Service
        (Spring Boot)]
        IntegrationService[Integration Service
        (Spring Boot + Azure SDK)]
    end

    subgraph "Persistence & Integration"
        MongoDB[(MongoDB)]
        AzureQueue[Azure Queue Storage]
        ExternalMRP[External MRP System]
    end

    Frontend -- "REST API" --> OrderService
    OrderService -- "Read/Write Data" --> MongoDB
    OrderService -- "Publishes Events" --> AzureQueue
    IntegrationService -- "Processes Queue Messages" --> AzureQueue
    IntegrationService -- "Syncs Data" --> ExternalMRP
    IntegrationService -- "Callback API (Updates)" --> OrderService
```

The architecture follows a client-server pattern with a decoupled frontend and backend. The `OrderService` centralizes core business logic and state management, while the `IntegrationService` isolates external system communication using asynchronous messaging via Azure Queue. This design separates immediate user-facing operations from long-running integration tasks, enhancing system responsiveness and resilience.