sequenceDiagram
    participant User
    participant Frontend as Frontend (Vue/React)
    participant Firebase as Firebase Auth
    participant Gateway as API Gateway (Go/NestJS)
    participant Redis as Redis Cache
    participant Service as Backend Service

    Note over User,Service: 1. AUTHENTICATION FLOW

    User->>Frontend: Click Login
    Frontend->>Firebase: Firebase OAuth Login
    Firebase-->>Frontend: Return idToken (Firebase JWT)
    Frontend->>Gateway: POST /auth<br/>Headers: { Authorization: Bearer <idToken> }

    Gateway->>Firebase: Verify idToken with Firebase Admin SDK
    Firebase-->>Gateway: Token valid, return user info

    Gateway->>Gateway: Generate internal JWT<br/>(user_id, email, roles, exp)
    Gateway->>Redis: Cache mapping<br/>Key: jwt_internal:<user_id><br/>Value: {firebase_token, internal_jwt, metadata}
    Redis-->>Gateway: OK

    Gateway-->>Frontend: 200 OK<br/>{ internalToken: "jwt_internal", user: {...} }

    Note over User,Service: 2. BUSINESS API CALL

    Frontend->>Gateway: POST /api/orders<br/>Headers: { Authorization: Bearer <internalJWT> }

    Gateway->>Gateway: Validate internal JWT<br/>(signature, exp, issuer)

    Gateway->>Gateway: Rate Limit Check<br/>(user_id + endpoint)

    Gateway->>Redis: GET user:rate_limit:<user_id>
    Redis-->>Gateway: { count: 5, ttl: 30 }

    alt Rate Limit Exceeded
        Gateway-->>Frontend: 429 Too Many Requests
    else Rate Limit OK
        Gateway->>Redis: INCR user:rate_limit:<user_id>
        Gateway->>Service: Forward request + internal JWT<br/>GET /orders

        Service->>Service: Validate internal JWT<br/>(verify signature, check roles)
        Service->>Service: Process business logic
        Service-->>Gateway: Response data

        Gateway-->>Frontend: JSON Response (200/400/500)
    end