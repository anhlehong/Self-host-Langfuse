# Langfuse (Self-hosted)

Langfuse dùng chung cho nhiều dự án AI, chạy local bằng Docker Compose. Mỗi dự án tạo **Project** và API keys riêng trong cùng một Langfuse instance.

## 1. Cấu trúc

```text
~/Work/shared/services/langfuse/
├── compose.yaml     # 6 services, tên container langfuse-*
├── .env.example     # Mẫu cấu hình; không chứa secret thực tế
├── .env             # Secret thực tế (không commit)
├── .gitignore
└── README.md
```

## 2. Cấu hình lần đầu

```bash
cd ~/Work/shared/services/langfuse
```

**Cách thủ công:** Copy mẫu rồi thay **tất cả** giá trị `REPLACE_ME` (kể cả `REPLACE_WITH_64_HEX_CHARACTERS`) bằng secret thực tế:

```bash
cp -n .env.example .env
nano .env
chmod 600 .env
```

Có thể tạo secret bằng `openssl rand -hex 32`. **Không dùng nguyên `.env.example` để chạy.**

**Hoặc tạo `.env` tự động** (chỉ lần đầu; không ghi đè nếu đã tồn tại):

```bash
if [ -e .env ]; then
  echo '.env already exists; not overwriting.'
else
  umask 077
  cat > .env <<EOF_ENV
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=$(openssl rand -hex 32)
SALT=$(openssl rand -hex 32)
ENCRYPTION_KEY=$(openssl rand -hex 32)
POSTGRES_PASSWORD=$(openssl rand -hex 24)
CLICKHOUSE_PASSWORD=$(openssl rand -hex 24)
REDIS_AUTH=$(openssl rand -hex 24)
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=$(openssl rand -hex 24)
EOF_ENV
fi
```

**Chọn một trong hai cách.** Giữ nguyên `.env` sau khi đã khởi tạo; đổi mật khẩu/khóa mã hóa có thể làm mất quyền truy cập dữ liệu cũ.

## 3. Khởi động

```bash
docker compose config --quiet  # Kiểm tra Compose
docker compose up -d           # Tạo và chạy 6 container
docker compose ps              # Xem trạng thái
curl -fsS http://localhost:3000/api/public/health
```

Mở **http://localhost:3000** → tạo user, Organization, Project → lấy API keys cho từng dự án.

## 4. Lệnh quản lý

```bash
docker compose stop                              # Dừng, giữ dữ liệu
docker compose start                             # Chạy lại
docker compose logs -f langfuse-web langfuse-worker  # Xem log
docker compose ps                                # Xem trạng thái
docker ps -a --filter 'name=langfuse-'           # Tìm container
```

Các container: `langfuse-web`, `langfuse-worker`, `langfuse-postgres`, `langfuse-clickhouse`, `langfuse-redis`, `langfuse-minio`.

## 5. Xuất CSV/JSON và API

- `compose.yaml` đã bật `LANGFUSE_S3_BATCH_EXPORT_ENABLED: "true"` (dùng MinIO hiện có). Không cần sửa `.env`.
- Trong dashboard: vào **Traces/Observations** → chọn bộ lọc → **Export** → chọn CSV/JSON.
- REST API có sẵn: `http://localhost:3000/api/docs` (đăng nhập API bằng Public Key + Secret Key của project).
- Đã chỉnh `compose.yaml` hoặc `.env`? Áp dụng mà không xóa dữ liệu:

```bash
docker compose config --quiet
docker compose up -d --no-deps --force-recreate langfuse-web langfuse-worker
curl -fsS http://localhost:3000/api/public/health
```

- Tải CSV/JSON hoặc media qua trình duyệt cần truy cập được `localhost:9090` trên **máy dùng trình duyệt**. Nếu truy cập Langfuse qua SSH tunnel, forward cả cổng `3000` và `9090`.
- ###### 5.1 Xác nhận Batch Export đã bật:

```bash
docker exec langfuse-web printenv LANGFUSE_S3_BATCH_EXPORT_ENABLED
docker exec langfuse-worker printenv LANGFUSE_S3_BATCH_EXPORT_ENABLED
```

- Cả 2 phải trả về true

## 6. Kết nối và lưu ý

- Langfuse API/UI: `http://localhost:3000` (chỉ bind vào host `127.0.0.1`).
- MinIO API: `127.0.0.1:9090`; PostgreSQL/Redis/ClickHouse không public ra host.
- Ứng dụng trên máy khác hoặc trong container khác cần URL/network truy cập được tới Langfuse; `localhost` của container không phải localhost của host.
- `.env` chứa secrets, **không commit**. `docker compose stop` giữ lại dữ liệu trong volumes.
- **Không chạy `docker compose down -v`** trừ khi chủ động muốn xóa dữ liệu Langfuse.
- Langfuse để trace AI (spans, latency, errors), không tự thu thập mọi dòng stdout của ứng dụng.
