flowchart TB

    %% =====================================================
    %% INCOMING TRAFFIC
    %% =====================================================

    TRAFFIC["Incoming Order Requests"]

    WAF["OCI WAF"]

    APIGW["OCI API Gateway<br/>Rate Limit / Routing"]


    %% =====================================================
    %% ORDER SERVICE
    %% =====================================================

    subgraph ORDER["Order Service - OKE"]

        ORDER_PODS["Order Service Pods<br/>Java / Spring Boot<br/>Stateless"]

        ORDER_METRICS["Order Metrics<br/>RPS / CPU / Latency<br/>Concurrency"]

    end


    %% =====================================================
    %% ORDER SERVICE AUTOSCALING
    %% =====================================================

    PROM["Prometheus<br/>Application Metrics"]

    HPA["Kubernetes HPA<br/>Order Service"]

    ORDER_POLICY["Scaling Policy<br/>Min / Max Replicas<br/>Scale-Up / Scale-Down<br/>Stabilization Window"]


    %% =====================================================
    %% KAFKA
    %% =====================================================

    KAFKA[["Kafka<br/>Order Topic<br/>Multiple Partitions"]]


    %% =====================================================
    %% SCM CONSUMER
    %% =====================================================

    subgraph SCM["SCM Integration Service - OKE"]

        CONSUMERS["SCM Consumer Pods<br/>Java / Spring Boot"]

        CONSUMER_METRICS["Consumer Metrics<br/>Lag / Throughput<br/>Processing Time"]

    end


    %% =====================================================
    %% KEDA CONSUMER SCALING
    %% =====================================================

    KEDA["KEDA<br/>Kafka Scaler"]

    KEDA_POLICY["Consumer Scaling Policy<br/>Min / Max Replicas<br/>Target Kafka Lag<br/>Cooldown / Stabilization"]


    %% =====================================================
    %% ORACLE SCM
    %% =====================================================

    ORACLE["Oracle SCM<br/>REST APIs"]


    %% =====================================================
    %% ORDER FLOW
    %% =====================================================

    TRAFFIC --> WAF
    WAF --> APIGW
    APIGW --> ORDER_PODS

    ORDER_PODS -->|"Publish Order Event"| KAFKA


    %% =====================================================
    %% ORDER SERVICE METRICS
    %% =====================================================

    ORDER_PODS --> ORDER_METRICS

    ORDER_METRICS --> PROM

    PROM --> HPA

    HPA --> ORDER_POLICY

    ORDER_POLICY -. "Increase / Decrease Order Pods" .-> ORDER_PODS


    %% =====================================================
    %% KAFKA CONSUMPTION
    %% =====================================================

    KAFKA -->|"Partitions"| CONSUMERS

    CONSUMERS -->|"SCM REST Calls"| ORACLE


    %% =====================================================
    %% CONSUMER METRICS
    %% =====================================================

    CONSUMERS --> CONSUMER_METRICS

    CONSUMER_METRICS --> KEDA

    KEDA --> KEDA_POLICY

    KEDA_POLICY -. "Increase / Decrease Consumers" .-> CONSUMERS


    %% =====================================================
    %% HIGH TRAFFIC
    %% =====================================================

    HIGH["Traffic Spike<br/>RPS ↑"]

    HIGH -. "High RPS / CPU / Latency" .-> HPA

    HIGH -. "More Events → Kafka Lag ↑" .-> KEDA


    %% =====================================================
    %% LOW TRAFFIC
    %% =====================================================

    LOW["Traffic Reduction<br/>RPS ↓"]

    LOW -. "Lower RPS / CPU" .-> HPA

    LOW -. "Kafka Lag ↓" .-> KEDA


    %% =====================================================
    %% CAPACITY GUARDRAILS
    %% =====================================================

    GUARD["Capacity Guardrails<br/>Resource Requests / Limits<br/>Min / Max Replicas<br/>Pod Disruption Budget"]

    ORDER_POLICY --> GUARD

    KEDA_POLICY --> GUARD


    %% =====================================================
    %% COLORS
    %% =====================================================

    classDef traffic fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#0D47A1;

    classDef edge fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#E65100;

    classDef service fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#1B5E20;

    classDef kafka fill:#FFF8E1,stroke:#F9A825,stroke-width:2px,color:#6D4C00;

    classDef scaling fill:#E8EAF6,stroke:#3949AB,stroke-width:2px,color:#1A237E;

    classDef oracle fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#880E4F;

    classDef high fill:#FFEBEE,stroke:#C62828,stroke-width:2px,color:#B71C1C;

    classDef low fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#1B5E20;


    class TRAFFIC traffic;

    class WAF,APIGW edge;

    class ORDER_PODS,ORDER_METRICS,CONSUMERS,CONSUMER_METRICS service;

    class KAFKA kafka;

    class PROM,HPA,ORDER_POLICY,KEDA,KEDA_POLICY,GUARD scaling;

    class ORACLE oracle;

    class HIGH high;

    class LOW low;
