
```mermaid
graph TB
    subgraph Frontend
        SPA[WinJS SPA Application]
        SPA --> |Navigation & Business Modules| Pages[Pages: Dashboard/Dealers/Quotes/Orders/Deliveries/Catalog]
    end
    
    subgraph Backend
        OrderService[OrderService: Spring Boot]
        IntegrationService[IntegrationService: Spring Boot]
    end
    
    subgraph Data
        MongoDB[(MongoDB)]
        subgraph Collections
            Catalog[(Catalog)]
            Quotes[(Quotes)]
            Orders[(Orders)]
            Shipments[(Shipments)]
            Dealers[(Dealers)]
        end
    end
    
    subgraph External
        GoogleMaps[Google Maps API]
        AzureQueues[Azure Storage Queues]
        PartsUnlimited[Parts Unlimited Website]
    end
    
    SPA --> |REST API Calls| OrderService
    Pages --> |Address Validation| GoogleMaps
    OrderService --> |CRUD Operations| MongoDB
    MongoDB --> Catalog
    MongoDB --> Quotes
    MongoDB --> Orders
    MongoDB --> Shipments
    MongoDB --> Dealers
    IntegrationService --> |Queue Operations| AzureQueues
    IntegrationService --> |Sync Operations| PartsUnlimited
    OrderService --> |Triggers| IntegrationService
```

The component boundaries separate presentation (WinJS SPA), business logic (Spring Boot services), and data persistence (MongoDB), with clear API contracts between layers. Communication patterns follow a request-response model for user interactions via REST APIs, while asynchronous queue-based messaging handles external system integration to maintain loose coupling.