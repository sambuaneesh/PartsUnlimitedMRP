
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---|---|---|---|---|---|
| Frontend Web Service | JavaScript, HTML, CSS | Apache Tomcat 7 | N/A | REST calls to Order Service | Static web pages, SPA-like structure |
| Order Service Backend | Java 8 | Spring Boot | MongoDB | RESTful APIs | Repository Pattern, REST API, Microservice |
| Integration Service | Java 8 | Spring Boot, Azure Storage SDK | N/A | REST API calls, Azure Queue Storage | Queue-based Messaging, Scheduled Tasks |
| Database Layer | N/A | MongoDB | MongoDB | N/A | Document Store |
| WinJS Web Client | JavaScript, HTML, CSS | WinJS | N/A | REST calls to backend | Single Page Application (SPA) |
| Dealer Management Service | JavaScript | WinJS | N/A | REST APIs | CRUD operations, ListView pattern |
| Quote Service | JavaScript | WinJS | N/A | REST APIs | Pricing management, popup dialogs |
| Order Service | JavaScript | WinJS | N/A | REST APIs | Lifecycle management, status flow |
| Delivery Service | JavaScript | WinJS | N/A | REST APIs | Delivery tracking, contact management |
| Catalog Service | JavaScript | WinJS | N/A | REST APIs | Product management, SKU lookup |
| Event Management Service | JavaScript | WinJS | N/A | N/A | Timeline tracking, chronological sorting |