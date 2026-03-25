graph TB
    subgraph Frontend
        FE[Vue/React App]
    end

    subgraph Firebase
        FA[Firebase Auth]
    end

    subgraph Gateway[API Gateway :41001]
        AM[Auth Middleware]
        RL[Rate Limiter]
        JV[JWT Validator]
        CP[Cache Proxy]
        RP[Reverse Proxy]
    end

    subgraph Cache[Redis :6379]
        R1[(JWT Cache)]
        R2[(Rate Limit Counter)]
    end

    subgraph Services
        S1[User Service :41002]
        S2[Order Service :41003]
        S3[Product Service :41004]
    end

    FE --> FA
    FA -->|idToken| FE
    FE -->|1. POST /auth| AM
    AM -->|verify| FA
    AM -->|store| R1
    AM -->|generate internal JWT| FE

    FE -->|2. POST /api/*| JV
    JV -->|validate| R1
    JV --> RL
    RL --> R2
    RL --> CP
    CP -->|forward| RP
    RP --> S1
    RP --> S2
    RP --> S3