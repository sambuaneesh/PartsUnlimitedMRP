
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Catalog Service | GET | /catalog | List all catalog items |
| Catalog Service | GET | /catalog/{sku} | Get specific item |
| Catalog Service | POST | /catalog | Create new item |
| Catalog Service | PUT | /catalog/{sku} | Update item |
| Catalog Service | DELETE | /catalog/{sku} | Delete item |
| Dealer Service | GET | /dealers | List all dealers |
| Dealer Service | GET | /dealers/{name} | Get specific dealer |
| Dealer Service | POST | /dealers | Create dealer |
| Dealer Service | PUT | /dealers/{name} | Update dealer |
| Dealer Service | DELETE | /dealers/{name} | Delete dealer |
| Quote Service | GET | /quotes | List quotes (supports ?name filter) |
| Quote Service | GET | /quotes/{quoteId} | Get specific quote |
| Quote Service | POST | /quotes | Create quote |
| Quote Service | PUT | /quotes/{quoteId} | Update quote |
| Quote Service | DELETE | /quotes/{quoteId} | Delete quote |
| Order Service | GET | /orders | List orders (supports ?dealer & ?status filters) |
| Order Service | GET | /orders/{orderId} | Get specific order |
| Order Service | POST | /orders?fromQuote={id} | Create order from quote |
| Order Service | PUT | /orders/{orderId} | Update order |
| Order Service | PUT | /orders/{orderId}/status | Update order status |
| Order Service | POST | /orders/{orderId}/events | Add order event |
| Order Service | DELETE | /orders/{orderId} | Delete order |
| Shipment Service | GET | /shipments | List shipments (supports ?status filter) |
| Shipment Service | GET | /shipments/{id} | Get specific shipment |
| Shipment Service | GET | /shipments/deliveries | Get confirmed deliveries |
| Shipment Service | POST | /shipments | Create shipment |
| Shipment Service | PUT | /shipments/{id} | Update shipment |
| Shipment Service | POST | /shipments/{id}/events | Add shipment event |
| Shipment Service | DELETE | /shipments/{orderId} | Delete shipment |
| System Service | HEAD | /ping | Health check |
| System Service | GET | /ping | Service info with build details |