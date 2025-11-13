| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| OrderService | HEAD | /ping | Liveness/health check. |
| OrderService | GET | /ping | Returns ping/validation messages and build info. |
| OrderService | GET | /catalog | List all catalog items. |
| OrderService | GET | /catalog/{sku} | Get a catalog item by SKU. |
| OrderService | POST | /catalog | Create a new catalog item. |
| OrderService | PUT | /catalog/{sku} | Update an existing catalog item by SKU. |
| OrderService | DELETE | /catalog/{sku} | Delete a catalog item by SKU. |
| OrderService | GET | /dealers | List all dealers. |
| OrderService | GET | /dealers/{name} | Get a dealer by name. |
| OrderService | POST | /dealers | Create a new dealer. |
| OrderService | PUT | /dealers/{name} | Update an existing dealer by name. |
| OrderService | DELETE | /dealers/{name} | Delete a dealer by name. |
| OrderService | GET | /quotes/{quoteId} | Get a quote by ID. |
| OrderService | GET | /quotes?name={nameFragment} | Search quotes by customer name fragment. |
| OrderService | POST | /quotes | Create a new quote. |
| OrderService | PUT | /quotes/{quoteId} | Update an existing quote by ID. |
| OrderService | DELETE | /quotes/{quoteId} | Delete a quote by ID. |
| OrderService | GET | /orders/{orderId} | Get an order by ID. |
| OrderService | GET | /orders?dealer={name}&status={OrderStatus} | List orders filtered by dealer and/or status. |
| OrderService | POST | /orders?fromQuote={quoteId} | Create an order from an existing quote. |
| OrderService | PUT | /orders/{orderId} | Update an existing order by ID. |
| OrderService | PUT | /orders/{orderId}/status | Update order status (adds an event). |
| OrderService | POST | /orders/{orderId}/events | Add an event to an order. |
| OrderService | DELETE | /orders/{orderId} | Delete an order by ID. |
| OrderService | GET | /shipments?status={OrderStatus} | List shipments, optionally filtered by associated order status. |
| OrderService | GET | /shipments/deliveries | List aggregated deliveries (shipment + order + quote) for DeliveryConfirmed. |
| OrderService | GET | /shipments/{orderId} | Get a shipment record by order ID. |
| OrderService | POST | /shipments | Create a shipment record. |
| OrderService | PUT | /shipments/{orderId} | Update a shipment record by order ID. |
| OrderService | POST | /shipments/{orderId}/events | Add an event to a shipment. |
| OrderService | DELETE | /shipments/{orderId} | Delete a shipment record by order ID. |
| Ancillary Flask Example App | GET | / | Hello endpoint (returns {"message":"hello"}). |
| Ancillary Flask Example App | GET | /tests | List items (demo/test endpoint). |
| Ancillary Flask Example App | POST | /tests | Append an item (demo/test endpoint). |