# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lê Văn Tài |
| Mã học viên | 2A202602464 |
| Repo | https://github.com/taitottinhday/K4-L3A-DAY12-LeVanTai-2A202602464-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | `https://day12-agent-production-49e3.up.railway.app` |
| Platform | Railway — FastAPI service và Redis cùng private network |
| Ngày kiểm tra | 28/09/2026 |

## Biến Môi Trường Đã Set

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự cấp lúc runtime; không hard-code |
| `AGENT_API_KEY` | ✅ | Railway secret; giá trị không lưu trong repository |
| `REDIS_URL` | ✅ | Reference tới `day12-redis.REDIS_URL` trên private network |
| `RATE_LIMIT_PER_MINUTE` | ✅ | `10` |
| `MONTHLY_BUDGET_USD` | ✅ | `10.0` |
| `LOG_LEVEL` | ✅ | `INFO` |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | ✅ | `30`, hỗ trợ graceful shutdown |

## Lệnh Kiểm Tra

Các lệnh dưới đây dùng Public URL thật ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-49e3.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-production-49e3.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-49e3.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-production-49e3.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-production-49e3.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
$ GET https://day12-agent-production-49e3.up.railway.app/health
HTTP 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}

$ GET https://day12-agent-production-49e3.up.railway.app/ready
HTTP 200 {"status":"ready","redis":true}

$ POST https://day12-agent-production-49e3.up.railway.app/ask (không có X-API-Key)
HTTP 401

$ POST https://day12-agent-production-49e3.up.railway.app/ask
  X-API-Key: [REDACTED]
  X-User-Id: deployment-check
HTTP 200
{"user_id":"deployment-check","history_length":0,"cost_usd":0.0000237,
 "tokens":{"in":6,"out":38}}

$ POST /ask 15 lần với cùng một X-User-Id trên Railway
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/health.png` — bằng chứng kiểm tra endpoint `/health`
- Ảnh Railway dashboard cho thấy `day12-agent` và `day12-redis` đã được kiểm tra
  trực tiếp ngày 28/09/2026.

---

## Trạng Thái Triển Khai

Deployment cloud thật đã hoàn tất, không dùng phương án `LOCAL_FALLBACK`.
`day12-agent` truy cập Redis qua private network của Railway; chỉ API FastAPI
được công khai qua HTTPS. Liveness, readiness và xác thực API key đều đã được
kiểm tra trực tiếp trên public domain.
