| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| Ordering Service | GET | `/ping` | Provides a basic liveness check to confirm the service is responsive. |
| Ordering Service | GET | `/status` | Retrieves detailed service status, including configuration and build information. |
| Ordering Service | GET | `/catalog` | Retrieves a list of all catalog items (parts). |
| Ordering Service | GET | `/catalog/{sku}` | Fetches a specific catalog item by its SKU. |
| Ordering Service | POST | `/catalog` | Adds a new item to the parts catalog. |
| Ordering Service | PUT | `/catalog/{sku}` | Updates an existing catalog item or creates it if it doesn't exist (upsert). |
| Ordering Service | DELETE | `/catalog/{sku}` | Removes a catalog item from the parts catalog. |
| Ordering Service | GET | `/quotes` | Searches for and retrieves quotes, typically filtered by customer name. |
| Ordering Service | GET | `/quotes/{id}` | Retrieves a specific quote by its unique ID. |
| Ordering Service | POST | `/quotes` | Creates a new sales quote. |
| Ordering Service | PUT | `/quotes/{id}` | Updates an existing sales quote. |
| Ordering Service | DELETE | `/quotes/{id}` | Deletes a sales quote by its ID. |
| Ordering Service | GET | `/orders` | Retrieves a list of orders, with optional filtering by dealer name and/or status. |
| Ordering Service | GET | `/orders/{id}` | Retrieves a specific order by its unique ID. |
| Ordering Service | POST | `/orders` | Creates a new order, often by converting an existing quote. |
| Ordering Service | PUT | `/orders/{id}` | Updates an existing order's details or status. |
| Ordering Service | PUT | `/orders/{id}/events` | Adds a new chronological event to an order's history log. |
| Ordering Service | DELETE | `/orders/{id}` | Deletes an order by its ID. |
| Ordering Service | GET | `/shipments` | Retrieves a list of shipments, with optional filtering by status. |
| Ordering Service | GET | `/shipments/{id}` | Retrieves a specific shipment record by its ID. |
| Ordering Service | POST | `/shipments` | Creates a new shipment record for an order. |
| Ordering Service | PUT | `/shipments/{id}/events` | Adds a new event (e.g., tracking update) to a shipment's history. |
| Ordering Service | DELETE | `/shipments/{id}` | Deletes a shipment record. |
| Ordering Service | GET | `/deliveries` | Retrieves an aggregated view of confirmed deliveries, linking shipments, orders, and quotes. |
| Ordering Service | GET | `/dealers` | Retrieves a list of all dealers or suppliers. |
| Ordering Service | GET | `/dealers/{name}` | Fetches specific dealer details by name. |
| Ordering Service | POST | `/dealers` | Adds a new dealer. |
| Ordering Service | PUT | `/dealers/{name}` | Updates an existing dealer's information. |
| Ordering Service | DELETE | `/dealers/{name}` | Removes a dealer by name. |