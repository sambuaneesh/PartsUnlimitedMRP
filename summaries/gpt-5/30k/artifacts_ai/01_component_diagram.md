```mermaid
graph TB

subgraph Client
  WebUI[Web UI (WinJS SPA)\nTomcat mrp.war\nHTTP 9080/80]
end

subgraph Services
  subgraph OrderService[OrderService (Spring Boot)\nHTTP 8080]
    CatalogCtrl[CatalogController\n/catalog]
    DealerCtrl[DealerController\n/dealers]
    QuoteCtrl[QuoteController\n/quotes]
    OrderCtrl[OrderController\n/orders]
    ShipmentCtrl[ShipmentController\n/shipments]
    PingCtrl[PingController\n/ping]
    Filters[AppInsights + CORS Filters]
  end

  subgraph IntegrationService[IntegrationService (Spring Boot)\nScheduled every 30s]
    OrdersTask[CreateOrderProcessTask\n(ingest orders)]
    ProductTask[UpdateProductProcessTask\n(export inventory)]
    MrpClient[MrpConnectService\n(REST client)]
  end
end

subgraph DataStores
  MongoDB[(MongoDB\nDB: ordering)]
  AzureOrders[(Azure Storage Queue\n"orders")]
  AzureProduct[(Azure Storage Queue\n"product")]
end

subgraph Telemetry
  AppInsights[(Azure Application Insights)]
end

%% Client -> API
WebUI -->|HTTP/JSON (CORS)| CatalogCtrl
WebUI -->|HTTP/JSON (CORS)| DealerCtrl
WebUI -->|HTTP/JSON (CORS)| QuoteCtrl
WebUI -->|HTTP/JSON (CORS)| OrderCtrl
WebUI -->|HTTP/JSON (CORS)| ShipmentCtrl

%% IntegrationService <-> Queues
AzureOrders -->|poll/dequeue| OrdersTask
OrdersTask -->|POST /quotes\nPOST /orders?fromQuote\nPOST /shipments| MrpClient
MrpClient -->|HTTP/JSON| OrderService

ProductTask -->|enqueue ProductMessage| AzureProduct
OrderService -->|GET /catalog| ProductTask

%% OrderService -> MongoDB
OrderService -->|Spring Data Mongo\nMongoOperationsWithRetry| MongoDB

%% Telemetry
OrderService -.->|requests + exceptions| AppInsights
OrderService -.->|Mongo dependency telemetry| AppInsights
```

Rationale:
- The system centers on a single OrderService owning all domain resources (catalog, dealers, quotes, orders, shipments) with MongoDB persistence, exposed via REST to a static Web UI over permissive CORS. 
- A separate IntegrationService runs scheduled workers that bridge Azure Storage Queues: it ingests order messages into the OrderService (synchronous REST) and exports catalog inventory to a product queue, forming an async boundary with at-least-once semantics; telemetry from the OrderService flows to Application Insights.