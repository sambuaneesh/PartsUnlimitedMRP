```markdown
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---------------|-------------|---------------|-------------------|
| CatalogController | GET | `/catalog` | List all catalog items |
| CatalogController | POST | `/catalog` | Add new catalog item |
| CatalogController | PUT | `/catalog/{sku}` | Update product |
| CatalogController | DELETE | `/catalog/{sku}` | Remove product |
| DealerController | GET | `/dealers` | List all dealers |
| DealerController | POST | `/dealers` | Add new dealer |
| QuoteController | GET | `/quotes/{id}` | Get specific quote |
| QuoteController | GET | `/quotes?name={customerName}` | Search quotes by customer |
| QuoteController | POST | `/quotes` | Create new quote |
| QuoteController | PUT | `/quotes/{id}` | Update quote |
| QuoteController | DELETE | `/quotes/{id}` | Delete quote |
| OrderController | GET | `/orders/{id}` | Get specific order |
| OrderController | GET | `/orders?dealer={name}&status={status}` | Filter orders |
| OrderController | POST | `/orders?fromQuote={quoteId}` | Create order from quote |
| OrderController | PUT | `/orders/{orderId}/status` | Update order status |
| OrderController | POST | `/orders/{orderId}/events` | Add order events/comments |
| ShipmentController | GET | `/shipments` | List all shipments |
| ShipmentController | GET | `/shipments?status={status}` | Filter by status |
| ShipmentController | GET | `/shipments/{orderId}` | Get shipment details |
| ShipmentController | POST | `/shipments` | Create shipment record |
| ShipmentController | PUT | `/shipments/{orderId}` | Update shipment |
| ShipmentController | POST | `/shipments/{orderId}/events` | Add shipment event |
| PingController | GET | `/ping` | Health checks and service status |
```