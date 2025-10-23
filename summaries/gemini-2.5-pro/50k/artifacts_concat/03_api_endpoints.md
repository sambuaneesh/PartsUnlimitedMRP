| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| Order Service | GET | `/ping` | Health check endpoint, may also return build information. |
| Order Service | GET | `/catalog` | Retrieves a list of all catalog items. |
| Order Service | POST | `/catalog` | Creates a new catalog item. |
| Order Service | GET | `/catalog/{sku}` | Retrieves a single catalog item by its SKU number. |
| Order Service | PUT | `/catalog/{sku}` | Updates an existing catalog item. |
| Order Service | DELETE | `/catalog/{sku}` | Deletes a catalog item. |
| Order Service | GET | `/dealers` | Retrieves a list of all dealers. |
| Order Service | POST | `/dealers` | Creates a new dealer. |
| Order Service | GET | `/dealers/{name}` | Retrieves a single dealer by name. |
| Order Service | PUT | `/dealers/{name}` | Updates an existing dealer. |
| Order Service | DELETE | `/dealers/{name}` | Deletes a dealer. |
| Order Service | GET | `/quotes` | Retrieves a list of all quotes, can be filtered by customer name (`?name=...`). |
| Order Service | POST | `/quotes` | Creates a new quote. |
| Order Service | GET | `/quotes/{quoteId}` | Retrieves a single quote by its ID. |
| Order Service | PUT | `/quotes/{quoteId}` | Updates an existing quote. |
| Order Service | DELETE | `/quotes/{quoteId}` | Deletes a quote. |
| Order Service | GET | `/orders` | Retrieves a list of all orders, filterable by dealer (`?dealer=...`) and status (`?status=...`). |
| Order Service | POST | `/orders?fromQuote={quoteId}` | Creates a new order from an existing quote. |
| Order Service | GET | `/orders/{orderId}` | Retrieves a single order by its ID. |
| Order Service | PUT | `/orders/{orderId}` | Updates the general information of an existing order. |
| Order Service | DELETE | `/orders/{orderId}` | Deletes an order. |
| Order Service | PUT | `/orders/{orderId}/status` | Updates the status of an existing order and adds a history event. |
| Order Service | POST | `/orders/{orderId}/events` | Adds a manual event to an order's history. |
| Order Service | GET | `/shipments` | Retrieves a list of all shipments, filterable by status (`?status=...`). |
| Order Service | POST | `/shipments` | Creates a new shipment record for an order. |
| Order Service | GET | `/shipments/{orderId}` | Retrieves a single shipment by its order ID. |
| Order Service | PUT | `/shipments/{orderId}` | Updates an existing shipment record. |
| Order Service | DELETE | `/shipments/{orderId}` | Deletes a shipment record. |
| Order Service | POST | `/shipments/{orderId}/events` | Adds an event to a shipment's history. |
| Order Service | GET | `/shipments/deliveries` | Retrieves aggregated delivery information (Shipment+Order+Quote). |