```markdown
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---------------|-------------|---------------|-------------------|
| Order Service | GET | `/catalog` | List all catalog items |
| Order Service | GET | `/catalog/{sku}` | Get specific catalog item |
| Order Service | POST | `/catalog` | Create new catalog item |
| Order Service | PUT | `/catalog/{sku}` | Update/upsert catalog item |
| Order Service | DELETE | `/catalog/{sku}` | Remove catalog item |
| Order Service | GET | `/dealers` | List all dealers |
| Order Service | GET | `/dealers/{name}` | Get specific dealer |
| Order Service | POST | `/dealers` | Create new dealer |
| Order Service | PUT | `/dealers/{name}` | Update dealer |
| Order Service | DELETE | `/dealers/{name}` | Remove dealer |
| Order Service | GET | `/quotes/{quoteId}` | Get quote by ID |
| Order Service | GET | `/quotes?name={customer}` | Get quotes by customer name |
| Order Service | POST | `/quotes` | Create quote |
| Order Service | PUT | `/quotes/{quoteId}` | Update quote |
| Order Service | DELETE | `/quotes/{quoteId}` | Delete quote |
| Order Service | GET | `/orders/{orderId}` | Get order by ID |
| Order Service | GET | `/orders?dealer=&status=` | Get orders by dealer/status |
| Order Service | POST | `/orders?fromQuote={quoteId}` | Create order from quote |
| Order Service | POST | `/orders/{orderId}/events` | Add order event |
| Order Service | PUT | `/orders/{orderId}` | Update order |
| Order Service | PUT | `/orders/{orderId}/status` | Update order status |
| Order Service | DELETE | `/orders/{orderId}` | Delete order |
| Order Service | GET | `/shipments?status=` | Get shipments by status |
| Order Service | GET | `/shipments/deliveries` | Get delivery confirmations |
| Order Service | GET | `/shipments/{orderId}` | Get shipment by order ID |
| Order Service | POST | `/shipments` | Create shipment record |
| Order Service | PUT | `/shipments/{orderId}` | Update shipment |
| Order Service | POST | `/shipments/{orderId}/events` | Add shipment event |
| Order Service | DELETE | `/shipments/{orderId}` | Delete shipment |
| Order Service | HEAD | `/ping` | Health check |
| Order Service | GET | `/ping` | System status with build info |
```