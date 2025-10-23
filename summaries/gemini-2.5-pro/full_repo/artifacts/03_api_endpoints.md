| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| OrderService | GET | /catalog | Retrieves all catalog items. |
| OrderService | GET | /catalog/{sku} | Retrieves a specific catalog item by its SKU. |
| OrderService | POST | /catalog | Adds a new catalog item. |
| OrderService | PUT | /catalog/{sku} | Updates an existing catalog item by its SKU. |
| OrderService | DELETE | /catalog/{sku} | Deletes a catalog item by its SKU. |
| OrderService | GET | /dealers | Retrieves all dealers. |
| OrderService | GET | /dealers/{name} | Retrieves a specific dealer by name. |
| OrderService | POST | /dealers | Adds a new dealer. |
| OrderService | PUT | /dealers/{name} | Updates an existing dealer by name. |
| OrderService | DELETE | /dealers/{name} | Deletes a dealer by name. |
| OrderService | GET | /orders/{orderId} | Retrieves a specific order by its ID. |
| OrderService | GET | /orders | Retrieves a list of orders, with optional filters for dealer and status. |
| OrderService | POST | /orders | Creates a new order from a quote ID. |
| OrderService | POST | /orders/{orderId}/events | Adds an event to a specific order. |
| OrderService | PUT | /orders/{orderId} | Updates a specific order. |
| OrderService | PUT | /orders/{orderId}/status | Updates the status of a specific order. |
| OrderService | DELETE | /orders/{orderId} | Deletes a specific order by its ID. |
| OrderService | HEAD | /ping | Checks if the service is available. |
| OrderService | GET | /ping | Retrieves service status and build information. |
| OrderService | GET | /quotes/{quoteId} | Retrieves a specific quote by its ID. |
| OrderService | GET | /quotes | Retrieves quotes filtering by customer name. |
| OrderService | POST | /quotes | Creates a new quote. |
| OrderService | PUT | /quotes/{quoteId} | Updates an existing quote by its ID. |
| OrderService | DELETE | /quotes/{quoteId} | Deletes a specific quote by its ID. |
| OrderService | GET | /shipments | Retrieves shipments, with an optional filter for status. |
| OrderService | GET | /shipments/deliveries | Retrieves detailed delivery information (shipment, order, and quote). |
| OrderService | GET | /shipments/{id} | Retrieves a specific shipment by its order ID. |
| OrderService | POST | /shipments | Creates a new shipment record for an order. |
| OrderService | PUT | /shipments/{id} | Updates an existing shipment record by its order ID. |
| OrderService | POST | /shipments/{id}/events | Adds an event to a specific shipment record. |
| OrderService | DELETE | /shipments/{orderId} | Deletes a specific shipment record by its order ID. |