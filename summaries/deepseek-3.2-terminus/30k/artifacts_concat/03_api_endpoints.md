```markdown
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---------------|-------------|---------------|-------------------|
| OrderService | GET | /catalog | Get all catalog items |
| OrderService | POST | /catalog | Add new catalog item |
| OrderService | GET | /catalog/{sku} | Get specific catalog item by SKU |
| OrderService | PUT | /catalog/{sku} | Update or upsert catalog item |
| OrderService | DELETE | /catalog/{sku} | Remove catalog item |
| OrderService | GET | /dealers | List all dealers |
| OrderService | POST | /dealers | Create new dealer |
| OrderService | GET | /dealers/{name} | Get specific dealer by name |
| OrderService | PUT | /dealers/{name} | Update dealer information |
| OrderService | DELETE | /dealers/{name} | Remove dealer |
| OrderService | GET | /quotes | Get quotes by customer name (query parameter) |
| OrderService | POST | /quotes | Create new quote |
| OrderService | GET | /quotes/{id} | Get specific quote by ID |
| OrderService | PUT | /quotes/{id} | Update quote |
| OrderService | DELETE | /quotes/{id} | Delete quote |
| OrderService | GET | /orders/{id} | Get order by ID |
| OrderService | GET | /orders | Get orders by dealer and status (query parameters) |
| OrderService | POST | /orders | Create order from quote (fromQuote query parameter) |
| OrderService | PUT | /orders/{id} | Update order |
| OrderService | PUT | /orders/{id}/status | Update order status |
| OrderService | POST | /orders/{id}/events | Add order event |
| OrderService | DELETE | /orders/{id} | Delete order |
| OrderService | GET | /shipments | Get shipments by status (query parameter) |
| OrderService | GET | /shipments/deliveries | Get delivery confirmations |
| OrderService | GET | /shipments/{id} | Get shipment by order ID |
| OrderService | POST | /shipments | Create shipment record |
| OrderService | PUT | /shipments/{id} | Update shipment |
| OrderService | POST | /shipments/{id}/events | Add shipment event |
| OrderService | DELETE | /shipments/{orderId} | Delete shipment |
| OrderService | HEAD | /ping | Health check |
| OrderService | GET | /ping | System status with build information |
| Flask REST API | GET | /tests | Retrieve list of testing types |
| Flask REST API | POST | /tests | Add new testing type |
| Flask REST API | GET | / | Health check endpoint |
```