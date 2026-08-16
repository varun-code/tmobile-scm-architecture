flowchart TB

    TITLE["LLD - T-Mobile SCM Integration Platform - Oracle SCM Integration Patterns"]

    TMOBILE["T-Mobile Microservices<br/>Order | Inventory | Shipment | Procurement"]

    GATEWAY["SCM Integration API Gateway<br/>Authentication | Validation | Routing"]

    ORCHESTRATOR["Integration Orchestrator<br/>Transformation | Business Routing"]

    SYNC["Synchronous Processor<br/>REST / SOAP"]

    ASYNC["Asynchronous Processor<br/>Kafka / Worker"]

    FILE["Batch and File Processor<br/>Bulk Processing"]

    SECURITY["Security and Resilience<br/>OAuth2 | Rate Limit | Retry | Circuit Breaker | Timeout"]

    REST["Oracle SCM REST APIs"]

    SOAP["Oracle SCM SOAP Services"]

    IMPORT["Oracle SCM File Import"]

    EVENTS["Oracle SCM Business Events"]

    DATA[("Oracle SCM Business Data")]

    DLQ[["Retry Topic / DLQ"]]

    ERROR[("Error Store")]

    TITLE --> TMOBILE
    TMOBILE --> GATEWAY
    GATEWAY --> ORCHESTRATOR

    ORCHESTRATOR --> SYNC
    ORCHESTRATOR --> ASYNC
    ORCHESTRATOR --> FILE

    SYNC --> SECURITY
    ASYNC --> SECURITY
    FILE --> SECURITY

    SECURITY --> REST
    SECURITY --> SOAP
    SECURITY --> IMPORT

    ASYNC --> KAFKA[["Kafka"]]
    KAFKA --> ASYNC

    EVENTS --> KAFKA

    REST --> DATA
    SOAP --> DATA
    IMPORT --> DATA

    REST -. "Response" .-> SYNC
    SOAP -. "Response / Fault" .-> SYNC
    IMPORT -. "Import Status" .-> FILE

    SECURITY -. "Failure" .-> DLQ
    DLQ --> ERROR

    classDef title fill:#263238,stroke:#263238,stroke-width:3px,color:#FFFFFF
    classDef tmobile fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    classDef integration fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    classDef async fill:#FFF8E1,stroke:#F9A825,stroke-width:2px
    classDef security fill:#EDE7F6,stroke:#512DA8,stroke-width:2px
    classDef oracle fill:#FCE4EC,stroke:#C2185B,stroke-width:2px
    classDef database fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
    classDef error fill:#FFEBEE,stroke:#C62828,stroke-width:2px

    class TITLE title
    class TMOBILE tmobile
    class GATEWAY,ORCHESTRATOR,SYNC,FILE integration
    class ASYNC,KAFKA async
    class SECURITY security
    class REST,SOAP,IMPORT,EVENTS oracle
    class DATA,ERROR database
    class DLQ error