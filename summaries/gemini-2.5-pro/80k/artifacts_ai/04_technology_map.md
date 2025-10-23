| Component Name | Language | Frameworks | Database | Communication | Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Clients/Web (Frontend SPA)** | JavaScript, HTML, CSS | WinJS | N/A | REST (AJAX) | Single-Page Application (SPA), Page Control, Facade / Data Access Layer (DAL), Client-side Joins |
| **OrderService (Backend Monolith)** | Java | Spring Boot, Spring Data | MongoDB | REST API | Monolith, Layered Architecture, Repository Pattern, Retry |
| **IntegrationService (Backend Integrator)** | Java | Spring Boot, Microsoft Azure Storage SDK | N/A | Message Queues (Azure), REST | Adapter / Anti-Corruption Layer, Scheduled Polling, Asynchronous Communication |