# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Trần Minh Hạnh |
| Mã học viên | 2A202601232 |
| Repo | https://github.com/hanhtran01/K4-Day12-Cloud-Services-And-Deployment-2A202601232-TranMinhHanh |

## Service

| Mục | Nội dung |
|---|---|
| Public URL | https://day12-chat-production-813a.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Tài liệu chỉ ghi tên biến và nguồn giá trị; không ghi giá trị secret.

| Biến | Đã set | Nguồn/ghi chú |
|---|---|---|
| `PORT` | ✅ | Railway service variable, cố định 8000 để khớp service domain |
| `API_TOKEN` | ✅ | Railway Variables, secret không nằm trong repo |
| `REDIS_URL` | ✅ | Reference `${{Redis.REDIS_URL}}` từ Redis service nội bộ Railway |
| `BUCKET_CAPACITY` | ✅ | Railway Variables |
| `REFILL_PER_MINUTE` | ✅ | Railway Variables |
| `DAILY_BUDGET_USD` | ✅ | Railway Variables |
| `LOG_LEVEL` | ✅ | Railway Variables |

## Kết Quả Kiểm Tra Thật

Kiểm tra ngày 2026-08-10:

```text
GET  /healthz                         -> 200 {"status":"ok","service":"day12-chat-service","version":"1.0.0"}
GET  /readyz                          -> 200 {"status":"ready","redis":true}
POST /chat (không Authorization)        -> 401, WWW-Authenticate: Bearer
POST /chat (Bearer token hợp lệ)     -> 200, có client_id, reply, turns_before, usage, usd_cost
Rate limit, 11 request cùng client     -> 200 200 200 200 200 200 200 200 200 200 429
```

Sự cố khi deploy: `railway.toml` ban đầu override Docker CMD bằng
`--port $PORT`. Railway không nội suy shell trong start command nên Uvicorn nhận
chuỗi literal `$PORT` và thoát. Cách sửa là bỏ override để dùng Docker
CMD có `sh -c` và `${PORT:-8000}`, sau đó set domain và service cùng port 8000.

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png`: Railway dashboard của service.
- `screenshots/healthz.png`: kết quả public `/healthz`.
