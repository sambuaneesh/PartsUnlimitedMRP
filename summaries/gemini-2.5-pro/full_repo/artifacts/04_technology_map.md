| Component Name | Language | Frameworks | Database | Communication | Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Web Client** | JavaScript, HTML, CSS | WinJS, Apache Tomcat (Server) | N/A | RESTful HTTP/JSON | Single-Page Application (SPA) |
| **Order Service** | Java, Groovy | Spring Boot, Spring Web MVC, Spring Data MongoDB | MongoDB | RESTful HTTP/JSON (Exposes API) | Repository, Dependency Injection, MVC |
| **Integration Service** | Java | Spring Boot | N/A | RESTful HTTP/JSON (Consumes), Azure Storage Queues | Scheduled Task, Message Queue Polling |