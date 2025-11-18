
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|----------------|----------|------------|----------|---------------|----------|
| Order Service | Java 8 | Spring Boot, Application Insights, Gradle | MongoDB (configurable with in-memory option) | REST API (Port 8080) | Repository Pattern, Factory Pattern, Validation Pipeline |
| Integration Service | Java 8 | Spring Boot, Azure Storage SDK, Jackson | Transient message data | REST API, Azure Queue Service | Queue-based Processing, Scheduled Tasks, Data Transformation |
| Frontend Application | JavaScript | WinJS (Windows Library for JavaScript) | N/A (client-side) | REST API calls to backend | Single Page Application, Two-way Data Binding, Optimistic UI Updates |
| Data Storage Layer | N/A | MongoDB | MongoDB Collections (catalog, dealers, quotes, orders, shipments) | Direct DB connection | Document-oriented Storage, Collection-based Organization |
| API Layer | Java/JavaScript | Spring Boot Controllers, REST endpoints | MongoDB via services | REST/HTTP | RESTful Design, CRUD Operations, Status Code Standardization |