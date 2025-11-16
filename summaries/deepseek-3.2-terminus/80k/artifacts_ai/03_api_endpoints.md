| Component Name | HTTP Method | Endpoint Path | Brief Description |
|----------------|-------------|---------------|-------------------|
| OrderService | GET | `/catalog` | List all products |
| OrderService | POST | `/catalog` | Add new product |
| OrderService | PUT | `/catalog/{sku}` | Update product |
| OrderService | DELETE | `/catalog/{sku}` | Remove product |
| OrderService | GET | `/dealers` | List all dealers |
| OrderService | POST | `/dealers` | Add new dealer |
| OrderService | GET | `/quotes/{id}` | Get specific quote |
| OrderService | GET | `/quotes` | Search quotes by customer |
| OrderService | POST | `/quotes` | Create new quote |
| OrderService | PUT | `/quotes/{id}` | Update quote |
| OrderService | DELETE | `/quotes/{id}` | Delete quote |
| OrderService | GET | `/orders/{id}` | Get specific order |
| OrderService | GET | `/orders` | Filter orders by dealer and status |
| OrderService | POST | `/orders` | Create order from quote |
| OrderService | PUT | `/orders/{orderId}/status` | Update order status |
| OrderService | POST | `/orders/{orderId}/events` | Add order events/comments |
| OrderService | GET | `/shipments` | List all shipments |
| OrderService | GET | `/shipments` | Filter shipments by status |
| OrderService | GET | `/shipments/{orderId}` | Get shipment details |
| OrderService | POST | `/shipments` | Create shipment record |
| OrderService | PUT | `/shipments/{orderId}` | Update shipment |
| OrderService | POST | `/shipments/{orderId}/events` | Add shipment event |
| OrderService | GET | `/shipments/deliveries` | Get delivery information |
| OrderService | GET | `/ping` | Health checks and service status |