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
| Public URL | `http://localhost:8000` (phương án local fallback) |
| Platform | Docker Compose + Nginx + Redis (local fallback); cloud target: Render |
| Ngày kiểm tra | 28/09/2026 |

## Biến Môi Trường Đã Set

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | đặt trong `.env` cục bộ; `.env` bị Git bỏ qua |
| `AGENT_API_KEY` | ✅ | đặt trong `.env`, không nằm trong repo |
| `REDIS_URL` | ✅ | Compose dùng `redis://redis:6379/0` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
$ docker compose up -d --build --scale agent=3
redis-1   Up (healthy)
agent-1   Up (healthy)
agent-2   Up (healthy)
agent-3   Up (healthy)
nginx-1   Up (healthy), 0.0.0.0:8000->80/tcp

$ GET http://localhost:8000/health
HTTP 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}

$ GET http://localhost:8000/ready
HTTP 200 {"status":"ready","redis":true}

$ POST http://localhost:8000/ask (không có X-API-Key)
HTTP 401

$ POST /ask 15 lần với cùng X-User-Id
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429

$ POST /ask 6 lần qua Nginx tới ba agent, cùng X-User-Id
history_length: 0 2 4 6 8 10
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/health.png` — ảnh thật của `/health` trên stack local (đã có)
- `screenshots/dashboard.png` — cần chụp thủ công Docker Desktop trước khi nộp

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

Môi trường hiện tại chưa có phiên đăng nhập/tài khoản cloud để tạo service và
Redis công khai. Vì vậy bài dùng `LOCAL_FALLBACK=true`: stack đã được build và
chạy thật bằng Docker Compose, gồm ba agent sau Nginx và một Redis dùng chung.
Đây là phương án dự phòng theo đề, nên CP5 bị giới hạn tối đa 9/15 điểm.
