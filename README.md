# CoffeeShop Fullstack App — React + .NET + PostgreSQL (Docker Deploy)

##  1. Giới thiệu

Dự án **CoffeeShop** là một ứng dụng web fullstack gồm:
- **Frontend:** ReactJS (đã build sẵn, chạy qua Nginx)
- **Backend:** ASP.NET Core Web API
- **Database:** PostgreSQL 16
- **Triển khai:** Docker Compose trên Ubuntu Server

**Mục tiêu**: triển khai ứng dụng **React + .NET** đầy đủ, **kết nối PostgreSQL**,
reverse proxy bằng Nginx, hoạt động qua domain nội bộ: **http://www.devopp.edu.vn/**

---

##  2. Kiến trúc hệ thống

┌───────────────────────────────────────────────────────┐
│                    Ubuntu Server                      │
│                                                       │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────┐  │
│  │ Nginx       │→→ │ ASP.NET Core │→→ │ PostgreSQL │  │
│  │ (Frontend)  │   │ (Backend)    │   │ (Database) │  │
│  │ Port 80     │   │ Port 8080    │   │ Port 5432  │  │
│  └─────────────┘   └──────────────┘   └────────────┘  │
│          │                  │                 │       │
│   /usr/share/nginx/html  ./publish        ./pg_data   │
└───────────────────────────────────────────────────────┘

---

##  3. Thành phần chính trong thư mục `/home/patz/webserver`

_________________________________________________________________________________________________
|             Thành phần | Mô tả                                                                |
|------------------------|----------------------------------------------------------------------|
| `docker-compose.yml`   | Cấu hình Docker Compose khởi tạo 3 services (db, backend, frontend). |
| `nginx.conf`           | Reverse proxy chuyển hướng /api → backend.                           |
| `publish/`             | Thư mục chứa file build .NET API.                                    |
| `build/`               | Thư mục chứa file build React (frontend).                            |
| `CoffeeShopBk1125.sql` | File backup database dạng custom dump.                               |
| `runtime-config.js`    | Cấu hình runtime cho frontend (API endpoint).                        |
-------------------------------------------------------------------------------------------------
---

##  4. Chuẩn bị môi trường

SSH vào server và chuyển đến thư mục dự án (ubuntu server):

- Cài Docker và Docker Compose trên Ubuntu:
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli
sudo systemctl enable --now docker
docker --version

sudo apt install docker-compose -y
docker compose version
```

---

##  5. File cấu hình

###  docker-compose.yml
Sử dụng PostgreSQL 16 + backend ASP.NET Core + frontend Nginx.

```yaml
version: '3.9'

services:
  db:
    image: postgres:16
    container_name: coffee_db
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: P@ssword@2024
      POSTGRES_DB: CoffeeShop
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./CoffeeShopBk1125.sql:/docker-entrypoint-initdb.d/CoffeeShopBk1125.sql
    networks:
      - coffee_net

  backend:
    build:
      context: ./publish
      dockerfile: Dockerfile
    container_name: coffee_backend
    restart: always
    ports:
      - "5094:8080"
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__ConnectionDb=Host=db;Port=5432;Database=CoffeeShop;Username=postgres;Password=P@ssword@2024
    depends_on:
      - db
    networks:
      - coffee_net

  frontend:
    image: nginx:alpine
    container_name: coffee_frontend
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./build:/usr/share/nginx/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - backend
    networks:
      - coffee_net

volumes:
  pg_data:

networks:
  coffee_net:
    driver: bridge


```

---

###  nginx.conf
Reverse proxy chuyển hướng các yêu cầu `/api/...` đến backend.
```bash
server {
    listen 80;
    server_name www.devopp.edu.vn 192.168.64.3;

    root /usr/share/nginx/html;  # hoặc thư mục build FE
    index index.html index.htm;

    # Proxy API tới backend
    location /api/ {
        proxy_pass http://coffee_backend:8080/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        #  Bổ sung fix CORS ở đây
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin' always;

        if ($request_method = OPTIONS) {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Length' 0;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            return 204;
        }
    }

    # Phần còn lại phục vụ frontend React
    location / {
        try_files $uri /index.html;
    }
}

```
## 6. Cập nhật cấu hình runtime & backend
- Domain frontend mà backend chấp nhận là http://www.devopp.edu.vn/
## Sửa lại Build/runtime-config.js:
```bash
window.__RUNTIME_CONFIG__ = {
  API_BASE_URL: "http://www.devopp.edu.vn/api/"
};
```

---
- Sửa lại để backend gọi đúng service PostgreSQL trong Docker:
## Sửa lại publish/appsettings.json:

```bash
"ConnectionStrings": {
    "ConnectionDb": "Host=db;Port=5432;Database=CoffeeShop;Username=postgres;Password=P@ssword@2024"
   },
```
##  7. Khởi chạy ứng dụng

Dọn sạch container cũ (nếu có):
```bash
sudo docker compose down -v
```

Build & khởi chạy:
```bash
sudo docker compose up -d --build
```

Kiểm tra:
```bash
sudo docker ps
```

---

##  8. Phục hồi database

Copy file dump vào container:
```bash
sudo docker cp CoffeeShopBk1125.sql coffee_db:/CoffeeShopBk1125.sql
sudo docker exec -it coffee_db bash
pg_restore -U postgres -d CoffeeShop /CoffeeShopBk1125.sql
```

Kiểm tra bảng:
```bash
psql -U postgres -d CoffeeShop
\dt
```

---

##  9. Kiểm tra hoạt động

### Backend (Swagger)
```bash
curl http://localhost:5094/api/staff
curl http://localhost:5094/api/Branch
```
Hoặc truy cập:

> http://<IP_ubuntu>:5094/api/staff
> http://<IP_ubuntu>:5094/api/Branch

### Frontend (React)
Mở trình duyệt:

> http://<IP_ubuntu>/

---

##  10. Cấu hình domain & CORS

Nếu dùng domain nội bộ:
```
<IP_Ubuntu>  www.devopp.edu.vn
```
Backend đã cho phép origin `http://www.devopp.edu.vn`.

---


##  11 Kết luận

Dự án **CoffeeShop Fullstack Deploy Lab** giúp sinh viên và lập trình viên:
- Hiểu rõ pipeline triển khai fullstack thực tế.
- Làm chủ Docker Compose, PostgreSQL, và reverse proxy Nginx.
- Áp dụng mô hình triển khai production-like trên Ubuntu Server.

---

 **Tác giả:** Nguyễn Phát  
 **Cập nhật:** 2025-11  
 **Môi trường chạy thử:** Ubuntu Server 22.04, Docker 27+, Docker Compose v2
