flowchart TB

    %% =========================
    %% NORMAL REQUEST FLOW
    %% =========================

    ORDER[Order Service]
    VALIDATE[Validation and Idempotency]
    CB[Resilience4j Circuit Breaker]
    RETRY[Resilience4j Retry]
    PAYMENT[Payment Service]
    DB[(Payment Database)]
    SUCCESS[Payment Success]
    ERROR[Error Handler]
    PENDING[Payment Pending]

    ORDER --> VALIDATE
    VALIDATE --> CB

    CB -->|CLOSED| RETRY
    RETRY -->|Call Payment| PAYMENT
    PAYMENT --> DB
    PAYMENT -->|2xx Success| SUCCESS
    SUCCESS --> ORDER


    %% =========================
    %% RETRY FLOW
    %% =========================

    PAYMENT -->|5xx / 429 / Timeout| RETRY
    RETRY -->|Retry with Backoff| PAYMENT

    RETRY -->|Retries Exhausted| ERROR


    %% =========================
    %% NON RETRYABLE ERRORS
    %% =========================

    VALIDATE -->|Validation Error| ERROR
    PAYMENT -->|4xx / Business Error| ERROR

    ERROR --> PENDING


    %% =========================
    %% CIRCUIT BREAKER
    %% =========================

    PAYMENT -->|Repeated Failures| CB

    CB -->|Failure Threshold Reached| OPEN

    OPEN[Circuit OPEN<br/>Fail Fast]

    OPEN -->|Reject Request| ERROR

    OPEN -->|Wait Duration Completed| HALF

    HALF[Circuit HALF-OPEN<br/>Limited Test Call]

    HALF -->|Test Call Succeeds| CLOSED
    HALF -->|Test Call Fails| OPEN

    CLOSED[Circuit CLOSED<br/>Normal Calls]

    CLOSED --> RETRY


    %% =========================
    %% ASYNC RECOVERY
    %% =========================

    PENDING -->|Async Recovery| DLQ
    DLQ[[Retry Topic / DLQ]]


    %% =========================
    %% COLORS
    %% =========================

    classDef service fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    classDef resilience fill:#FFF8E1,stroke:#F9A825,stroke-width:2px
    classDef payment fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    classDef error fill:#FFEBEE,stroke:#C62828,stroke-width:2px
    classDef state fill:#EDE7F6,stroke:#512DA8,stroke-width:2px
    classDef success fill:#E8F5E9,stroke:#388E3C,stroke-width:2px

    class ORDER,VALIDATE service
    class CB,RETRY resilience
    class PAYMENT,DB payment
    class ERROR,PENDING,DLQ error
    class OPEN,HALF,CLOSED state
    class SUCCESS success
