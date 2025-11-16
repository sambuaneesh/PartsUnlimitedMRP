```markdown
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---------------|----------|------------|----------|---------------|----------|
| OrderService | Java | Spring Boot, Spring Data MongoDB, Spring Web, Jackson, Application Insights | MongoDB (primary), PostgreSQL (alternative) | REST API, Application Insights telemetry | Repository Pattern, MVC, Factory Pattern, Service Layer, Configuration Properties, Filter Pattern |
| IntegrationService | Java | Spring Boot, Azure Storage SDK, Application Insights | None (stateless) | Azure Storage Queues, REST API calls to OrderService | Message Queue Pattern, Scheduled Task Pattern, REST Client |
| Web Client (Frontend) | HTML5, CSS3, JavaScript | WinJS, Google Maps Places API | None (client-side only) | REST API calls to OrderService, Google Maps API | MVVM Pattern, Single Page Application, Component-based UI, Client-side Routing |
```