| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Web UI (Clients) | GET | /mrp | Serve SPA entrypoint (index) from Tomcat. |
| Web UI (Clients) | GET | /mrp/* | Serve static assets (HTML/JS/CSS) for the SPA. |
| OrderService | GET | /catalog | List catalog items (404 if empty). |
| OrderService | GET | /catalog/{sku} | Get catalog item by SKU. |
| OrderService | POST | /catalog | Create catalog item (409 on duplicate, 400 on validation). |
| OrderService | PUT | /catalog/{sku} | Update catalog item by SKU (404 if not found, 400 on validation). |
| OrderService | DELETE | /catalog/{sku} | Delete catalog item by SKU (204 on success, 404 if not found). |
| OrderService | GET | /dealers | List dealers (404 if empty; intentionally heavy for APM demo). |
| OrderService | GET | /dealers/{name} | Get dealer by name. |
| OrderService | POST | /dealers | Create or update dealer (201/200; 409 on duplicate, 400 on validation). |
| OrderService | PUT | /dealers/{name} | Update dealer by name (404 if not found, 400 on validation). |
| OrderService | DELETE | /dealers/{name} | Delete dealer by name (204 on success, 404 if not found). |
| OrderService | GET | /quotes/{id} | Get quote by ID. |
| OrderService | GET | /quotes?name={filter} | Search quotes by customer/dealer name (404 if none). |
| OrderService | POST | /quotes | Create quote (server generates ID if missing; 400 on duplicate/validation). |
| OrderService | PUT | /quotes/{id} | Update quote by ID (404 if not found, 400 on validation). |
| OrderService | DELETE | /quotes/{id} | Delete quote by ID (204 on success, 404 if not found). |
| OrderService | GET | /orders/{id} | Get order by ID. |
| OrderService | GET | /orders?dealer={name}&status={OrderStatus} | List orders filtered by dealer and/or status (both optional; 404 if none). |
| OrderService | POST | /orders?fromQuote={quoteId} | Create order from existing quote (201; 409 if order already exists; 400 on bad request). |
| OrderService | POST | /orders/{id}/events | Append order event (date stamped by server; 201 on create; 400 if not found). |
| OrderService | PUT | /orders/{id} | Update order by ID (404 if not found, 400 on validation). |
| OrderService | PUT | /orders/{id}/status | Update order status and append event (400 if not found). |
| OrderService | DELETE | /orders/{id} | Delete order by ID (204 on success, 404 if not found). |
| OrderService | GET | /shipments?status={OrderStatus} | List shipments filtered by associated order status (default None; 404 if none). |
| OrderService | GET | /shipments/deliveries | List Delivery aggregates (shipment + order + quote) for DeliveryConfirmed orders (404 if none). |
| OrderService | GET | /shipments/{orderId} | Get shipment by order ID. |
| OrderService | POST | /shipments | Create shipment (requires existing order; 400 on invalid/duplicate; 201 on success). |
| OrderService | PUT | /shipments/{orderId} | Update shipment by order ID (404 if not found; 400 on validation/id mismatch). |
| OrderService | POST | /shipments/{orderId}/events | Append shipment event (200/201; 400 on validation; 404 if not found). |
| OrderService | DELETE | /shipments/{orderId} | Delete shipment by order ID (204 on success, 404 if not found). |
| OrderService | HEAD | /ping | Liveness check. |
| OrderService | GET | /ping | Health/config/build info (ping/validation messages, build number/timestamp). |