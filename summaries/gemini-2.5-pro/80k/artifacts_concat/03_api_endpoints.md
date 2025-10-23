| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| OrderService | GET | `/catalog` | Get all catalog items. |
| OrderService | POST | `/catalog` | Add a new catalog item. |
| OrderService | GET | `/catalog/{sku}` | Get a catalog item by SKU. |
| OrderService | PUT | `/catalog/{sku}` | Update a catalog item. |
| OrderService | DELETE | `/catalog/{sku}` | Remove a catalog item. |
| OrderService | GET | `/dealers` | Get all dealers. |
| OrderService | POST | `/dealers` | Add a new dealer. |
| OrderService | GET | `/dealers/{name}` | Get a dealer by name. |
| OrderService | PUT | `/dealers/{name}` | Update a dealer. |
| OrderService | DELETE | `/dealers/{name}` | Remove a dealer. |
| OrderService | GET | `/quotes` | Get quotes, filterable by customer name. |
| OrderService | POST | `/quotes` | Create a new quote. |
| OrderService | GET | `/quotes/{quoteId}` | Get a quote by ID. |
| OrderService | PUT | `/quotes/{quoteId}` | Update a quote. |
| OrderService | DELETE | `/quotes/{quoteId}` | Delete a quote. |
| OrderService | GET | `/orders` | Get orders, filterable by dealer name and/or status. |
| OrderService | POST | `/orders` | Create an order from a quote (requires `fromQuote` query param). |
| OrderService | GET | `/orders/{orderId}` | Get an order by ID. |
| OrderService | PUT | `/orders/{orderId}` | Update an entire order object. |
| OrderService | DELETE | `/orders/{orderId}` | Delete an order. |
| OrderService | POST | `/orders/{orderId}/events` | Add a comment or event to an order. |
| OrderService | PUT | `/orders/{orderId}/status` | Update an order's status. |
| OrderService | GET | `/shipments` | Get shipments, filterable by order status. |
| OrderService | POST | `/shipments` | Create a shipment record for an order. |
| OrderService | GET | `/shipments/deliveries` | Get aggregated delivery information (Shipment+Order+Quote). |
| OrderService | GET | `/shipments/{id}` | Get a shipment by its associated order ID. |
| OrderService | PUT | `/shipments/{id}` | Update a shipment record. |
| OrderService | DELETE | `/shipments/{orderId}` | Delete a shipment record by its associated order ID. |
| OrderService | POST | `/shipments/{id}/events` | Add an event to a shipment. |
| OrderService | GET | `/ping` | Health and build information check. |
| OrderService | HEAD | `/ping` | Health and build information check (headers only). |