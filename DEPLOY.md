# Jopup - Deployment Repo

Monorepo deploy cho hệ thống Jopup trên VPS với Docker + Nginx.

## Kiến trúc

```
VPS (Ubuntu 22.04)
│
├── Nginx (port 80/443) ─── Reverse Proxy
│     ├── ducdev04.pro.vn        → Landing (Next.js :3000)
│     ├── admin.ducdev04.pro.vn  → Admin Panel (Vite/React :80)
│     └── api.ducdev04.pro.vn    → Backend API (.NET 8 :9000)
│
├── PostgreSQL 16
├── Redis 7.2
└── Certbot (Let's Encrypt SSL)
```

## Cấu trúc thư mục

```
.
├── docker-compose.yml
├── .env.example
├── .env                    ← tạo từ .env.example (không commit)
├── nginx/
│   ├── nginx.conf
│   ├── conf.d/
│   │   ├── landing.conf    ← ducdev04.pro.vn
│   │   ├── admin.conf      ← admin.ducdev04.pro.vn
│   │   └── api.conf        ← api.ducdev04.pro.vn
│   └── ssl/                ← SSL certs (tạo bởi certbot)
├── jobup_be/               ← .NET 8 API (submodule/clone)
├── jobup_admin/            ← React + Vite (submodule/clone)
└── jobup_client/
    └── landing/            ← Next.js (submodule/clone)
```

---

## Hướng dẫn deploy lần đầu

### 1. Chuẩn bị VPS

```bash
# Ubuntu 22.04
sudo apt update && sudo apt upgrade -y

# Cài Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Kiểm tra
docker --version
docker compose version
```

### 2. Clone repo và cấu hình

```bash
# Clone deploy repo (chứa 3 submodule)
git clone https://github.com/yourorg/jopup-deploy.git
cd jopup-deploy

# Clone/init các submodule (nếu dùng git submodule)
git submodule update --init --recursive

# Tạo file .env
cp .env.example .env
nano .env   # Điền đầy đủ thông tin thật
```

### 3. Cập nhật domain trong Nginx config

```bash
# Domain đã được cấu hình sẵn: ducdev04.pro.vn
# Nếu đổi domain khác, thay thế trong 3 file conf.d/*.conf
```

Domain hiện tại đã được cấu hình:
- `nginx/conf.d/landing.conf` → `server_name ducdev04.pro.vn www.ducdev04.pro.vn`
- `nginx/conf.d/admin.conf`   → `server_name admin.ducdev04.pro.vn`
- `nginx/conf.d/api.conf`     → `server_name api.ducdev04.pro.vn`

### 4. Deploy lần đầu (HTTP)

```bash
# Build và chạy tất cả services
docker compose up -d --build

# Kiểm tra trạng thái
docker compose ps

# Xem logs
docker compose logs -f
```

### 5. Lấy SSL Certificate (Let's Encrypt)

> **Điều kiện**: DNS của domain đã trỏ về IP của VPS

```bash
# Đảm bảo Nginx đang chạy với HTTP (bước 4)
# Chạy certbot để lấy cert
docker compose run --rm certbot

# Cert sẽ được lưu tại: ./nginx/ssl/live/ducdev04.pro.vn/
```

### 6. Bật HTTPS trong Nginx

Sau khi có cert, mở 3 file conf và **bỏ comment** block SSL:

```bash
# Bỏ comment các dòng return 301 (HTTP redirect)
# Bỏ comment toàn bộ server block 443

# Reload nginx
docker compose exec nginx nginx -s reload
```

### 7. Auto-renew SSL (crontab)

```bash
# Thêm vào crontab của VPS
crontab -e

# Renew mỗi ngày lúc 3:00 sáng
0 3 * * * cd /path/to/jopup-deploy && docker compose run --rm certbot renew && docker compose exec nginx nginx -s reload
```

---

## Các lệnh thường dùng

```bash
# Xem logs realtime
docker compose logs -f api
docker compose logs -f nginx

# Restart một service
docker compose restart api

# Rebuild khi có code mới
docker compose up -d --build api
docker compose up -d --build admin
docker compose up -d --build landing

# Rebuild toàn bộ
docker compose up -d --build

# Dừng toàn bộ
docker compose down

# Dừng và xóa volumes (CẨN THẬN - mất data DB)
docker compose down -v

# Truy cập PostgreSQL
docker compose exec postgres psql -U jopup_user -d jopup_db

# Truy cập Redis CLI
docker compose exec redis redis-cli

# Xem resource usage
docker stats
```

## Deploy code mới (CI/CD manual)

```bash
cd /path/to/jopup-deploy

# Pull code mới
git pull
git submodule update --remote

# Rebuild service cần update
docker compose up -d --build api        # backend
docker compose up -d --build admin      # admin panel
docker compose up -d --build landing    # landing page
```

## Biến môi trường quan trọng

| Biến | Mô tả | Ví dụ |
|------|--------|-------|
| `DOMAIN` | Domain chính | `jobup.vn` |
| `FE_BASE_URL` | URL landing | `https://jobup.vn` |
| `BE_BASE_URL` | URL API | `https://api.jobup.vn` |
| `POSTGRES_PASSWORD` | Mật khẩu DB | strong password |
| `JWT_KEY` | Secret key JWT (≥32 chars) | random string |
| `SMTP_PASSWORD` | Gmail App Password | 16-char app password |
