```markdown
```mermaid
graph TB
    subgraph "Clients & External Systems"
        WebClient["Web Front-End"]
    end

    subgraph "Application Services"
        style OrderingService fill:#D6EAF8,stroke:#333,stroke-width:2px
        style IntegrationService fill:#D5F5E3,stroke:#333,stroke-width:2px

        OrderingService["<strong>Ordering Service</strong><br>Spring Boot microservice<br><em>Exposes REST APIs for Orders, Quotes, Catalog, etc.</em>"]
        IntegrationService["<strong>Integration Service</strong><br>Spring Boot application<br><em>Handles scheduled data synchronization.</em>"]
    end

    subgraph "External Dependencies"
        AzureQueues[("Azure Cloud Queues")]
        MongoDB[("MongoDB Database")]
        AppInsights[("Application Insights<br>APM & Monitoring")]
    end

    %% Interactions
    WebClient -- "Synchronous API Calls<br>(View Catalog, Manage Quotes)" --> OrderingService
    WebClient -- "1. Submits OrderMessage" --> AzureQueues

    IntegrationService -- "2. Consumes OrderMessage (Scheduled)" --> AzureQueues
    IntegrationService -- "3. Creates Order via REST API" --> OrderingService
    IntegrationService -- "4. Gets Catalog Data via REST API" --> OrderingService
    IntegrationService -- "5. Publishes Product Updates (Scheduled)" --> AzureQueues

    OrderingService -- "CRUD Operations<br>(Persistence)" --> MongoDB

    OrderingService -- "Sends Telemetry" --> AppInsights
    IntegrationService -- "Sends Telemetry" --> AppInsights
```

The architecture is decomposed into two primary components: the `Ordering Service` and the `Integration Service`, reflecting their distinct responsibilities as separate Spring Boot applications. The `Ordering Service` handles core business logic and synchronous request-response interactions via a REST API, while the `Integration Service` manages asynchronous, scheduled background processing. This hybrid communication model uses REST for immediate transactional operations and Azure Cloud Queues for decoupled, resilient workflows like order ingestion and broadcasting product updates.
```