# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Giáp Hoàng Thịnh |
| Mã học viên | 2A202601492 |
| Repo | https://github.com/giapthinh123/K4-Day12-2A202601492-GiapHoangThinh |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-production-c443.up.railway.app/ |
| Platform | Railway |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | redis://default:gWYLdSCXcDlcmDDvdDjHPkYFmxnTjQzt@day12-chat-redis.railway.internal:6379 |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-chat-production-c443.up.railway.app/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-chat-production-c443.up.railway.app/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST https://day12-chat-production-c443.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-chat-production-c443.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-chat-production-c443.up.railway.app/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
1. GET /healthz:
HTTP/1.1 200 OK
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

2. GET /readyz:
HTTP/1.1 200 OK
{"status":"ready","redis":true}

3. POST /chat (không có token):
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
{"detail":"invalid or missing bearer token"}

4. POST /chat (có token):
HTTP/1.1 200 OK
{"reply":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 12 lượt trao đổi trước đó.)","client_id":"sv-test","turns_before":12,"usd_cost":7.155e-05,"usage":{"prompt":297,"completion":45}}

5. Rate limit test:
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
{"detail":"rate limit exceeded"}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

