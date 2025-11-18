
| Component Name | Language | Frameworks | Database | Communication | Patterns |
|---|---|---|---|---|---|
| Frontend Web Client | JavaScript | WinJS, DateJS | N/A | REST (HTTP/JSON), Google Maps API | SPA (Single Page Application), MVVM, Component-based Architecture |
| Catalog Service | Java | Spring Boot, Spring Data MongoDB | MongoDB | REST (HTTP/JSON) | Repository, RESTful API |
| Dealer Service | Java | Spring Boot, Spring Data MongoDB | MongoDB | REST (HTTP/JSON) | Repository, RESTful API |
| Quote Service | Java | Spring Boot, Spring Data MongoDB | MongoDB | REST (HTTP/JSON) | Repository, RESTful API, State Machine (for expiry) |
| Order Service | Java | Spring Boot, Spring Data MongoDB | MongoDB | REST (HTTP/JSON) | Repository, Factory, Layered Architecture, RESTful API, State Machine (for status workflow) |
| Shipment Service | Java | Spring Boot, Spring Data MongoDB | MongoDB | REST (HTTP/JSON) | Repository, RESTful API, Event Logging (for tracking) |
| Integration Service | Java | Spring Boot | N/A | REST (HTTP/JSON), Azure Service Bus / Azure Storage Queues | Messaging, Anti-corruption Layer, Scheduled Tasks |
| CI/CD & Infrastructure | YAML, JSON, Python, Shell | Gradle, Travis CI, Jenkins, Docker, Ansible, Chef, Puppet, Azure ARM | N/A | SSH, REST APIs | Infrastructure as Code (IaC), Configuration Management |