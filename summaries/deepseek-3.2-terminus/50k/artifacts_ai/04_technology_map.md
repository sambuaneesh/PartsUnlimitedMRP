```markdown
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---------------|----------|------------|----------|---------------|----------|
| OrderService | Java | Spring Boot, Spring Data MongoDB, Application Insights | MongoDB, PostgreSQL | REST API (Port 8080), Spring Boot Actuator (Port 8081) | Repository Pattern, Factory Pattern, MVC, Microservice |
| IntegrationService | Java | Spring Boot, Azure Storage SDK, Gradle | Azure Storage Queues | Azure Storage Queues, REST API, Scheduled Tasks | Queue-based Messaging, Scheduled Task Pattern, Factory Pattern |
| Frontend Client | JavaScript, HTML, CSS | WinJS, Google Maps API | None (REST API to OrderService) | REST API calls to OrderService | MVVM, Single Page Application (SPA), Page-based Navigation |
| MongoDB Database | N/A | Spring Data MongoDB | MongoDB Collections (Catalog, Dealers, Quotes, Orders, Shipments) | TCP/IP connections from OrderService | Document Storage, Repository Pattern |
| Azure Storage Queues | N/A | Azure Storage SDK | Azure Storage Queues (orders, product) | HTTP/REST API from IntegrationService | Message Queue, Asynchronous Communication |
| Infrastructure (ARM/Ansible) | JSON, YAML | Azure Resource Manager, Ansible | N/A | SSH, Azure REST API | Infrastructure as Code, Declarative Configuration |
```