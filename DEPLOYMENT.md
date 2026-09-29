# Triển khai Render — CP5

## Thông tin học viên

| Mục | Nội dung |
| --- | --- |
| Họ và tên | Phạm Hoàng Trọng |
| Mã học viên | 2A202602765 |
| Repository | https://github.com/ToRong31/K4-L3B-PhamHoangTrong-2A202602765-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
| --- | --- |
| Platform | Render Blueprint |
| Public URL | https://day12-agent-i0vc.onrender.com |
| Web service | `day12-agent` (Free, Docker) |
| Key Value | `day12-redis` (Free) |
| Commit đã deploy | `07c8263` |
| Ngày deploy | 2026-09-29 |

## Biến môi trường trên Render

| Tên biến | Nguồn |
| --- | --- |
| `AGENT_API_KEY` | Nhập trong Render khi tạo Blueprint; `sync: false`, không lưu trong repo |
| `REDIS_URL` | Render lấy `connectionString` nội bộ từ `day12-redis` |
| `RATE_LIMIT_PER_MINUTE` | Cấu hình trong `render.yaml`: `10` |
| `MONTHLY_BUDGET_USD` | Cấu hình trong `render.yaml`: `10.0` |
| `LOG_LEVEL` | Cấu hình trong `render.yaml`: `INFO` |
| `PORT` | Không tự ghi đè; Dockerfile đọc biến do platform cấp, mặc định `8000` |

## Kết quả kiểm tra URL thật

Thực hiện ngày 2026-09-29 từ máy cục bộ:

```text
GET  https://day12-agent-i0vc.onrender.com/health
HTTP 200  {"status":"ok","service":"day12-agent","version":"1.0.0"}

GET  https://day12-agent-i0vc.onrender.com/ready
HTTP 200  {"status":"ready","redis":true}

POST https://day12-agent-i0vc.onrender.com/ask
Body: {"question":"Hello"}; không gửi X-API-Key
HTTP 401  {"detail":"invalid or missing API key"}

POST https://day12-agent-i0vc.onrender.com/ask
Body: {"question":"Hello"}; gửi X-API-Key từ DEPLOY_API_KEY và X-User-Id: cp5-auth-check
HTTP 200  user_id=cp5-auth-check; có câu trả lời
```

Có thể kiểm tra lại bằng PowerShell:

```powershell
curl.exe -i https://day12-agent-i0vc.onrender.com/health
curl.exe -i https://day12-agent-i0vc.onrender.com/ready
curl.exe -i -X POST https://day12-agent-i0vc.onrender.com/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```

Đã chạy bài kiểm tra `/ask` với key hợp lệ qua `DEPLOY_API_KEY` trong `.env` cục bộ. Giá trị key không nằm trong tài liệu hoặc Git.

## Ảnh minh chứng

- `screenshots/dashboard.png`: dashboard Render, service ở trạng thái Live và có public URL.
- `screenshots/health.png`: phản hồi `/health` trên URL thật.

## Lưu ý về gói Free

Render Free Web Service có thể ngủ khi không hoạt động nên request đầu tiên có thể chậm. Render Free Key Value không có persistence; dữ liệu Redis có thể mất khi instance Key Value khởi động lại. Nguồn: https://render.com/docs/free
