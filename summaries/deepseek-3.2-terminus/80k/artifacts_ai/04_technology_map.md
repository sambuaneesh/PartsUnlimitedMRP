```markdown
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---------------|----------|------------|----------|---------------|----------|
| OrderService | Java | Spring Boot, Spring Data MongoDB, Spring Web, Jackson, Application Insights, JUnit | MongoDB (primary), In-memory (testing) | REST API (synchronous) | Repository Pattern, MVC, Factory Pattern, RESTful API, Service Layer, Filter Pattern |
| IntegrationService | Java | Spring Boot, Azure Storage SDK, Application Insights | None (uses OrderService via API) | Azure Storage Queues (asynchronous), REST API (synchronous) | Message Queue Pattern, Scheduled Task Pattern, Background Processing |
| Web Client | JavaScript, HTML5, CSS3 | WinJS, Google Maps Places API | None (client-side only) | REST API (synchronous to OrderService) | MVVM Pattern, Single Page Application, Component-based UI, Two-way Data Binding |
```