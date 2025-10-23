| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| OrderService | HEAD | `/api/ping` | Performs a basic health check. |
| OrderService | GET | `/api/ping` | Retrieves detailed service status and build information. |
| OrderService | GET | `/api/catalog` | Retrieves a list of all catalog items. |
| OrderService | POST | `/api/catalog` | Adds a new catalog item. |
| OrderService | GET | `/api/catalog/{sku}` | Retrieves a single catalog item by its SKU. |
| OrderService | PUT | `/api/catalog/{sku}` | Updates an existing catalog item or creates it if it doesn't exist (Upsert). |
| OrderService | DELETE | `/api/catalog/{sku}` | Removes a catalog item by its SKU. |
| OrderService | GET | `/api/dealers` | Retrieves a list of all dealers. |
| OrderService | POST | `/api/dealers` | Adds a new dealer. |
| OrderService | GET | `/api/dealers/{name}` | Retrieves a single dealer by name. |
| OrderService | PUT | `/api/dealers/{name}` | Updates an existing dealer by name. |
| OrderService | DELETE | `/api/dealers/{name}` | Removes a dealer by name. |
| OrderService | GET | `/api/quotes` | Retrieves quotes, optionally filtered by customer name (`?name=...`). |
| OrderService | POST | `/api/quotes` | Creates a new quote. |
| OrderService | GET | `/api/quotes/{quoteId}` | Retrieves a single quote by its ID. |
| OrderService | PUT | `/api/quotes/{quoteId}` | Updates an existing quote by its ID. |
| OrderService | DELETE | `/api/quotes/{quoteId}` | Deletes a quote by its ID. |
| OrderService | GET | `/api/orders` | Retrieves orders, optionally filtered by dealer and status (`?dealer=...&status=...`). |
| OrderService | POST | `/api/orders` | Creates a new order from an existing quote, specified by query parameter (`?fromQuote=...`). |
| OrderService | GET | `/api/orders/{orderId}` | Retrieves a single order by its ID. |
| OrderService | PUT | `/api/orders/{orderId}` | Updates an entire order object. |
| OrderService | DELETE | `/api/orders/{orderId}` | Deletes an order by its ID. |
| OrderService | POST | `/api/orders/{orderId}/events` | Adds a history event (e.g., a comment) to an order. |
| OrderService | PUT | `/api/orders/{orderId}/status` | Updates the status of an order and adds a corresponding history event. |
| OrderService | GET | `/api/shipments` | Retrieves shipment records, optionally filtered by the associated order's status (`?status=...`). |
| OrderService | POST | `/api/shipments` | Creates a new shipment record for an order. |
| OrderService | GET | `/api/shipments/deliveries` | Retrieves a list of aggregated delivery objects, joining shipment, order, and quote data. |
| OrderService | GET | `/api/shipments/{orderId}` | Retrieves the shipment record for a given order ID. |
| OrderService | PUT | `/api/shipments/{orderId}` | Updates an existing shipment record. |
| OrderService | DELETE | `/api/shipments/{orderId}` | Removes a shipment record by its associated order ID. |
| OrderService | POST | `/api/shipments/{orderId}/events` | Adds a tracking event to a shipment's history. |