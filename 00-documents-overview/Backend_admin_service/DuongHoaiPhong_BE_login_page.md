sequenceDiagram
    autonumber
    participant FE as Frontend (Vue)
    participant GW as API Gateway (Golang)
    participant Redis as Redis Cache
    participant Admin as Admin Service (NestJS)
    participant DB as Admin DB (Supabase)

    FE->>GW: POST /login (Firebase Token)
    Note over GW: Rate Limit Check<br/>Verify Firebase Token<br/>Tạo App JWT
    GW->>Admin: Forward Request + App JWT
    
    Admin->>DB: Query Admin Profile by UID/Email
    DB-->>Admin: Trả về Record Profile & Role
    
    alt Không tìm thấy hoặc Bị khóa (Banned)
        Admin-->>GW: Xử lý lỗi (403/404)
        GW-->>FE: Báo lỗi "Tài khoản không hợp lệ"
    else Hợp lệ & Có quyền Admin
        Admin->>DB: Ghi log đăng nhập (Insert Audit Log: IP, Time)
        Admin-->>GW: Trả về Profile Admin & Permissions
        Note over GW: Cache User Session vào Redis (nếu cần)
        GW-->>FE: 200 OK + App JWT + Profile Data
    end