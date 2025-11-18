
```mermaid
graph TB
    subgraph "Client Layer"
        WC[WinJS Web Client<br/>SPA Application]
    end
    
    subgraph "Web Server"
        TC[Tomcat Server<br/>Port 9080]
    end
    
    subgraph "Backend Services"
        OS[OrderService<br/>Spring Boot<br/>Port 8080]
        IS[IntegrationService<br/>Spring Boot]
    end
    
    subgraph "Data Store"
        DB[(MongoDB<br/>Port 27017)]
    end
    
    subgraph "External Systems"
        PU[Parts Unlimited Website]
        AZ[Azure Storage Queues]
    end
    
    WC -->|HTTP/REST API| TC
    TC -->|Serves WAR| WC
    WC -->|Direct API Calls| OS
    OS -->|Repository Pattern| DB
    IS -->|Queue Messages| AZ
    AZ -->|Order Messages| IS
    PU -->|Product Updates| AZ
    IS -->|REST Calls| OS
    
    OS -->|Controllers:<br/>• Catalog<br/>• Orders<br/>• Quotes<br/>• Dealers<br/>• Shipments| DB
```

The architecture maintains clear separation between the WinJS SPA frontend, Tomcat web server, and Spring Boot services, with MongoDB as the shared data store. Communication uses synchronous REST APIs for direct interactions and asynchronous Azure queues for decoupled integration with the external website. The OrderService encapsulates core business logic through domain-specific controllers while the IntegrationService acts as an anti-corruption layer handling message transformation.