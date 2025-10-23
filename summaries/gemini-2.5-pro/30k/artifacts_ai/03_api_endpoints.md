| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| Order Service | GET | `/api/ping` | Performs a health check of the service. |
| Order Service | GET | `/api/catalog` | Retrieves a list of all catalog items. |
| Order Service | POST | `/api/catalog` | Creates a new catalog item. |
| Order Service | GET | `/api/catalog/{id}` | Retrieves a single catalog item by its ID. |
| Order Service | PUT | `/api/catalog/{id}` | Updates an existing catalog item. |
| Order Service | DELETE | `/api/catalog/{id}` | Deletes a catalog item. |
| Order Service | GET | `/api/dealers` | Retrieves a list of all dealers. |
| Order Service | POST | `/api/dealers` | Creates a new dealer. |
| Order Service | GET | `/api/dealers/{id}` | Retrieves a single dealer by its ID. |
| Order Service | PUT | `/api/dealers/{id}` | Updates an existing dealer. |
| Order Service | DELETE | `/api/dealers/{id}` | Deletes a dealer. |
| Order Service | GET | `/api/quotes` | Retrieves a list of all quotes, with support for filtering by name. |
| Order Service | POST | `/api/quotes` | Creates a new quote. |
| Order Service | GET | `/api/quotes/{id}` | Retrieves a single quote by its ID. |
| Order Service | PUT | `/api/quotes/{id}` | Updates an existing quote. |
| Order Service | DELETE | `/api/quotes/{id}` | Deletes a quote. |
| Order Service | GET | `/api/orders` | Retrieves a list of orders, with filtering by dealer and status. |
| Order Service | POST | `/api/orders?fromQuote={quoteId}` | Creates a new order from an existing quote. |
| Order Service | GET | `/api/orders/{orderId}` | Retrieves a single order by its ID. |
| Order Service | PUT | `/api/orders/{orderId}` | Updates an existing order. |
| Order Service | DELETE | `/api/orders/{orderId}` | Deletes an order. |
| Order Service | POST | `/api/orders/{orderId}/events` | Adds a historical event to an order. |
| Order Service | PUT | `/api/orders/{orderId}/status` | Updates the status of an order and logs a corresponding event. |
| Order Service | GET | `/api/shipments` | Retrieves a list of all shipment records. |
| Order Service | POST | `/api/shipments/{orderId}` | Creates a shipment record for a given order ID. |
| Order Service | GET | `/api/shipments/{orderId}` | Retrieves the shipment record for a given order ID. |
| Order Service | PUT | `/api/shipments/{orderId}` | Updates the shipment record for a given order ID. |
| Order Service | DELETE | `/api/shipments/{orderId}` | Deletes the shipment record for a given order ID. |
| Order Service | POST | `/api/shipments/{orderId}/events` | Adds a historical event to a shipment. |
| Order Service | GET | `/api/shipments/deliveries` | Retrieves an aggregated list of deliveries, joining shipment, order, and quote data. |