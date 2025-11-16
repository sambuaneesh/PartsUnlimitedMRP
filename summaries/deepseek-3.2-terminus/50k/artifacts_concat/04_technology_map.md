```markdown
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---------------|----------|------------|----------|---------------|----------|
| OrderService | Java | Spring Boot, Spring Data MongoDB, Application Insights | MongoDB (primary), PostgreSQL, In-Memory (testing) | REST APIs, Application Insights (monitoring) | Microservices, Repository Pattern, MVC, Factory Pattern, RESTful APIs |
| IntegrationService | Java | Spring Boot, Azure Storage Queue SDK, Application Insights | Azure Storage Queues (messaging) | Azure Storage Queues (async), REST APIs (sync), Scheduled Tasks | Microservices, Queue-based Integration, Scheduled Task Pattern, Factory Pattern |
| Web Client | JavaScript | WinJS, Single Page Application (SPA) architecture | None (relies on backend services) | REST APIs (to OrderService), Google Maps Places API | MVVM (via WinJS.Binding), SPA, Navigation Pattern, Event Delegation |
| Infrastructure (IaC) | JSON (ARM), YAML (Ansible) | Azure Resource Manager, Ansible | N/A | SSH, Azure Resource Manager API | Infrastructure as Code, Configuration Management, Load Balancing |
| MongoDB Database | N/A | Spring Data MongoDB, MongoDB Driver | MongoDB | TCP/IP connections from services | Document Store, Repository Pattern |
| Azure Storage Queues | N/A | Azure Storage SDK | Azure Storage Queues | HTTP/REST API | Message Queue, Async Communication, Producer-Consumer |
```