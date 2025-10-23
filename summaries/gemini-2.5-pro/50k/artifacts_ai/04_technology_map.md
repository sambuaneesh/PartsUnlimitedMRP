| Component Name | Language | Frameworks | Database | Communication | Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Web UI (MRP Client)** | JavaScript, HTML, CSS | WinJS, Date.js | N/A | Synchronous RESTful HTTP (AJAX) | Single Page Application (SPA), MVVM-like, Data Access Layer |
| **Order Service** | Java 8 | Spring Boot, Spring MVC | MongoDB, PostgreSQL, In-memory | Exposes a REST API | Monolith, Repository, Factory, Decorator, Domain/Persistence Model Separation |
| **Integration Service** | Java 8 | Spring Boot | N/A | Asynchronous (Azure Storage Queues), Synchronous RESTful HTTP (as a client) | Adapter / Anti-Corruption Layer, Polling Consumer |
| **Infrastructure & DevOps Tooling** | Shell, PowerShell, Python, JSON, YAML, Ruby | Ansible, Chef, Puppet, Docker, Azure ARM, Travis CI, Jenkins, Locust | N/A | N/A (Orchestration & deployment) | Infrastructure as Code (IaC), Configuration Management (CM), CI/CD |