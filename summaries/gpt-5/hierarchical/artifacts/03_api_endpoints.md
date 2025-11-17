| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| Ordering Service - Ping | GET | /ping | Liveness probe to verify the service is up. |
| Ordering Service - Status | GET | /status | Returns service status and build metadata. |
| Ordering Service - Catalog | GET | /catalog | List all catalog items. |
| Ordering Service - Catalog | GET | /catalog/{sku} | Retrieve a catalog item by SKU. |
| Ordering Service - Catalog | POST | /catalog | Create a new catalog item (validates SKU uniqueness). |
| Ordering Service - Catalog | PUT | /catalog/{sku} | Upsert an existing catalog item by SKU. |
| Ordering Service - Catalog | DELETE | /catalog/{sku} | Remove a catalog item by SKU. |
| Ordering Service - Dealers | GET | /dealers | Retrieve all dealers (includes intentional load loop for APM). |
| Ordering Service - Dealers | GET | /dealers/{name} | Retrieve a dealer by name. |
| Ordering Service - Dealers | POST | /dealers | Create a new dealer record. |
| Ordering Service - Dealers | PUT | /dealers/{name} | Update an existing dealer by name. |
| Ordering Service - Dealers | DELETE | /dealers/{name} | Remove a dealer by name. |
| Ordering Service - Quotes | GET | /quotes/{id} | Retrieve a quote by ID. |
| Ordering Service - Quotes | GET | /quotes?customerName={name} | List quotes filtered by customer name. |
| Ordering Service - Quotes | POST | /quotes | Create a new quote (ID auto-generated if missing). |
| Ordering Service - Quotes | PUT | /quotes/{id} | Update an existing quote. |
| Ordering Service - Quotes | DELETE | /quotes/{id} | Delete a quote by ID. |
| Ordering Service - Orders | GET | /orders/{id} | Retrieve an order by ID. |
| Ordering Service - Orders | GET | /orders?dealerName={name}&status={status} | List orders filtered by dealer and optional status. |
| Ordering Service - Orders | POST | /orders | Create a new order from an existing quote. |
| Ordering Service - Orders | PUT | /orders/{id} | Update/replace an order (e.g., status, details). |
| Ordering Service - Orders | POST | /orders/{id}/events | Append a dated event/comment to an order. |
| Ordering Service - Orders | DELETE | /orders/{id} | Delete an order by ID. |
| Ordering Service - Shipments | GET | /shipments?status={orderStatus} | List shipments filtered by related order status (or all with None). |
| Ordering Service - Shipments | GET | /shipments/{orderId} | Retrieve the shipment for a given order. |
| Ordering Service - Shipments | POST | /shipments | Create a shipment for an existing order (enforces one per order). |
| Ordering Service - Shipments | PUT | /shipments/{orderId} | Update/replace an existing shipment. |
| Ordering Service - Shipments | POST | /shipments/{orderId}/events | Append a shipment event (date/comments). |
| Ordering Service - Shipments | DELETE | /shipments/{orderId} | Remove a shipment by order ID. |
| Ordering Service - Deliveries | GET | /deliveries/confirmed | List confirmed deliveries (aggregated shipment/order/quote view). |