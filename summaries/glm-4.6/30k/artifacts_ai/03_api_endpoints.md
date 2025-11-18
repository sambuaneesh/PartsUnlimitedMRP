
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Order Service | GET | /catalog | List all catalog items |
| Order Service | POST | /catalog | Add new catalog item |
| Order Service | PUT | /catalog/{skuNumber} | Update catalog item |
| Order Service | DELETE | /catalog/{skuNumber} | Remove catalog item |
| Order Service | GET | /dealers | List all dealers |
| Order Service | POST | /dealers | Add new dealer |
| Order Service | PUT | /dealers/{name} | Update dealer |
| Order Service | DELETE | /dealers/{name} | Remove dealer |
| Order Service | GET | /quotes | List all quotes |
| Order Service | GET | /quotes?name={customerName} | Get quotes by customer |
| Order Service | POST | /quotes | Create new quote |
| Order Service | PUT | /quotes/{id} | Update quote |
| Order Service | DELETE | /quotes/{id} | Delete quote |
| Order Service | POST | /orders | Create new order |
| Order Service | POST | /orders?fromQuote={id} | Create order from existing quote |
| Order Service | GET | /orders/{id} | Get order by ID |
| Order Service | PUT | /orders/{id}/status | Update order status |
| Order Service | POST | /orders/{id}/events | Add order event |
| Order Service | DELETE | /orders/{id} | Delete order |
| Order Service | GET | /shipments | List all shipments |
| Order Service | GET | /shipments?status={status} | Filter shipments by status |
| Order Service | POST | /shipments | Create new shipment |
| Order Service | PUT | /shipments/{id} | Update shipment |
| Order Service | POST | /shipments/{id}/events | Add shipment event |
| Order Service | GET | /shipments/deliveries | Aggregate delivery data |
| Order Service | DELETE | /shipments/{id} | Delete shipment |
| Order Service | HEAD | /ping | Service health check |
| Order Service | GET | /ping | Service health check with metadata |