```mermaid
graph TB
    WebClient[Web Client<br/>WinJS SPA] -->|REST API| OrderService[OrderService<br/>Spring Boot]
    IntegrationService[IntegrationService<br/>Spring Boot] -->|REST API| OrderService
    IntegrationService -->|Reads/Processes| AzureQueue[Azure Storage Queues<br/>orders & product]
    OrderService -->|Persistence| MongoDB[(MongoDB<br/>Primary Data Store)]
    
    subgraph OrderService Components
        CatalogController[CatalogController]
        DealerController[DealerController]
        QuoteController[QuoteController]
        OrderController[OrderController]
        ShipmentController[ShipmentController]
        PingController[PingController]
    end
    
    subgraph Web Client Modules
        DealersModule[Dealers Module]
        QuotesModule[Quotes Module]
        OrdersModule[Orders Module]
        DeliveriesModule[Deliveries Module]
        CatalogModule[Catalog Module]
        EventsModule[Events Module]
    end
    
    WebClient --> DealersModule
    WebClient --> QuotesModule
    WebClient --> OrdersModule
    WebClient --> DeliveriesModule
    WebClient --> CatalogModule
    WebClient --> EventsModule

```

The system follows a layered client-server architecture with clear separation between presentation, business logic, and data persistence layers. Component boundaries are defined by business domains (catalog, dealers, quotes, orders, shipments) with the OrderService acting as the centralized backend API gateway. Communication primarily uses synchronous REST patterns between frontend and backend, while IntegrationService employs asynchronous queue-based processing for external system integration, creating a hybrid request-response and event-driven architecture.