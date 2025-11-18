
```mermaid
graph TB
    subgraph "Presentation Layer"
        WinJS[WinJS SPA<br/>Port 9080]
    end
    
    subgraph "Business Logic Layer"
        OrderService[Order Service<br/>Port 8080<br/>Spring Boot]
        IntegrationService[Integration Service<br/>Spring Boot]
    end
    
    subgraph "Data Storage Layer"
        MongoDB[MongoDB<br/>Port 27017<br/>Collections: catalog, dealers, quotes, orders, shipments]
    end
    
    subgraph "External Systems"
        AzureQueue[Azure Queue Service]
        WebsiteAPI[Parts Unlimited Website<br/>REST API]
        MRPAPI[MRP System<br/>REST API]
    end
    
    WinJS -->|REST API| OrderService
    OrderService -->|Direct DB Access| MongoDB
    IntegrationService -->|Queue Operations| AzureQueue
    IntegrationService -->|Periodic Sync| WebsiteAPI
    IntegrationService -->|Order Processing| MRPAPI
    IntegrationService -.->|Data Sync| MongoDB
```

The component boundaries reflect the three-tier monolithic architecture with clear separation of concerns between presentation, business logic, and data layers. Communication primarily flows from the WinJS frontend through REST APIs to the Order Service for core business operations, while the Integration Service handles asynchronous external system connectivity via queue-based messaging and scheduled tasks.