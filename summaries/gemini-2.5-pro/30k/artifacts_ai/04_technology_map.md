| Component Name | Language | Frameworks | Database | Communication | Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Frontend Service** | JavaScript, HTML5, CSS3 | Microsoft WinJS | N/A | RESTful HTTP | Single Page Application (SPA), Master-Detail UI |
| **Order Service** | Java 8 | Spring Boot, Spring MVC, Spring Data MongoDB | MongoDB | Exposes RESTful HTTP API | Layered Architecture, Repository, Factory, Decorator |
| **Integration Service** | Java 8 | Spring Boot | MongoDB (Shared) | Asynchronous (Azure Storage Queues), Synchronous (REST API calls) | Anti-Corruption Layer (ACL), Queue-Based Decoupling, Scheduled Polling |