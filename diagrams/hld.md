flowchart TB

    %% =====================================================
    %% USERS / PARTNERS
    %% =====================================================

    USERS["Users / Partners"]


    %% =====================================================
    %% T-MOBILE EDGE
    %% =====================================================

    subgraph EDGE["T-Mobile API / Edge Layer"]

        WAF["OCI WAF<br/>DDoS / OWASP / IP Protection"]

        APIGW["OCI API Gateway<br/>Authentication / Authorization<br/>Rate Limit / Routing"]

    end


    %% =====================================================
    %% T-MOBILE BUSINESS SERVICES
    %% =====================================================

    subgraph SERVICES["T-Mobile Business Services"]

        ORDER["Order Service<br/>Java / Spring Boot"]

        INVENTORY["Inventory Service"]

        SHIPMENT["Shipment Service"]

        PROCUREMENT["Procurement Service"]

    end


    %% =====================================================
    %% DATA
    %% =====================================================

    DB[("T-Mobile Order Database")]


    %% =====================================================
    %% EVENT STREAMING
    %% =====================================================

    KAFKA[["Kafka<br/>Enterprise Event Streaming<br/>Order / SCM Events"]]


    %% =====================================================
    %% SCM INTEGRATION
    %% =====================================================

    subgraph INTEGRATION["T-Mobile SCM Integration"]

        SCM_INT["SCM Integration Service<br/>Kafka Consumers"]

    end


    %% =====================================================
    %% T-MOBILE SECURITY
    %% =====================================================

    subgraph SECURITY["OCI Security & Secrets"]

        IAM["OCI IAM<br/>Roles / Policies<br/>Workload Identity"]

        VAULT["OCI Vault<br/>SCM Credentials<br/>Secrets / Certificates / Keys"]

    end


    %% =====================================================
    %% ORACLE SCM SECURITY
    %% =====================================================

    AUTH["Oracle Authorization Server<br/>OAuth2"]


    %% =====================================================
    %% ORACLE SCM
    %% =====================================================

    subgraph ORACLE["Oracle SCM Cloud"]

        SCM_API["Oracle SCM<br/>REST APIs"]

        SCM_DATA[("Oracle SCM<br/>Business Data")]

    end


    %% =====================================================
    %% PRIMARY FLOW
    %% =====================================================

    USERS --> WAF

    WAF --> APIGW

    APIGW --> ORDER

    APIGW --> INVENTORY

    APIGW --> SHIPMENT

    APIGW --> PROCUREMENT


    %% =====================================================
    %% DATABASE
    %% =====================================================

    ORDER -->|"Order Persistence"| DB


    %% =====================================================
    %% EVENTS
    %% =====================================================

    ORDER -->|"Order Events"| KAFKA

    INVENTORY -->|"Inventory Events"| KAFKA

    SHIPMENT -->|"Shipment Events"| KAFKA

    PROCUREMENT -->|"Procurement Events"| KAFKA


    %% =====================================================
    %% SCM INTEGRATION
    %% =====================================================

    KAFKA -->|"Async Events"| SCM_INT


    %% =====================================================
    %% OCI IAM
    %% =====================================================

    IAM -. "Authorization / Policies" .-> APIGW

    IAM -. "Workload Identity" .-> ORDER

    IAM -. "Service Identity" .-> SCM_INT


    %% =====================================================
    %% OCI VAULT
    %% =====================================================

    SCM_INT -. "Retrieve SCM Credentials" .-> VAULT

    VAULT -. "Client ID / Secret" .-> SCM_INT


    %% =====================================================
    %% OAUTH2
    %% =====================================================

    SCM_INT -->|"OAuth2 Client Credentials"| AUTH

    AUTH -->|"Short-Lived Bearer Token"| SCM_INT


    %% =====================================================
    %% ORACLE SCM API
    %% =====================================================

    SCM_INT -->|"Authorization: Bearer Token"| SCM_API

    SCM_API --> SCM_DATA


    %% =====================================================
    %% COLORS
    %% =====================================================

    classDef users fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#0D47A1;

    classDef edge fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#E65100;

    classDef service fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#1B5E20;

    classDef database fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px,color:#4A148C;

    classDef kafka fill:#FFF8E1,stroke:#F9A825,stroke-width:2px,color:#6D4C00;

    classDef integration fill:#E0F7FA,stroke:#00838F,stroke-width:2px,color:#006064;

    classDef security fill:#EDE7F6,stroke:#512DA8,stroke-width:2px,color:#311B92;

    classDef auth fill:#FFFDE7,stroke:#827717,stroke-width:2px,color:#5F5B00;

    classDef oracle fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#880E4F;


    %% =====================================================
    %% APPLY COLORS
    %% =====================================================

    class USERS users;

    class WAF,APIGW edge;

    class ORDER,INVENTORY,SHIPMENT,PROCUREMENT service;

    class DB database;

    class KAFKA kafka;

    class SCM_INT integration;

    class IAM,VAULT security;

    class AUTH auth;

    class SCM_API,SCM_DATA oracle;
