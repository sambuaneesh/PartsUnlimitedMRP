
```mermaid
graph TB
    subgraph "Frontend Layer"
        WebClient[Web Client<br/>WinJS SPA<br/>Port 9080]
    end
    
    subgraph "Backend Services"
        OrderService[Order Service<br/>Spring Boot<br/>Port 8080]
        IntegrationService[Integration Service<br/>Spring Boot<br/>Async Processing]
    end
    
    subgraph "Database Layer"
        MongoDB[(MongoDB<br/>Port 27017)]
        
        subgraph "Collections"
            Catalog[(catalog)]
            Dealers[(dealers)]
            Quotes[(quotes)]
            Orders[(orders)]
            Shipments[(shipments)]
        end
        
        MongoDB --> Catalog
        MongoDB --> Dealers
        MongoDB --> Quotes
        MongoDB --> Orders
        MongoDB --> Shipments
    end
    
    subgraph "External Systems"
        ExternalWebsite[Parts Unlimited Website]
        MessageQueue[Azure Queue Service]
    end
    
    subgraph "Deployment Infrastructure"
        Docker[Docker Containers]
        Azure[Azure Stack Templates]
        Chef[Chef Cookbooks]
        Puppet[Puppet Manifests]
    end
    
    %% Communication paths
    WebClient --> |REST API| OrderService
    OrderService --> |Direct Operations| MongoDB
    IntegrationService --> |Queue Messages| MessageQueue
    IntegrationService --> |REST API| OrderService
    MessageQueue --> |Asynchronous Processing| IntegrationService
    ExternalWebsite --> |Order Data| MessageQueue
    
    %% Deployment relationships
    OrderService -.-> |Deployed via| Docker
    IntegrationService -.-> |Deployed via| Docker
    WebClient -.-> |Deployed via| Docker
    Docker -.-> |Managed by| Chef
    Docker -.-> |Managed by| Puppet
    Docker -.-> |Provisioned by| Azure
```

The component boundaries reflect a three-tier architecture with clear separation between presentation, business logic, and data layers. The Order Service serves as the primary API gateway handling all synchronous operations for orders, quotes, catalog, and dealers, while the Integration Service isolates asynchronous external communications using message queues. MongoDB collections are organized by business domain rather than service boundaries, indicating a monolithic data pattern that suggests natural service boundaries for future microservice decomposition.