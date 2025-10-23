| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| OrderService | GET | `/catalog` | List all catalog items. |
| OrderService | POST | `/catalog` | Add a new item to the catalog. |
| OrderService | GET | `/catalog/{sku}` | Get a catalog item by its SKU. |
| OrderService | PUT | `/catalog/{sku}` | Update a catalog item by its SKU. |
| OrderService | DELETE | `/catalog/{sku}` | Remove a catalog item by its SKU. |
| OrderService | GET | `/dealers` | List all dealers. |
| OrderService | POST | `/dealers` | Add a new dealer. |
| OrderService | GET | `/dealers/{name}` | Get a dealer by name. |
| OrderService | PUT | `/dealers/{name}` | Update a dealer by name. |
| OrderService | DELETE | `/dealers/{name}` | Remove a dealer by name. |
| OrderService | POST | `/quotes` | Create a new quote. |
| OrderService | GET | `/quotes?name=...` | Get quotes by customer name. |
| OrderService | GET | `/quotes/{quoteId}` | Get a quote by its ID. |
| OrderService | PUT | `/quotes/{quoteId}` | Update a quote by its ID. |
| OrderService | DELETE | `/quotes/{quoteId}` | Delete a quote by its ID. |
| OrderService | POST | `/orders?fromQuote={quoteId}` | Create an order from an existing quote. |
| OrderService | GET | `/orders` | Get orders, filterable by dealer name and/or status. |
| OrderService | GET | `/orders/{orderId}` | Get an order by its ID. |
| OrderService | PUT | `/orders/{orderId}` | Update an order by its ID. |
| OrderService | DELETE | `/orders/{orderId}` | Delete an order by its ID. |
| OrderService | PUT | `/orders/{orderId}/status` | Update the status of a specific order. |
| OrderService | POST | `/orders/{orderId}/events` | Add a comment or event to an order's log. |
| OrderService | POST | `/shipments` | Create a new shipment for an order. |
| OrderService | GET | `/shipments` | Get shipments, filterable by status. |
| OrderService | GET | `/shipments/deliveries` | Get aggregated delivery information (Shipment+Order+Quote). |
| OrderService | GET | `/shipments/{orderId}` | Get a shipment by its associated order ID. |
| OrderService | PUT | `/shipments/{orderId}` | Update a shipment by its associated order ID. |
| OrderService | DELETE | `/shipments/{orderId}` | Delete a shipment by its associated order ID. |
| OrderService | POST | `/shipments/{orderId}/events` | Add an event to a shipment's log. |
| OrderService | GET | `/ping` | Health and build information check. |
| OrderService | HEAD | `/ping` | Health and build information check. |