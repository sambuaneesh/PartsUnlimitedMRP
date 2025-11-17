```markdown
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---------------|----------|------------|----------|---------------|----------|
| integration | Java | Spring Boot | None | Azure Queue Storage, Background Processing | Scheduled Task Pattern, Centralized Configuration Pattern |
| integration.services | Java | Spring Boot, Azure Storage SDK, Jackson | None | Azure Queue Storage, REST APIs (RestTemplate) | Factory Pattern, Service Layer Pattern, Hybrid Communication Strategy |
| integration.models | Java | Jackson | None | Cloud Queue Messaging | Data Transfer Object (DTO) Pattern, Generic Type Safety |
| integration.infrastructure | Java | None | None | Configuration Access | Centralized Configuration Pattern, Layered Abstraction Pattern |
| integration.scheduled | Java | Spring Boot Scheduling | None | Azure Queue Storage, REST APIs | Message-Based Integration Pattern, Scheduled Task Pattern, Background Processing Architecture |
| integration.models.website | Java | Jackson | None | REST API (JSON) | DTO Pattern, Layered Architecture, Message-Based Integration |
| integration.models.mrp | Java | None | None | Internal Data Exchange | DTO Pattern, Domain-Driven Design, Integration-First Architecture |
| smpl.ordering | Java | Spring Boot, Application Insights | MongoDB, PostgreSQL | REST API, Telemetry | Externalized Configuration Pattern, Factory Pattern, Cross-Cutting Concern Separation |
| smpl.ordering.controllers | Java | Spring Boot MVC, Application Insights | None | REST API (HTTP) | RESTful Resource-Oriented Architecture, Controller Pattern, Test-Driven Development |
| smpl.ordering.models | Java | None | None | Domain Object Exchange | Domain-Driven Design, JavaBean Convention, Event Sourcing Pattern, Aggregate Root Design |
| smpl.ordering.repositories | Java | Spring Data (implied) | MongoDB, In-Memory | Repository Interface | Repository Pattern, Factory Pattern, Optimistic Concurrency Control |
| smpl.ordering.repositories.mongodb | Java | Spring Data MongoDB, Application Insights | MongoDB | MongoDB Driver | Repository Pattern, Resilience Pattern, Domain-Driven Design Alignment |
| smpl.ordering.repositories.mock | Java | None | In-Memory | Mock Data Access | Repository Pattern, Mock Object Pattern, Defensive Programming |
| smpl.ordering.repositories.mongodb.models | Java | Spring Data MongoDB | MongoDB | MongoDB Document Mapping | Data Mapper Pattern, Domain-Driven Design, Event Sourcing Influence |
| smpl.ordering.repositories.mock.test | Java | JUnit, Spring Test | In-Memory | Test Configuration | Inheritance Pattern, Repository Factory Configuration, Test-First Architecture |
| smpl.ordering.repositories.mongodb.test | Java | JUnit, Spring Test, MongoDB | MongoDB | MongoDB Integration | Integration Test Pattern, Template Method Pattern, Marker Interface Pattern |
```