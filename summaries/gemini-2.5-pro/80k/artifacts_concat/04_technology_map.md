| Component Name | Language | Frameworks | Database | Communication | Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Clients/Web** | JavaScript, HTML, CSS | WinJS | N/A | REST (HTTP/AJAX) | Single-Page Application (SPA), Page Control, Facade (Client-side DAL) |
| **OrderService** | Java | Spring Boot, Spring Data MongoDB | MongoDB | REST (HTTP) | Monolith, Layered Architecture, Repository Pattern |
| **IntegrationService** | Java | Spring Boot, Azure Storage SDK | N/A | REST (HTTP Client), Message Queues (Azure Storage) | Adapter / Anti-Corruption Layer, Asynchronous Polling Consumer |