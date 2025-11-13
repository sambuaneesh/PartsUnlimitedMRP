| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Clients (Web SPA) | GET | / | Serves SPA UI and static assets (mrp.war). |
| IntegrationService | N/A | N/A | No public HTTP endpoints; scheduled worker using Azure Storage Queues. |
| OrderService | GET | /catalog | List all catalog items. |
| OrderService | GET | /catalog/{sku} | Retrieve catalog item by SKU. |
| OrderService | POST | /catalog | Create a catalog item. |
| OrderService | PUT | /catalog/{sku} | Update a catalog item. |
| OrderService | DELETE | /catalog/{sku} | Delete a catalog item. |
| OrderService | GET | /dealers | List all dealers. |
| OrderService | GET | /dealers/{name} | Retrieve dealer by name. |
| OrderService | POST | /dealers | Create a dealer. |
| OrderService | PUT | /dealers/{name} | Update a dealer. |
| OrderService | DELETE | /dealers/{name} | Delete a dealer. |
| OrderService | GET | /quotes/{quoteId} | Retrieve quote by ID. |
| OrderService | GET | /quotes?name={fragment} | Search quotes by customer name (case-insensitive contains). |
| OrderService | POST | /quotes | Create a quote (auto-creates Dealer if missing). |
| OrderService | PUT | /quotes/{quoteId} | Update a quote. |
| OrderService | DELETE | /quotes/{quoteId} | Delete a quote. |
| OrderService | GET | /orders/{orderId} | Retrieve order by ID. |
| OrderService | GET | /orders?dealer={name}&status={OrderStatus} | List orders filtered by dealer and/or status. |
| OrderService | POST | /orders?fromQuote={quoteId} | Create order from quote (idempotent). |
| OrderService | POST | /orders/{orderId}/events | Add an order event (date set to current short date). |
| OrderService | PUT | /orders/{orderId} | Update an order. |
| OrderService | PUT | /orders/{orderId}/status | Update order status (appends event with current date). |
| OrderService | DELETE | /orders/{orderId} | Delete an order. |
| OrderService | GET | /shipments | List all shipments. |
| OrderService | GET | /shipments?status={OrderStatus} | List shipments for orders with the given status. |
| OrderService | GET | /shipments/deliveries | List delivery aggregates for DeliveryConfirmed orders. |
| OrderService | GET | /shipments/{orderId} | Retrieve shipment by order ID. |
| OrderService | POST | /shipments | Create a shipment record. |
| OrderService | PUT | /shipments/{orderId} | Update a shipment record. |
| OrderService | POST | /shipments/{orderId}/events | Add a shipment event (date set to current short date). |
| OrderService | DELETE | /shipments/{orderId} | Delete a shipment record. |
| OrderService | HEAD | /ping | Liveness check. |
| OrderService | GET | /ping | Ping text with build info. |