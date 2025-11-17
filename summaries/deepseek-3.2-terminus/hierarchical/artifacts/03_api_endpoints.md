```markdown
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---------------|-------------|---------------|-------------------|
| CatalogController | GET | /catalog | Retrieve all catalog items (parts inventory) |
| CatalogController | GET | /catalog/{sku} | Retrieve specific catalog item by SKU |
| CatalogController | POST | /catalog | Create new catalog item with validation |
| CatalogController | PUT | /catalog/{sku} | Update existing catalog item (upsert operation) |
| CatalogController | DELETE | /catalog/{sku} | Remove catalog item from inventory |
| DealerController | GET | /dealers | Retrieve all dealers (suppliers) |
| DealerController | GET | /dealers/{name} | Retrieve specific dealer by name |
| DealerController | POST | /dealers | Add new dealer to the system |
| DealerController | PUT | /dealers/{name} | Update existing dealer information |
| DealerController | DELETE | /dealers/{name} | Remove dealer from the system |
| OrderController | GET | /orders | Retrieve all orders |
| OrderController | GET | /orders/{id} | Retrieve specific order by ID |
| OrderController | GET | /orders/dealer/{dealerName} | Retrieve orders by dealer name |
| OrderController | GET | /orders/status/{status} | Retrieve orders by status |
| OrderController | POST | /orders | Create new order (optionally from quote) |
| OrderController | PUT | /orders/{id} | Update existing order |
| OrderController | PATCH | /orders/{id} | Partially update order status/events |
| OrderController | DELETE | /orders/{id} | Remove order from system |
| PingController | GET | /ping | Basic health check endpoint |
| PingController | GET | /ping/status | Comprehensive service status with configuration |
| QuoteController | GET | /quotes | Retrieve all quotes |
| QuoteController | GET | /quotes/{id} | Retrieve specific quote by ID |
| QuoteController | GET | /quotes/customer/{customerName} | Retrieve quotes by customer name |
| QuoteController | POST | /quotes | Create new quote |
| QuoteController | PUT | /quotes/{id} | Update existing quote |
| QuoteController | DELETE | /quotes/{id} | Remove quote from system |
| ShipmentController | GET | /shipments | Retrieve all shipments |
| ShipmentController | GET | /shipments/{id} | Retrieve specific shipment by ID |
| ShipmentController | GET | /shipments/status/{status} | Retrieve shipments by status |
| ShipmentController | GET | /shipments/order/{orderId}/delivery | Retrieve delivery information for order |
| ShipmentController | POST | /shipments | Create new shipment record |
| ShipmentController | PUT | /shipments/{id} | Update existing shipment |
| ShipmentController | PATCH | /shipments/{id}/events | Add event to shipment tracking |
| ShipmentController | DELETE | /shipments/{id} | Remove shipment record |
```