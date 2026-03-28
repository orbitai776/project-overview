sequenceDiagram
    autonumber
    participant FE as Frontend (Vue)
    participant GW as API Gateway (Golang)
    participant Admin as Admin Service (NestJS)
    participant Redis as Redis Cache
    participant Others as Các Microservices / OTLP

    FE->>GW: GET /api/v1/admin/dashboard/services (kèm JWT)
    GW->>Admin: Verify JWT -> Route Request
    
    Admin->>Redis: Check Cache (Key: dashboard_services_overview)
    
    alt Cache Hit (Có dữ liệu trong Redis)
        Redis-->>Admin: Trả về JSON tổng hợp
        Admin-->>GW: Response nhanh chóng
    else Cache Miss (Không có hoặc Hết hạn)
        par Gọi song song (Promise.all) kèm Timeout
            Admin->>Others: GET User Service Stats (Total Users, Active)
            Admin->>Others: GET Partner Service Stats (Total Partners, Tokens)
            Admin->>Others: GET Gateway/OTLP (Health, Uptime)
        end
        Others-->>Admin: Trả về số liệu cục bộ
        
        Admin->>Admin: Aggregate (Tổng hợp & tính toán lại data)
        Admin->>Redis: Lưu cục JSON vừa tổng hợp (TTL: 3 phút)
        Admin-->>GW: Trả về JSON tổng hợp
    end
    
    GW-->>FE: 200 OK
    Note over FE: Cập nhật Dashboard Metrics