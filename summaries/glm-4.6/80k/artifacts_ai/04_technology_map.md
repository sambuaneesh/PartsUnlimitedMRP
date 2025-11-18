
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---|---|---|---|---|---|
| Frontend Web Service | JavaScript | WinJS, Apache Tomcat 7 | None (via REST APIs) | REST API | SPA, Page-based navigation, Data binding |
| Order Service | Java 8 | Spring Boot | MongoDB | REST APIs | Repository, Cross-cutting concerns, Custom exceptions |
| Integration Service | Java 8 | Spring Boot, Azure Storage SDK | None | REST APIs, Azure Queue Storage | Scheduled tasks, Queue-based messaging |
| Repository Layer | Java | Spring Data, Application Insights | MongoDB, Mock | N/A | Repository, Factory, Interface-based design, Retry mechanism |
| MongoDB Database | N/A | N/A | MongoDB | N/A | Document schema, Collection-based storage |
| Docker Containers | Shell scripts | Docker | MongoDB container | Exposed ports (27017, 28017, 8080, 80) | Multi-container, Docker networking |
| Azure Deployment | ARM templates | Azure Stack, NSG | Azure Storage | SSH (22), HTTP (80/443), Application ports (9080, 8080) | Infrastructure as Code, Network Security Groups |