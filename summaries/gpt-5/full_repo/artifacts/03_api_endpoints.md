| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| OrderService | GET | /ping | Health/status endpoint returning service and build info. |
| OrderService | HEAD | /ping | Lightweight liveness probe. |
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
| OrderService | GET | /quotes?name={customerNameFragment} | Search quotes by customer name (contains, case-insensitive). |
| OrderService | POST | /quotes | Create a new quote. |
| OrderService | PUT | /quotes/{quoteId} | Update an existing quote by ID. |
| OrderService | DELETE | /quotes/{quoteId} | Delete a quote by ID. |
| OrderService | GET | /orders/{orderId} | Get an order by ID. |
| OrderService | GET | /orders?dealer={dealerName}&status={orderStatus} | List orders by dealer and optional status (or by status only if dealer omitted). |
| OrderService | POST | /orders?fromQuote={quoteId} | Create a new order from a quote. |
| OrderService | POST | /orders/{orderId}/events | Add an event (comment) to an order. |
| OrderService | PUT | /orders/{orderId} | Replace/update an order by ID. |
| OrderService | PUT | /orders/{orderId}/status | Update order status (and append event). |
| OrderService | DELETE | /orders/{orderId} | Delete an order by ID. |
| OrderService | GET | /shipments?status={orderStatus} | List shipments filtered by associated order status (or all if omitted). |
| OrderService | GET | /shipments/deliveries | List deliveries (joined shipment, order, quote) with DeliveryConfirmed status. |
| OrderService | GET | /shipments/{orderId} | Get a shipment record by order ID. |
| OrderService | POST | /shipments | Create a new shipment record. |
| OrderService | PUT | /shipments/{orderId} | Update a shipment record by order ID. |
| OrderService | POST | /shipments/{orderId}/events | Add an event to a shipment record. |
| OrderService | DELETE | /shipments/{orderId} | Delete a shipment record by order ID. |