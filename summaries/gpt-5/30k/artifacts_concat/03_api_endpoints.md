| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| OrderService | GET | /catalog | List catalog items |
| OrderService | GET | /catalog/{sku} | Get catalog item by SKU |
| OrderService | POST | /catalog | Create catalog item |
| OrderService | PUT | /catalog/{sku} | Update catalog item by SKU |
| OrderService | DELETE | /catalog/{sku} | Delete catalog item by SKU |
| OrderService | GET | /dealers | List dealers |
| OrderService | GET | /dealers/{name} | Get dealer by name |
| OrderService | POST | /dealers | Create dealer |
| OrderService | PUT | /dealers/{name} | Update dealer by name |
| OrderService | DELETE | /dealers/{name} | Delete dealer by name |
| OrderService | GET | /quotes/{quoteId} | Get quote by ID |
| OrderService | GET | /quotes?name={customerNameFragment} | Search quotes by customer name fragment |
| OrderService | POST | /quotes | Create quote (ID generated if missing) |
| OrderService | PUT | /quotes/{quoteId} | Update quote by ID |
| OrderService | DELETE | /quotes/{quoteId} | Delete quote by ID |
| OrderService | GET | /orders/{orderId} | Get order by ID |
| OrderService | GET | /orders | List orders (optional filters dealer and status) |
| OrderService | POST | /orders?fromQuote={quoteId} | Create order from quote ID |
| OrderService | POST | /orders/{orderId}/events | Add event to order |
| OrderService | PUT | /orders/{orderId} | Update full order by ID |
| OrderService | PUT | /orders/{orderId}/status | Update order status (appends event) |
| OrderService | DELETE | /orders/{orderId} | Delete order by ID |
| OrderService | GET | /shipments | List shipments (all or filtered by related order status) |
| OrderService | GET | /shipments?status={status} | List shipments filtered by related order status |
| OrderService | GET | /shipments/deliveries | List delivery aggregates (shipment + order + quote) |
| OrderService | GET | /shipments/{orderId} | Get shipment by order ID |
| OrderService | POST | /shipments | Create shipment record (one per order) |
| OrderService | PUT | /shipments/{orderId} | Update shipment by order ID |
| OrderService | POST | /shipments/{orderId}/events | Add shipment event |
| OrderService | DELETE | /shipments/{orderId} | Delete shipment by order ID |
| OrderService | HEAD | /ping | Liveness check |
| OrderService | GET | /ping | Health/config/build info |
| Web Client (Tomcat) | GET | /mrp | Serve MRP SPA (static web UI) |
| Load Testing Flask Sample | GET | / | Hello endpoint for sample app |
| Load Testing Flask Sample | GET | /tests | List in-memory test items |
| Load Testing Flask Sample | POST | /tests | Create in-memory test item |