```markdown
1. ```mermaid
graph TB
    WebFrontend[Web Frontend Service<br/>WinJS Application<br/>Port: 9080]
    
    OrderService[Order Service<br/>Spring Boot Application<br/>Port: 8080]
    IntegrationService[Integration Service<br/>Async Processing]
    
    MongoDB[(MongoDB<br/>Primary Data Store)]
    AzureQueues[Azure Storage Queues<br/>Messaging]
    GoogleMaps[Google Maps API<br/>Address Validation]
    
    WebFrontend --> OrderService
    WebFrontend --> GoogleMaps
    
    OrderService --> MongoDB
    OrderService --> IntegrationService
    
    IntegrationService --> AzureQueues
    IntegrationService --> OrderService
    
    subgraph "Order Service Domains"
        CatalogController[Catalog Controller<br/>Product Management]
        DealerController[Dealer Controller<br/>Partner Management]
        QuoteController[Quote Controller<br/>Pricing & Quotes]
        OrderController[Order Controller<br/>Order Lifecycle]
        ShipmentController[Shipment Controller<br/>Delivery Tracking]
    end
    
    CatalogController --> MongoDB
    DealerController --> MongoDB
    QuoteController --> MongoDB
    OrderController --> MongoDB
    ShipmentController --> MongoDB
```

2. The component boundaries follow a layered architecture with clear separation between presentation (Web Frontend), business logic (Order Service), and integration layers (Integration Service). Communication patterns include synchronous REST APIs for real-time operations between frontend and backend, and asynchronous queue-based messaging for external system integration and batch processing. The Order Service maintains a monolithic structure internally but exposes domain-specific REST APIs for different business capabilities.
```