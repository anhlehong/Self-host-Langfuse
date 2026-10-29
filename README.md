# Langfuse self-hosted

Langfuse dùng chung cho các dự án AI, chạy local bằng Docker Compose. Mỗi dự án tạo Project và API keys riêng.

## Khởi tạo

```bash
cp .env.example .env
```

Thay tất cả giá trị `REPLACE_ME` trong `.env` bằng secret riêng; có thể tạo secret bằng `openssl rand -hex 32`. Không commit `.env`.

## Chạy và kiểm tra

```bash
docker compose up -d
docker compose ps
curl -fsS http://localhost:3000/api/public/health
```

Mở http://localhost:3000 để tạo Organization và Project. Các image trong `compose.yaml` được ghim phiên bản và digest; cập nhật chúng tại đó. `LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES` đang bật cho dịch vụ web.

## Quản lý

```bash
docker compose logs -f langfuse-web langfuse-worker
docker compose stop   # Dừng, giữ dữ liệu
docker compose start  # Chạy lại
```

Langfuse UI/API dùng cổng `3000`; MinIO dùng `9090`. Các cổng database không public ra host. Dữ liệu được giữ trong Docker volumes; chỉ dùng `docker compose down -v` khi muốn xóa dữ liệu.

Batch export dùng MinIO hiện có. Tải CSV/JSON hoặc media qua trình duyệt cần máy truy cập Langfuse kết nối được tới `localhost:9090`; nếu dùng SSH tunnel, forward cả cổng `3000` và `9090`.
