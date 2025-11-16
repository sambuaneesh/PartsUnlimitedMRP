```mermaid
graph TB
    %% Frontend Components
    WebClient[Web Client<br/>WinJS Frontend] -->|REST API| OrderService
    
    %% Core Backend Services
    OrderService[Order Service<br/>Spring Boot] -->|Data Persistence| MongoDB[(MongoDB)]
    IntegrationService[Integration Service<br/>Queue Processing] -->|Order Updates| OrderService
    
    %% External Systems
    ExternalSystems[External Systems<br/>Parts Unlimited Website] -->|Async Queue| IntegrationService
    
    %% Service Internal Structure
    subgraph OrderService_Internal
        CatalogController[Catalog Controller]
        OrderController[Order Controller]
        QuoteController[Quote Controller]
        ShipmentController[Shipment Controller]
        DealerController[Dealer Controller]
        PingController[Health Check]
    end
    
    %% Data Flow Patterns
    WebClient -->|HTTP Requests| OrderService_Internal
    OrderService_Internal -->|CRUD Operations| MongoDB

    %% Communication Styles
    WebClient -.->|Synchronous REST| OrderService
    ExternalSystems -.->|Asynchronous Queue| IntegrationService
    IntegrationService -.->|Synchronous API| OrderService
```

The component boundaries are organized around business domains: Order Service handles core ordering workflows, Integration Service manages external system communication, and Web Client provides the user interface. Communication patterns follow a hybrid approach with synchronous REST APIs for user interactions and asynchronous queue-based messaging for external integrations. The architecture maintains clear separation between presentation, business logic, and data persistence layers.