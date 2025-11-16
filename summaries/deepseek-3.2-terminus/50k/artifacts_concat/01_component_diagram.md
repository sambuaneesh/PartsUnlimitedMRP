```mermaid
graph TB
    %% Frontend Components
    FrontendClient[Frontend Client<br/>WinJS SPA]
    WebUI[Web UI<br/>HTML/CSS/JS]
    Navigation[Navigation System]
    DataBinding[Data Binding Layer]
    
    %% Backend Services
    OrderService[Order Service<br/>Spring Boot]
    IntegrationService[Integration Service<br/>Spring Boot]
    
    %% Order Service Subcomponents
    CatalogController[Catalog Controller]
    OrderController[Order Controller]
    QuoteController[Quote Controller]
    ShipmentController[Shipment Controller]
    DealerController[Dealer Controller]
    
    %% Integration Service Subcomponents
    OrderProcessTask[Order Process Task]
    ProductUpdateTask[Product Update Task]
    QueueService[Queue Service]
    MrpConnectService[MRP Connect Service]
    
    %% External Systems
    MongoDB[(MongoDB<br/>Database)]
    AzureQueue[(Azure Storage<br/>Queues)]
    MRPSystem[External MRP System]
    GoogleMaps[Google Maps API]
    AppInsights[Application Insights]
    
    %% Internal Dependencies
    OrderRepository[Order Repository]
    QuoteRepository[Quote Repository]
    CatalogRepository[Catalog Repository]
    ShipmentRepository[Shipment Repository]
    DealerRepository[Dealer Repository]
    
    %% Component Relationships
    FrontendClient --> WebUI
    FrontendClient --> Navigation
    FrontendClient --> DataBinding
    
    FrontendClient --> OrderService
    FrontendClient --> GoogleMaps
    
    OrderService --> CatalogController
    OrderService --> OrderController
    OrderService --> QuoteController
    OrderService --> ShipmentController
    OrderService --> DealerController
    OrderService --> AppInsights
    
    CatalogController --> CatalogRepository
    OrderController --> OrderRepository
    QuoteController --> QuoteRepository
    ShipmentController --> ShipmentRepository
    DealerController --> DealerRepository
    
    OrderRepository --> MongoDB
    QuoteRepository --> MongoDB
    CatalogRepository --> MongoDB
    ShipmentRepository --> MongoDB
    DealerRepository --> MongoDB
    
    IntegrationService --> OrderProcessTask
    IntegrationService --> ProductUpdateTask
    IntegrationService --> QueueService
    IntegrationService --> MrpConnectService
    IntegrationService --> AppInsights
    
    OrderProcessTask --> AzureQueue
    ProductUpdateTask --> AzureQueue
    QueueService --> AzureQueue
    MrpConnectService --> OrderService
    MrpConnectService --> MRPSystem
    
    %% Cross-service communication
    OrderProcessTask -.-> OrderService
    ProductUpdateTask -.-> CatalogController

```

**Rationale:** The component boundaries follow domain-driven design principles with clear separation between frontend presentation, business logic services, and integration layers. Communication patterns include synchronous REST APIs for frontend-backend interactions, asynchronous queue-based messaging for external system integration, and repository-based data access patterns for database operations.