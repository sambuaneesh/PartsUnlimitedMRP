```mermaid
graph TB
    subgraph User Facing
        WebUI["Web UI (Static JS/HTML on Tomcat)"]
    end

    subgraph Backend Services
        OrderService["Order Service (Java/Spring)"]
        IntegrationService["Integration Service (Java)"]
    end

    subgraph Data Stores
        MongoDB["MongoDB"]
        AzureQueues["Azure Storage Queues"]
    end
    
    subgraph External
        ExternalWebsite["External PartsUnlimited Website"]
    end

    WebUI -- "REST API Calls" --> OrderService
    OrderService -- "Reads/Writes" --> MongoDB
    
    IntegrationService -- "Pulls Order Messages" --> AzureQueues
    AzureQueues -- "Pushes Order Messages" --> IntegrationService
    IntegrationService -- "Creates Orders / Gets Catalog (REST)" --> OrderService
    IntegrationService -- "Pushes Inventory Updates" --> AzureQueues

    ExternalWebsite -.->|Sends Orders| AzureQueues
    AzureQueues -.->|Receives Inventory| ExternalWebsite
```

The architecture consists of a client-side Web UI that interacts with a backend Order Service via a REST API for core operations like managing quotes, orders, and catalog items. This Order Service uses MongoDB as its primary data store. A separate Integration Service handles asynchronous communication with an external website, using Azure Storage Queues to decouple the systems for processing incoming orders and sending out inventory updates.