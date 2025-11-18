
| Component Name | HTTP Method | Endpoint Path | Brief Description |
|---|---|---|---|
| CatalogController | GET | /catalog | Retrieve all catalog items. |
| CatalogController | GET | /catalog/{sku} | Retrieve a specific catalog item by its SKU. |
| CatalogController | POST | /catalog | Add a new catalog item to the system. |
| CatalogController | PUT | /catalog/{sku} | Update an existing catalog item (upsert operation). |
| CatalogController | DELETE | /catalog/{sku} | Remove a catalog item from the system. |
| DealerController | GET | /dealers | Retrieve a list of all dealers. |
| DealerController | GET | /dealers/{name} | Retrieve a specific dealer by name. |
| DealerController | POST | /dealers | Add a new dealer to the system. |
| DealerController | PUT | /dealers/{name} | Update an existing dealer's information. |
| DealerController | DELETE | /dealers/{name} | Remove a dealer from the system. |
| OrderController | POST | /orders | Create a new order, typically from a quote. |
| OrderController | GET | /orders/{id} | Retrieve a specific order by its ID. |
| OrderController | GET | /orders | Retrieve a list of orders, filterable by dealer or status. |
| OrderController | POST | /orders/{id}/events | Add a tracking event to an order's log. |
| OrderController | PUT | /orders/{id}/status | Update the status of an order. |
| PingController | GET | /ping | Simple health check endpoint for the service. |
| PingController | GET | /ping/status | Comprehensive status and diagnostic endpoint. |
| QuoteController | POST | /quotes | Create a new quote. |
| QuoteController | GET | /quotes/{id} | Retrieve a specific quote by its ID. |
| QuoteController | GET | /quotes | Retrieve a list of quotes, filterable by customer name. |
| QuoteController | PUT | /quotes/{id} | Update an existing quote. |
| QuoteController | DELETE | /quotes/{id} | Delete a quote. |
| ShipmentController | POST | /shipments | Create a new shipment record. |
| ShipmentController | GET | /shipments/{id} | Retrieve a specific shipment by its ID. |
| ShipmentController | GET | /shipments | Retrieve a list of shipments, filterable by status. |
| ShipmentController | PUT | /shipments/{id} | Update an existing shipment record. |
| ShipmentController | POST | /shipments/{id}/events | Add an event to the shipment's tracking log. |
| ShipmentController | DELETE | /shipments/{id} | Delete a shipment record. |
| ShipmentController | GET | /shipments/{id}/delivery | Retrieve enriched delivery data combining shipment, order, and quote details. |