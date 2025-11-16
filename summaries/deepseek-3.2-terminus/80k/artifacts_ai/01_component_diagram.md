```mermaid
graph TB
    WebClient[Web Client SPA<br/>WinJS/HTML5/Tomcat] --> OrderService[OrderService<br/>Spring Boot REST API]
    IntegrationService[IntegrationService<br/>Queue Processing] --> OrderService
    IntegrationService --> AzureQueues[Azure Storage Queues<br/>External Integration]
    OrderService --> MongoDB[(MongoDB<br/>Primary Data Storage)]
    WebClient --> GoogleMaps[Google Maps API<br/>Address Validation]
    
    subgraph OrderService_Internal
        CatalogController[Catalog Controller]
        DealerController[Dealer Controller]
        QuoteController[Quote Controller]
        OrderController[Order Controller]
        ShipmentController[Shipment Controller]
    end
    
    OrderService --> OrderService_Internal
    
    %% Data flow relationships
    QuoteController --> OrderController
    OrderController --> ShipmentController
    CatalogController --> DealerController
    CatalogController --> QuoteController

```

The component boundaries are organized around functional domains with OrderService serving as the central business logic hub containing all core business operations. Communication follows synchronous REST patterns between frontend and backend, while IntegrationService uses asynchronous queue-based processing for external system integration. The Web Client maintains a clear separation as a presentation layer, delegating all data operations to OrderService through a centralized data service layer.