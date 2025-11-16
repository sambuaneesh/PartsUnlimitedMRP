```mermaid
graph TB
    FrontendClient[Frontend Client<br/>WinJS SPA] -->|REST API| OrderService[Order Service<br/>Spring Boot]
    
    OrderService -->|Data Persistence| MongoDB[(MongoDB<br/>Primary Database)]
    OrderService -->|Health Monitoring| AppInsights[Application Insights]
    
    IntegrationService[Integration Service<br/>Spring Boot] -->|Queue Processing| AzureQueues[Azure Storage Queues]
    IntegrationService -->|REST API| ExternalMRP[External MRP System]
    
    AzureQueues -->|Order Messages| IntegrationService
    AzureQueues -->|Product Updates| IntegrationService
    
    FrontendClient -->|Google Maps API| GoogleMaps[Google Maps API]
    
    OrderService -->|Alternative Storage| PostgreSQL[(PostgreSQL<br/>Alternative DB)]
    
    subgraph Infrastructure
        ARMTemplates[ARM Templates<br/>Azure Deployment]
        Ansible[Ansible Playbooks<br/>VM Management]
    end

```

The component boundaries reflect clear separation between the frontend SPA, core business services, and integration layer. The Frontend Client communicates synchronously with OrderService via REST APIs for all business operations, while IntegrationService handles asynchronous queue-based communication with external systems. OrderService maintains data sovereignty over all core business entities, with MongoDB as the primary data store and PostgreSQL as an alternative option.