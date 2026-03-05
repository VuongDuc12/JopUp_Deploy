# Jopup - Hướng dẫn Deploy VPS (Docker + Nginx + HTTPS)

> **Cập nhật**: 2026-03-05 — Fix 502 Bad Gateway, chuẩn hóa HTTPS, đổi domain mới.

## Kiến trúc hệ thống

```
Internet
   │
   ▼
┌──────────────────────────────────────────────────────────────────┐
│  VPS (Ubuntu 22.04)                                              │
│                                                                  │
│  Nginx (port 80 → redirect HTTPS, port 443 → SSL termination)   │
│     ├── jobuptest.ducdev04.pro.vn        → landing:3000          │
│     ├── admin-jobuptest.ducdev04.pro.vn  → admin:80              │
│     └── api-jobuptest.ducdev04.pro.vn    → api:9000              │
│                                                                  │
│  [landing]   Next.js standalone    (internal :3000)              │
│  [admin]     React/Vite → Nginx   (internal :80)                │
│  [api]       .NET 8 Kestrel       (internal :9000)              │
│  [postgres]  PostgreSQL 16        (internal :5432)              │
│  [redis]     Redis 7.2            (internal :6379)              │
│  [certbot]   Let's Encrypt        (run on-demand)              │
│                                                                  │
│  Network: jopup_internal (bridge)                                │
│  SSL: Let's Encrypt via Certbot + webroot challenge              │
└──────────────────────────────────────────────────────────────────┘
```

**Điểm quan trọng:**
- SSL terminate tại Nginx → các container nội bộ giao tiếp qua HTTP
- API container **không** dùng `UseHttpsRedirection` (đã xóa) → tránh redirect loop
- `VITE_API_BASE_URL` và `NEXT_PUBLIC_API_URL` embed vào bundle lúc build → phải rebuild admin/landing khi đổi URL

## Cấu trúc thư mục

```
.
├── docker-compose.yml          ← Orchestration toàn bộ services
├── .env.example                ← Template biến môi trường
├── .env                        ← Biến thật (KHÔNG commit, tạo từ .env.example)
├── DEPLOY.md                   ← File này
├── nginx/
│   ├── nginx.conf              ← Config chính (gzip, headers, include conf.d)
│   ├── conf.d/                 ← Phase 1: HTTP-only configs (dùng lấy cert)
│   │   ├── landing.conf
│   │   ├── admin.conf
│   │   └── api.conf
│   ├── conf.d-ssl/             ← Phase 2: HTTPS configs (copy đè conf.d/ sau khi có cert)
│   │   ├── landing.conf
│   │   ├── admin.conf
│   │   └── api.conf
│   └── ssl/                    ← SSL certs (tạo bởi certbot, KHÔNG commit)
├── jobup_be/                   ← .NET 8 API source
├── jobup_admin/                ← React + Vite admin panel source
└── jobup_client/
    └── landing/                ← Next.js landing page source
```

---

## PHẦN 1: Chuẩn bị VPS

### 1.1. Cài đặt Docker

```bash
# Ubuntu 22.04+
sudo apt update && sudo apt upgrade -y

# Cài Docker Engine + Docker Compose plugin
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Đăng nhập lại để áp dụng group mới
# (logout/login hoặc dùng lệnh dưới)
newgrp docker

# Kiểm tra cài đặt
docker --version           # Docker version 24.x+
docker compose version     # Docker Compose version v2.x+
```

### 1.2. Mở firewall

```bash
# Cho phép HTTP + HTTPS (cần cho certbot + web)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp      # SSH
sudo ufw enable
sudo ufw status
```

### 1.3. Cấu hình DNS

Truy cập bảng quản lý domain (ví dụ: Cloudflare, Namecheap, v.v.) và tạo **4 bản ghi A Record** trỏ về IP VPS:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `jobuptest.ducdev04.pro.vn` | `<IP_VPS>` | Auto |
| A | `www.jobuptest.ducdev04.pro.vn` | `<IP_VPS>` | Auto |
| A | `admin-jobuptest.ducdev04.pro.vn` | `<IP_VPS>` | Auto |
| A | `api-jobuptest.ducdev04.pro.vn` | `<IP_VPS>` | Auto |

> **Quan trọng**: Chờ DNS propagate (5-30 phút) trước khi chạy certbot. Kiểm tra:
> ```bash
> # Kiểm tra DNS đã trỏ đúng chưa
> dig +short jobuptest.ducdev04.pro.vn
> dig +short admin-jobuptest.ducdev04.pro.vn
> dig +short api-jobuptest.ducdev04.pro.vn
> # Phải trả về IP VPS
> ```

---

## PHẦN 2: Clone repo & Cấu hình

### 2.1. Clone source code

```bash
# Clone deploy repo
git clone <your-repo-url> jopup-deploy
cd jopup-deploy

# Nếu dùng git submodule
git submodule update --init --recursive
```

### 2.2. Tạo file .env

```bash
cp .env.example .env
nano .env
```

**Nội dung `.env` cần điền (ví dụ):**

```env
# ─── Domain ────────────────────────────────────────────────────────────────────
DOMAIN=jobuptest.ducdev04.pro.vn
# ⚠️ Phase 1: Dùng http:// trước (chưa có SSL cert)
# ⚠️ Phase 2: Đổi thành https:// sau khi certbot thành công, rồi rebuild admin + landing
FE_BASE_URL=http://jobuptest.ducdev04.pro.vn
BE_BASE_URL=http://api-jobuptest.ducdev04.pro.vn

# ─── PostgreSQL ────────────────────────────────────────────────────────────────
POSTGRES_DB=jopup_db
POSTGRES_USER=jopup_user
POSTGRES_PASSWORD=<MẬT_KHẨU_MẠNH>       # ← Đổi! Ví dụ: openssl rand -base64 24

# ─── JWT ───────────────────────────────────────────────────────────────────────
JWT_KEY=<KEY_NGẪU_NHIÊN_32_KÝ_TỰ>       # ← Đổi! Tạo: openssl rand -base64 32
JWT_EXPIRED_HOURS=168
JWT_ISSUER=Jopup
JWT_AUDIENCE=Client

# ─── SMTP (Gmail App Password) ────────────────────────────────────────────────
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com        # ← Đổi thành email thật
SMTP_PASSWORD=xxxx xxxx xxxx xxxx         # ← Gmail App Password (16 ký tự)

# ─── Certbot ──────────────────────────────────────────────────────────────────
CERTBOT_EMAIL=your-email@gmail.com        # ← Email nhận thông báo cert
```

> **Lưu ý bảo mật**:
> - **KHÔNG** commit file `.env` lên git
> - Đổi tất cả giá trị mặc định (`jopup_password`, `Lt6JvZbU15auGycWskmfIRXcA4EjBScU`) thành giá trị ngẫu nhiên mạnh
> - Gmail App Password: Vào Google Account → Security → 2-Step Verification → App passwords

### 2.3. Tạo thư mục SSL (cần tồn tại trước khi mount)

```bash
mkdir -p nginx/ssl
```

---

## PHẦN 3: Deploy Phase 1 — HTTP (để lấy SSL cert)

> **Mục đích**: Chạy tất cả services qua HTTP trước, để certbot có thể xác thực domain qua webroot challenge.

### 3.1. Build và chạy tất cả services

```bash
# Build tất cả images và start containers
docker compose up -d --build

# Theo dõi quá trình build (có thể mất 3-10 phút lần đầu)
docker compose logs -f
# Ctrl+C để thoát xem logs
```

### 3.2. Kiểm tra trạng thái services

```bash
docker compose ps
```

**Kết quả mong đợi** — tất cả services phải ở trạng thái `running` hoặc `Up (healthy)`:

```
NAME              STATUS
jopup_postgres    Up (healthy)
jopup_redis       Up (healthy)
jopup_api         Up (healthy)     ← chờ ~40s để healthy
jopup_admin       Up
jopup_landing     Up
jopup_nginx       Up
```

### 3.3. Debug nếu container crash

```bash
# Xem logs từng service
docker compose logs api        # Backend API
docker compose logs admin      # Admin panel
docker compose logs landing    # Landing page
docker compose logs nginx      # Nginx proxy
docker compose logs postgres   # Database

# Kiểm tra chi tiết container
docker inspect jopup_api | grep -A 5 "State"
```

**Lỗi phổ biến:**

| Triệu chứng | Nguyên nhân | Giải pháp |
|-------------|-------------|-----------|
| `api` restart loop | DB chưa ready | Chờ postgres healthy, restart: `docker compose restart api` |
| `landing` crash | Build lỗi Next.js | Check `docker compose logs landing`, đảm bảo `output: "standalone"` trong `next.config.ts` |
| `admin` crash | Build lỗi Vite | Check `docker compose logs admin`, fix source nếu có lỗi build |
| `nginx` crash | Config sai | `docker compose logs nginx`, check cú pháp nginx conf |

### 3.4. Verify HTTP hoạt động

```bash
# Từ VPS
curl -I http://jobuptest.ducdev04.pro.vn
# Mong đợi: HTTP/1.1 200 OK

curl -I http://admin-jobuptest.ducdev04.pro.vn
# Mong đợi: HTTP/1.1 200 OK

curl -I http://api-jobuptest.ducdev04.pro.vn/swagger/index.html
# Mong đợi: HTTP/1.1 200 OK

# Test API endpoint
curl http://api-jobuptest.ducdev04.pro.vn/health/redis
# Mong đợi: "Healthy"
```

> **Nếu nhận 502 Bad Gateway**: container backend chưa start xong hoặc crash. Kiểm tra `docker compose logs api`.

### 3.5. Lấy SSL Certificate (Let's Encrypt)

```bash
# Chạy certbot (chỉ cần 1 lần)
docker compose --profile certbot run --rm certbot
```

**Kết quả thành công:**
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/jobuptest.ducdev04.pro.vn/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/jobuptest.ducdev04.pro.vn/privkey.pem
```

**Nếu certbot thất bại:**

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `DNS problem: NXDOMAIN` | DNS chưa propagate | Chờ thêm, kiểm tra `dig +short domain` |
| `Connection refused` | Port 80 bị chặn | Kiểm tra firewall: `sudo ufw status` |
| `Timeout during connect` | Nginx không chạy | `docker compose ps nginx`, restart nếu cần |
| `Too many certificates` | Rate limit Let's Encrypt | Chờ 1 tuần, hoặc dùng staging: thêm `--staging` vào certbot command |

**Verify cert tồn tại:**
```bash
ls -la nginx/ssl/live/jobuptest.ducdev04.pro.vn/
# Phải thấy: fullchain.pem, privkey.pem, cert.pem, chain.pem
```

---

## PHẦN 4: Deploy Phase 2 — Bật HTTPS

> **Repo đã có sẵn 2 bộ Nginx config:**
> - `nginx/conf.d/` — Phase 1 (HTTP-only, dùng để lấy cert)
> - `nginx/conf.d-ssl/` — Phase 2 (HTTPS + HTTP→HTTPS redirect)
>
> **Không cần nano/vim sửa từng dòng!** Chỉ cần copy đè.

### 4.1. Copy configs HTTPS đè lên configs HTTP (1 lệnh!)

```bash
# Copy 3 file HTTPS config đè lên conf.d/
cp nginx/conf.d-ssl/* nginx/conf.d/
```

Kiểm tra config hợp lệ:
```bash
docker compose exec nginx nginx -t
# Mong đợi: nginx: configuration file ... test is successful
```

### 4.2. Cập nhật .env sang HTTPS (1 lệnh!)

```bash
# Đổi http:// → https:// trong .env bằng sed
sed -i 's|FE_BASE_URL=http://|FE_BASE_URL=https://|' .env
sed -i 's|BE_BASE_URL=http://|BE_BASE_URL=https://|' .env
```

Verify:
```bash
grep '_URL=' .env
# Mong đợi:
# FE_BASE_URL=https://jobuptest.ducdev04.pro.vn
# BE_BASE_URL=https://api-jobuptest.ducdev04.pro.vn
```

### 4.3. Rebuild admin + landing + reload Nginx

> **Tại sao phải rebuild?** Vì `VITE_API_BASE_URL` (admin) và `NEXT_PUBLIC_API_URL` (landing) được embed vào JavaScript bundle lúc build thông qua Docker build args. Khi đổi URL từ `http://` sang `https://`, phải build lại để frontend gọi API đúng URL.

```bash
# Rebuild admin + landing (embed URL mới) + reload Nginx config SSL
docker compose up -d --build admin landing
docker compose exec nginx nginx -s reload
```

### 4.4. Verify HTTPS hoạt động

```bash
# Test HTTPS
curl -I https://jobuptest.ducdev04.pro.vn
# Mong đợi: HTTP/2 200

curl -I https://admin-jobuptest.ducdev04.pro.vn
# Mong đợi: HTTP/2 200

curl -I https://api-jobuptest.ducdev04.pro.vn/swagger/index.html
# Mong đợi: HTTP/2 200

# Test HTTP → HTTPS redirect
curl -I http://jobuptest.ducdev04.pro.vn
# Mong đợi: HTTP/1.1 301 Moved Permanently
#           Location: https://jobuptest.ducdev04.pro.vn/

# Test SSL cert
curl -vI https://jobuptest.ducdev04.pro.vn 2>&1 | grep "subject\|expire"
# Mong đợi: subject và expire date hợp lệ
```

### 4.5. Verify trên trình duyệt

1. **Landing**: Mở `https://jobuptest.ducdev04.pro.vn` → Hiển thị trang chủ, có khóa xanh SSL
2. **Admin**: Mở `https://admin-jobuptest.ducdev04.pro.vn` → Hiển thị trang login admin
3. **API Swagger**: Mở `https://api-jobuptest.ducdev04.pro.vn/swagger` → Hiển thị Swagger UI
4. **Admin → API**: Login thử trên admin → không bị lỗi CORS, không bị Mixed Content

---

## PHẦN 5: Auto-renew SSL Certificate

Let's Encrypt cert hết hạn sau 90 ngày. Setup auto-renew:

```bash
# Mở crontab
crontab -e

# Thêm dòng này (renew mỗi ngày lúc 3:00 sáng)
0 3 * * * cd /path/to/jopup-deploy && docker compose --profile certbot run --rm certbot renew --quiet && docker compose exec nginx nginx -s reload >> /var/log/certbot-renew.log 2>&1
```

> **Thay** `/path/to/jopup-deploy` bằng đường dẫn thực tế đến thư mục deploy.

**Kiểm tra crontab:**
```bash
crontab -l
```

**Test renew (dry run):**
```bash
docker compose --profile certbot run --rm certbot renew --dry-run
```

---

## PHẦN 6: Các lệnh thường dùng

### Quản lý services

```bash
# Xem trạng thái tất cả services
docker compose ps

# Xem logs realtime
docker compose logs -f              # Tất cả
docker compose logs -f api          # Chỉ API
docker compose logs -f nginx        # Chỉ Nginx
docker compose logs -f admin        # Chỉ Admin
docker compose logs -f landing      # Chỉ Landing
docker compose logs --tail=100 api  # 100 dòng cuối

# Restart một service (không rebuild)
docker compose restart api
docker compose restart nginx

# Dừng toàn bộ
docker compose down

# ⚠️ Dừng + XÓA toàn bộ volumes (MẤT DATA DB!)
docker compose down -v
```

### Deploy code mới

```bash
cd /path/to/jopup-deploy

# Pull code mới
git pull
git submodule update --remote

# Rebuild service cần update
docker compose up -d --build api        # Chỉ rebuild backend
docker compose up -d --build admin      # Chỉ rebuild admin panel
docker compose up -d --build landing    # Chỉ rebuild landing page

# Rebuild toàn bộ
docker compose up -d --build
```

> **Lưu ý**: Nếu đổi `BE_BASE_URL` hoặc `FE_BASE_URL` trong `.env`, phải rebuild **admin** và **landing** vì URL được embed lúc build.

### Truy cập database & cache

```bash
# PostgreSQL
docker compose exec postgres psql -U jopup_user -d jopup_db

# Redis CLI
docker compose exec redis redis-cli

# Xem resource usage
docker stats
```

### Nginx

```bash
# Test config syntax (chạy trước khi reload!)
docker compose exec nginx nginx -t

# Reload config (không downtime)
docker compose exec nginx nginx -s reload

# Xem access log
docker compose exec nginx tail -f /var/log/nginx/access.log

# Xem error log
docker compose exec nginx tail -f /var/log/nginx/error.log
```

---

## PHẦN 7: Troubleshooting

### 502 Bad Gateway

| Nguyên nhân | Kiểm tra | Giải pháp |
|-------------|----------|-----------|
| Container backend crash/chưa ready | `docker compose ps api` | `docker compose restart api`, chờ healthy |
| Container admin crash | `docker compose ps admin` | `docker compose logs admin`, fix build error |
| Container landing crash | `docker compose ps landing` | `docker compose logs landing`, fix build error |
| Nginx config sai | `docker compose exec nginx nginx -t` | Sửa config, reload |
| Port không khớp | `docker compose logs nginx` | Verify proxy_pass port match container EXPOSE |

### CORS bị block

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Origin thiếu trong CORS policy | Thêm domain vào `ServiceExtensions.cs` → `WithOrigins(...)`, rebuild api |
| Mixed Content (HTTP API từ HTTPS page) | Đảm bảo `BE_BASE_URL` dùng `https://`, rebuild admin + landing |
| `AllowCredentials` + wildcard origin | Không dùng `AllowAnyOrigin()` cùng `AllowCredentials()` (đã xử lý) |

### SSL/HTTPS

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Certbot fail: DNS chưa propagate | Chờ DNS, kiểm tra: `dig +short domain` |
| Cert hết hạn | `docker compose --profile certbot run --rm certbot renew` |
| Nginx không nhận cert mới | `docker compose exec nginx nginx -s reload` |
| Redirect loop (HTTP ↔ HTTPS) | Kiểm tra không còn `UseHttpsRedirection` trong backend code |
| Browser cache redirect cũ | Clear cache hoặc test incognito |

### Trang trắng (Blank page) trên Admin

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| JS bundle load lỗi | F12 → Console, kiểm tra lỗi 404 cho .js files |
| `VITE_API_BASE_URL` sai | Kiểm tra `.env`, rebuild: `docker compose up -d --build admin` |
| Nginx không serve SPA routing | Kiểm tra `try_files $uri $uri/ /index.html` trong `jobup_admin/nginx.conf` |

---

## PHẦN 8: Biến môi trường

### Biến bắt buộc trong `.env`

| Biến | Mô tả | Ví dụ |
|------|--------|-------|
| `DOMAIN` | Domain chính | `jobuptest.ducdev04.pro.vn` |
| `FE_BASE_URL` | URL landing (HTTPS) | `https://jobuptest.ducdev04.pro.vn` |
| `BE_BASE_URL` | URL API (HTTPS) | `https://api-jobuptest.ducdev04.pro.vn` |
| `POSTGRES_DB` | Tên database | `jopup_db` |
| `POSTGRES_USER` | DB username | `jopup_user` |
| `POSTGRES_PASSWORD` | DB password (mạnh!) | `openssl rand -base64 24` |
| `JWT_KEY` | Secret key JWT (≥32 chars) | `openssl rand -base64 32` |
| `JWT_EXPIRED_HOURS` | Thời gian hết hạn JWT | `168` (7 ngày) |
| `JWT_ISSUER` | JWT Issuer | `Jopup` |
| `JWT_AUDIENCE` | JWT Audience | `Client` |
| `SMTP_HOST` | SMTP server | `smtp.gmail.com` |
| `SMTP_PORT` | SMTP port | `587` |
| `SMTP_USERNAME` | Email gửi | `your-email@gmail.com` |
| `SMTP_PASSWORD` | Gmail App Password | `xxxx xxxx xxxx xxxx` |
| `CERTBOT_EMAIL` | Email cho Let's Encrypt | `your-email@gmail.com` |

### Biến embed lúc build (phải rebuild khi đổi)

| Build Arg | Dùng bởi | Lấy từ `.env` |
|-----------|----------|----------------|
| `VITE_API_BASE_URL` | Admin panel (Vite) | `${BE_BASE_URL}` |
| `NEXT_PUBLIC_API_URL` | Landing (Next.js) | `${BE_BASE_URL}` |

---

## PHẦN 9: Đổi domain

Nếu mai sau cần đổi domain (ví dụ: `jobup.vn`):

1. **DNS**: Tạo A records cho domain mới trỏ về IP VPS
2. **`.env`**: Đổi `DOMAIN`, `FE_BASE_URL`, `BE_BASE_URL`
3. **Nginx conf.d/**: Đổi `server_name` trong cả 3 file, cập nhật `ssl_certificate` path
4. **CORS**: Cập nhật `ServiceExtensions.cs` → `WithOrigins(...)`, thêm domain mới
5. **Certbot**: Chạy lại để lấy cert cho domain mới:
   ```bash
   docker compose --profile certbot run --rm certbot
   ```
6. **Rebuild tất cả**:
   ```bash
   docker compose up -d --build
   ```

---

## PHẦN 10: Checklist Deploy

### Lần đầu deploy (copy & check)

- [ ] VPS: Docker + Docker Compose đã cài
- [ ] VPS: Firewall mở port 80, 443
- [ ] DNS: 4 A records trỏ về IP VPS (landing, www, admin, api)
- [ ] DNS: Đã propagate (`dig +short` trả về IP đúng)
- [ ] `.env`: Tạo từ `.env.example`, điền đầy đủ, password mạnh
- [ ] `nginx/ssl/` directory tồn tại
- [ ] Phase 1: `docker compose up -d --build` thành công
- [ ] Phase 1: Tất cả containers running (`docker compose ps`)
- [ ] Phase 1: HTTP truy cập được cả 3 domain
- [ ] Certbot: Lấy cert thành công, file cert tồn tại
- [ ] Phase 2: `cp nginx/conf.d-ssl/* nginx/conf.d/` — copy HTTPS configs
- [ ] Phase 2: `sed` đổi `.env` sang `https://`
- [ ] Phase 2: `docker compose up -d --build admin landing` — rebuild với URL HTTPS
- [ ] Phase 2: `docker compose exec nginx nginx -s reload` — áp dụng SSL config
- [ ] HTTPS: Cả 3 domain có khóa xanh trên trình duyệt
- [ ] HTTP → HTTPS: Redirect 301 hoạt động
- [ ] Admin: Login thành công, gọi API không lỗi CORS
- [ ] Crontab: Auto-renew cert đã setup
