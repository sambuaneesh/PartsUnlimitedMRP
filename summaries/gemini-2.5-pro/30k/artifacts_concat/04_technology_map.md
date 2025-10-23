| Component Name | Language | Frameworks | Database | Communication | Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Clients/Web** | JavaScript, HTML5, CSS3 | WinJS, Apache Tomcat (for serving) | N/A | RESTful HTTP (to OrderService), Google Maps Places API | Single Page Application (SPA), Master-Detail UI, Data Access Layer |
| **OrderService** | Java | Spring Boot, Spring MVC, Spring Data MongoDB | MongoDB | Exposes RESTful HTTP API, MongoDB Wire Protocol | Layered Architecture, RESTful Service, Repository, Factory, Decorator, Dependency Injection |
| **IntegrationService** | Java | Spring Boot, Azure Storage SDK | N/A | Asynchronous (Azure Storage Queues), RESTful HTTP (to OrderService) | Anti-Corruption Layer (ACL), Queue-Based Decoupling, Scheduled Polling |