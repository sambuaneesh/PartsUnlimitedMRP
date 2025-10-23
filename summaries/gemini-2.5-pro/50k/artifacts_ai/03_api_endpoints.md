| Component Name | HTTP Method | Endpoint Path | Brief Description |
| :--- | :--- | :--- | :--- |
| Order Service | GET | `/ping` | Health check and build information endpoint. |
| Order Service | HEAD | `/ping` | Health check and build information endpoint. |
| Order Service | GET | `/catalog` | Get all catalog items. |
| Order Service | POST | `/catalog` | Create a new catalog item. |
| Order Service | GET | `/catalog/{sku}` | Get a specific catalog item by SKU. |
| Order Service | PUT | `/catalog/{sku}` | Update a specific catalog item by SKU. |
| Order Service | DELETE | `/catalog/{sku}` | Delete a specific catalog item by SKU. |
| Order Service | GET | `/dealers` | Get all dealers. |
| Order Service | POST | `/dealers` | Create a new dealer. |
| Order Service | GET | `/dealers/{name}` | Get a specific dealer by name. |
| Order Service | PUT | `/dealers/{name}` | Update a specific dealer by name. |
| Order Service | DELETE | `/dealers/{name}` | Delete a specific dealer by name. |
| Order Service | POST | `/quotes` | Create a new quote. |
| Order Service | GET | `/quotes?name={customerName}` | Find quotes by customer name. |
| Order Service | GET | `/quotes/{id}` | Get a specific quote by ID. |
| Order Service | PUT | `/quotes/{id}` | Update a specific quote by ID. |
| Order Service | DELETE | `/quotes/{id}` | Delete a specific quote by ID. |
| Order Service | POST | `/orders` | Create a new order from a quote (via `fromQuote` parameter). |
| Order Service | GET | `/orders?dealer={d}&status={s}` | Find orders, filterable by dealer and/or status. |
| Order Service | GET | `/orders/{id}` | Get a specific order by ID. |
| Order Service | PUT | `/orders/{id}` | Update a specific order by ID. |
| Order Service | DELETE | `/orders/{id}` | Delete a specific order by ID. |
| Order Service | PUT | `/orders/{id}/status` | Update an order's status and add a history event. |
| Order Service | POST | `/orders/{id}/events` | Manually add a history event to an order. |
| Order Service | GET | `/shipments` | Get shipments, filterable by status. |
| Order Service | POST | `/shipments` | Create a new shipment for an order. |
| Order Service | GET | `/shipments/{orderId}` | Get a shipment by its order ID. |
| Order Service | PUT | `/shipments/{orderId}` | Update a shipment by its order ID. |
| Order Service | DELETE | `/shipments/{orderId}` | Delete a shipment by its order ID. |
| Order Service | POST | `/shipments/{orderId}/events` | Add a history event to a shipment. |
| Order Service | GET | `/shipments/deliveries` | Get aggregated delivery information (Shipment+Order+Quote). |