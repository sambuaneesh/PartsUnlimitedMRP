| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| OrderService | HEAD | /ping | Liveness endpoint; returns 200 OK. |
| OrderService | GET | /ping | Returns ping/validation messages and build metadata. |
| OrderService | GET | /catalog | List catalog items; 404 if empty. |
| OrderService | GET | /catalog/{sku} | Get catalog item by SKU. |
| OrderService | POST | /catalog | Create catalog item; 201 Created, 409 if SKU exists. |
| OrderService | PUT | /catalog/{sku} | Update catalog item by SKU. |
| OrderService | DELETE | /catalog/{sku} | Delete catalog item by SKU. |
| OrderService | GET | /dealers | List dealers (note: intentionally heavy path); 404 if empty. |
| OrderService | GET | /dealers/{name} | Get dealer by name. |
| OrderService | POST | /dealers | Create dealer; 201 Created, 409 if exists. |
| OrderService | PUT | /dealers/{name} | Update dealer by name. |
| OrderService | DELETE | /dealers/{name} | Delete dealer by name. |
| OrderService | GET | /quotes/{quoteId} | Get quote by ID. |
| OrderService | GET | /quotes?name={nameFragment} | Search quotes by customer name (contains; case-insensitive). |
| OrderService | POST | /quotes | Create quote; returns created quote and Location. |
| OrderService | PUT | /quotes/{quoteId} | Update quote by ID. |
| OrderService | DELETE | /quotes/{quoteId} | Delete quote by ID. |
| OrderService | GET | /orders/{orderId} | Get order by ID. |
| OrderService | GET | /orders?dealer={name}&status={OrderStatus} | List orders filtered by dealer and/or status. |
| OrderService | POST | /orders?fromQuote={quoteId} | Create order from a quote. |
| OrderService | POST | /orders/{orderId}/events | Add an event to an order. |
| OrderService | PUT | /orders/{orderId} | Update order by ID. |
| OrderService | PUT | /orders/{orderId}/status | Update order status (accepts status and optional event comments). |
| OrderService | DELETE | /orders/{orderId} | Delete order by ID. |
| OrderService | GET | /shipments?status={OrderStatus} | List shipments filtered by order status. |
| OrderService | GET | /shipments/deliveries | List deliveries (aggregated Shipment + Order + Quote for DeliveryConfirmed). |
| OrderService | GET | /shipments/{orderId} | Get shipment by order ID. |
| OrderService | POST | /shipments | Create shipment for an order. |
| OrderService | PUT | /shipments/{orderId} | Update shipment by order ID. |
| OrderService | POST | /shipments/{orderId}/events | Add an event to a shipment. |
| OrderService | DELETE | /shipments/{orderId} | Delete shipment by order ID. |