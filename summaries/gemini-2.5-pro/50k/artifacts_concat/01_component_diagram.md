```markdown
```mermaid
graph TB
    subgraph "External Systems"
        direction LR
        External_Website["External Parts Unlimited Website"]
        Google_Maps["Google Maps Places API"]
    end

    subgraph "Parts Unlimited MRP System"
        Web_UI["Web UI (SPA)"]
        Order_Service["Order Service (Monolith)"]
        Integration_Service["Integration Service"]
        Message_Queues["Azure Message Queues"]
        MongoDB["MongoDB Database"]
    end

    %% User-facing Interactions
    Web_UI -- "REST API Calls" --> Order_Service
    Web_UI -- "Address Autocomplete" --> Google_Maps

    %% Core Data Flow
    Order_Service -- "Reads/Writes" --> MongoDB

    %% Asynchronous Integration Flow
    External_Website -- "Writes Order Messages" --> Message_Queues
    Integration_Service -- "Polls Order Messages" --> Message_Queues
    Integration_Service -- "REST API Calls" --> Order_Service
    Integration_Service -- "Writes Product Updates" --> Message_Queues
    External_Website -- "Reads Product Updates" --> Message_Queues

    %% Styling
    classDef component fill:#f9f,stroke:#333,stroke-width:2px;
    classDef database fill:#ccf,stroke:#333,stroke-width:2px;
    classDef external fill:#cfc,stroke:#333,stroke-width:2px;
    class Web_UI,Order_Service,Integration_Service,Message_Queues component;
    class MongoDB database;
    class External_Website,Google_Maps external;
```

The architecture consists of a static Web UI, a monolithic backend `Order Service`, and a background `Integration Service`. The UI communicates directly with the `Order Service` via synchronous REST API calls for all data, while the `Integration Service` uses message queues for asynchronous, decoupled communication with an external website before interacting with the same core REST API.
```