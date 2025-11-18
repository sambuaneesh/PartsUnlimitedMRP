
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Order Service | GET | /catalog | List all catalog items |
| Order Service | POST | /catalog | Create new catalog item |
| Order Service | GET | /catalog/{sku} | Get specific catalog item |
| Order Service | PUT | /catalog/{sku} | Update catalog item |
| Order Service | DELETE | /catalog/{sku} | Delete catalog item |
| Order Service | GET | /dealers | List all dealers |
| Order Service | POST | /dealers | Create new dealer |
| Order Service | GET | /dealers/{name} | Get specific dealer |
| Order Service | PUT | /dealers/{name} | Update dealer |
| Order Service | DELETE | /dealers/{name} | Delete dealer |
| Order Service | GET | /quotes | List all quotes |
| Order Service | POST | /quotes | Create new quote |
| Order Service | GET | /quotes/{quoteId} | Get specific quote |
| Order Service | PUT | /quotes/{quoteId} | Update quote |
| Order Service | DELETE | /quotes/{quoteId} | Delete quote |
| Order Service | GET | /orders | List all orders |
| Order Service | POST | /orders | Create order from quote |
| Order Service | GET | /orders/{orderId} | Get specific order |
| Order Service | PUT | /orders/{orderId} | Update order |
| Order Service | DELETE | /orders/{orderId} | Delete order |
| Order Service | PUT | /orders/{orderId}/status | Update order status |
| Order Service | POST | /orders/{orderId}/events | Add order event |
| Order Service | GET | /shipments | List all shipments |
| Order Service | POST | /shipments | Create new shipment |
| Order Service | GET | /shipments/{id} | Get specific shipment |
| Order Service | PUT | /shipments/{id} | Update shipment |
| Order Service | DELETE | /shipments/{orderId} | Delete shipment |
| Order Service | GET | /shipments/deliveries | Get confirmed deliveries |
| Order Service | POST | /shipments/{id}/events | Add shipment event |
| Order Service | HEAD | /ping | Health check |
| Order Service | GET | /ping | Service info with build details |