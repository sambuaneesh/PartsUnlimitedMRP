| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Clients (Web UI) | GET | /mrp/* | Serves static single-page application (HTML/JS/CSS) that calls OrderService APIs |
| OrderService | HEAD | /ping | Liveness probe |
| OrderService | GET | /ping | Health/build info string |
| OrderService | GET | /catalog | List all catalog items |
| OrderService | GET | /catalog/{sku} | Get catalog item by SKU |
| OrderService | POST | /catalog | Create catalog item |
| OrderService | PUT | /catalog/{sku} | Update catalog item by SKU |
| OrderService | DELETE | /catalog/{sku} | Delete catalog item by SKU |
| OrderService | GET | /dealers | List all dealers |
| OrderService | GET | /dealers/{name} | Get dealer by name |
| OrderService | POST | /dealers | Create dealer |
| OrderService | PUT | /dealers/{name} | Update dealer by name |
| OrderService | DELETE | /dealers/{name} | Delete dealer by name |
| OrderService | GET | /quotes/{quoteId} | Get quote by ID |
| OrderService | GET | /quotes?name={fragment} | Search quotes by customer name fragment |
| OrderService | POST | /quotes | Create quote |
| OrderService | PUT | /quotes/{quoteId} | Update quote by ID |
| OrderService | DELETE | /quotes/{quoteId} | Delete quote by ID |
| OrderService | GET | /orders/{orderId} | Get order by ID |
| OrderService | GET | /orders?dealer={name}&status={status} | List orders filtered by dealer and/or status |
| OrderService | POST | /orders?fromQuote={quoteId} | Create order from a quote |
| OrderService | POST | /orders/{orderId}/events | Add event to an order |
| OrderService | PUT | /orders/{orderId} | Replace order by ID |
| OrderService | PUT | /orders/{orderId}/status | Update order status (adds event) |
| OrderService | DELETE | /orders/{orderId} | Delete order by ID |
| OrderService | GET | /shipments | List all shipments |
| OrderService | GET | /shipments?status={status} | List shipments for orders with given status |
| OrderService | GET | /shipments/deliveries | List delivery aggregates (Shipment + Order + Quote) |
| OrderService | GET | /shipments/{orderId} | Get shipment by order ID |
| OrderService | POST | /shipments | Create shipment record |
| OrderService | PUT | /shipments/{orderId} | Update shipment by order ID |
| OrderService | POST | /shipments/{orderId}/events | Add event to a shipment |
| OrderService | DELETE | /shipments/{orderId} | Delete shipment by order ID |