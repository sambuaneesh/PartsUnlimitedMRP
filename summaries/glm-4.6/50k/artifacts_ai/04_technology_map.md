
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---|---|---|---|---|---|
| Frontend Application | JavaScript | WinJS | N/A (via API) | REST (WinJS.xhr) calls to backend API | MVVM, Single Page Application (SPA) |
| Order Service | Java | Spring Boot | MongoDB | RESTful API (server-side) | Repository, Factory, Template Method, Modular Monolith |
| Integration Service | Java | Spring Boot | N/A (via queues/REST) | Azure Storage Queue, REST client | Queue-Based Messaging, Anti-Corruption Layer, Scheduled Tasks |
| Data Layer | N/A | N/A | MongoDB | Driver communication from Java services (Spring Data) | Document-oriented DB schema |