
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|----------------|-------------|---------------|-------------------|
| Order Service | GET | /catalog | List all catalog items |
| Order Service | POST | /catalog | Add new catalog item |
| Order Service | PUT | /catalog/{skuNumber} | Update catalog item |
| Order Service | DELETE | /catalog/{skuNumber} | Remove catalog item |
| Order Service | GET | /dealers | List all dealers |
| Order Service | POST | /dealers | Add new dealer |
| Order Service | PUT | /dealers/{name} | Update dealer |
| Order Service | DELETE | /dealers/{name} | Remove dealer |
| Order Service | GET | /quotes | List quotes (with optional customer filter) |
| Order Service | POST | /quotes | Create new quote |
| Order Service | PUT | /quotes/{id} | Update quote |
| Order Service | DELETE | /quotes/{id} | Delete quote |
| Order Service | GET | /quotes?name={customerName} | Get quotes by customer name |
| Order Service | POST | /orders | Create order from quote |
| Order Service | GET | /orders/{id} | Get order by ID |
| Order Service | PUT | /orders/{id}/status | Update order status |
| Order Service | POST | /orders/{id}/events | Add order event |
| Order Service | POST | /shipments | Create shipment |
| Order Service | GET | /shipments | List shipments (with status filter) |
| Order Service | PUT | /shipments/{id} | Update shipment |
| Order Service | POST | /shipments/{id}/events | Add shipment event |
| Order Service | GET | /shipments/deliveries | Get delivery information |
| Order Service | HEAD | /ping | Health check |
| Order Service | GET | /ping | Health check with build metadata |