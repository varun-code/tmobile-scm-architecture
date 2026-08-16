flowchart TB

    %% =====================================================
    %% EXTERNAL ACTORS
    %% =====================================================

    USERS["Users / Partners"]


    %% =====================================================
    %% T-MOBILE SYSTEM
    %% =====================================================

    TMOBILE["T-Mobile<br/>Microservices"]


    %% =====================================================
    %% INTEGRATION SYSTEM
    %% =====================================================

    INTEGRATION["SCM Integration<br/>Platform"]


    %% =====================================================
    %% ORACLE SCM
    %% =====================================================

    ORACLE["Oracle SCM Cloud<br/>Supply Chain Platform"]


    %% =====================================================
    %% MAIN BUSINESS FLOW
    %% =====================================================

    USERS -->|"Business Requests"| TMOBILE

    TMOBILE -->|"Integration Events"| INTEGRATION

    INTEGRATION -->|"Secure SCM Integration"| ORACLE


    %% =====================================================
    %% RESPONSE / STATUS
    %% =====================================================

    ORACLE -. "SCM Responses / Status" .-> INTEGRATION

    INTEGRATION -. "Business Status" .-> TMOBILE

    TMOBILE -. "Business Response" .-> USERS


    %% =====================================================
    %% COLORS
    %% =====================================================

    classDef external fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#0D47A1;

    classDef tmobile fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#1B5E20;

    classDef integration fill:#E0F7FA,stroke:#00838F,stroke-width:2px,color:#006064;

    classDef oracle fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#880E4F;


    class USERS external;

    class TMOBILE tmobile;

    class INTEGRATION integration;

    class ORACLE oracle;
