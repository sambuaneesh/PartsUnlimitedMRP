```markdown
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---------------|-------------|---------------|-------------------|
| OrderService | GET | /catalog | List all catalog items |
| OrderService | GET | /catalog/{sku} | Get specific catalog item |
| OrderService | POST | /catalog | Add new catalog item |
| OrderService | PUT | /catalog/{sku} | Update catalog item |
| OrderService | DELETE | /catalog/{sku} | Remove catalog item |
| OrderService | GET | /dealers | List all dealers |
| OrderService | GET | /dealers/{name} | Get specific dealer |
| OrderService | POST | /dealers | Add new dealer |
| OrderService | PUT | /dealers/{name} | Update dealer |
| OrderService | DELETE | /dealers/{name} | Remove dealer |
| OrderService | GET | /quotes/{quoteId} | Get specific quote |
| OrderService | GET | /quotes?name={customerName} | Search quotes by customer |
| OrderService | POST | /quotes | Create new quote |
| OrderService | PUT | /quotes/{quoteId} | Update quote |
| OrderService | DELETE | /quotes/{quoteId} | Delete quote |
| OrderService | GET | /orders/{orderId} | Get specific order |
| OrderService | GET | /orders?dealer={dealerName} | Get orders by dealer |
| OrderService | POST | /orders?fromQuote={quoteId} | Create order from quote |
| OrderService | PUT | /orders/{orderId} | Update order |
| OrderService | PUT | /orders/{orderId}/status | Update order status |
| OrderService | POST | /orders/{orderId}/events | Add order event |
| OrderService | DELETE | /orders/{orderId} | Delete order |
| OrderService | GET | /shipments | List all shipments |
| OrderService | GET | /shipments/{id} | Get specific shipment |
| OrderService | GET | /shipments/deliveries | Get delivery aggregates |
| OrderService | POST | /shipments | Create shipment record |
| OrderService | PUT | /shipments/{id} | Update shipment |
| OrderService | POST | /shipments/{id}/events | Add shipment event |
| OrderService | DELETE | /shipments/{orderId} | Delete shipment |
| OrderService | GET | /ping | Health checks and monitoring |
```