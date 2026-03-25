sequenceDiagram
    participant User
    participant Frontend as Frontend (Vue/React)
    participant Firebase as Firebase Auth
    participant Gateway as API Gateway (Go/NestJS)
    participant Redis as Redis Cache
    participant Database as Database (PostgreSQL/MySQL)
    participant Service as Backend Service

    Note over User,Service: 1. AUTHENTICATION FLOW

    User->>Frontend: Click Login
    Frontend->>Firebase: Firebase OAuth Login
    Firebase-->>Frontend: Return idToken (Firebase JWT)
    Frontend->>Gateway: POST /auth<br/>Headers: { Authorization: Bearer <idToken> }

    Gateway->>Firebase: Verify idToken with Firebase Admin SDK
    Firebase-->>Gateway: Token valid, return user info<br/>(uid, email, name, picture)

    Gateway->>Database: SELECT * FROM users WHERE firebase_uid = ?
    Database-->>Gateway: User exists? (Yes/No)

    alt User not exists
        Gateway->>Database: INSERT INTO users<br/>(firebase_uid, email, name, picture, created_at)
        Database-->>Gateway: User created (user_id)
    else User exists
        Gateway->>Database: UPDATE users<br/>(last_login_at, updated_at)
        Database-->>Gateway: User updated
    end

    Gateway->>Database: SELECT user_id, email, roles, subscription_tier,<br/>total_usage, token_balance, rate_limit_config<br/>FROM users WHERE firebase_uid = ?
    Database-->>Gateway: Full user data

    Gateway->>Gateway: Generate internal JWT<br/>(user_id, email, roles, subscription_tier, exp)

    Gateway->>Redis: Cache user data<br/>Key: user:info:<user_id><br/>Value: {user_id, email, roles, subscription_tier, total_usage, token_balance}<br/>TTL: 3600s
    Redis-->>Gateway: OK

    Gateway->>Redis: Initialize/Cache rate limit<br/>Key: rate_limit:<user_id><br/>Value: {limit, window, current_count}<br/>TTL: based on window
    Redis-->>Gateway: OK

    Gateway->>Redis: Cache mapping<br/>Key: jwt_internal:<user_id><br/>Value: {internal_jwt, firebase_uid, metadata}<br/>TTL: 3600s
    Redis-->>Gateway: OK

    Gateway-->>Frontend: 200 OK<br/>{ internalToken: "jwt_internal", user: {...} }

    Note over User,Service: 2. BUSINESS API CALL

    Frontend->>Gateway: POST /api/orders<br/>Headers: { Authorization: Bearer <internalJWT> }

    Gateway->>Gateway: Validate internal JWT<br/>(signature, exp, issuer, audience)

    Gateway->>Redis: GET user:info:<user_id>
    Redis-->>Gateway: User info (cached)

    alt Cache Miss
        Gateway->>Database: SELECT user data
        Database-->>Gateway: User data
        Gateway->>Redis: SET user:info:<user_id>
    end

    Gateway->>Redis: GET rate_limit:<user_id>
    Redis-->>Gateway: Rate limit data

    alt Rate Limit Exceeded
        Gateway-->>Frontend: 429 Too Many Requests
    else Rate Limit OK
        Gateway->>Redis: INCR rate_limit:<user_id>:counter
        Gateway->>Redis: EXPIRE rate_limit:<user_id>:counter window
        
        Gateway->>Gateway: Check token_balance / quota
        
        alt Insufficient quota/tokens
            Gateway-->>Frontend: 403 Forbidden (Quota exceeded)
        else Quota OK
            Gateway->>Service: Forward request + internal JWT<br/>GET /orders<br/>Headers: { X-User-ID, X-User-Roles, X-Subscription-Tier }
            
            Service->>Service: Validate internal JWT<br/>(verify signature, check roles)
            Service->>Service: Process business logic
            Service-->>Gateway: Response data
            
            Gateway->>Redis: DECR user:info:<user_id>:token_balance (if using token model)
            Gateway->>Database: ASYNC UPDATE user_usage_logs<br/>(increment usage counter)
            
            Gateway-->>Frontend: JSON Response (200/400/500)
        end
    end