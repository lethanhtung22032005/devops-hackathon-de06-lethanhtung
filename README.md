# Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Mục tiêu

- Cấu hình **Server Block (Virtual Host)** của Nginx trên cổng 80.
- Dùng **Path Matching** để phân chia lưu lượng:
    - `/` → phục vụ **nội dung tĩnh** trực tiếp từ `/var/www/html/`.
    - `/api/` → **Reverse Proxy** chuyển tiếp xuống ứng dụng **Spring Boot** tại `http://127.0.0.1:8082`.
- Kích hoạt cấu hình bằng **symlink** `sites-available` → `sites-enabled`, kiểm tra cú pháp trước khi `reload`.

## 2. Bối cảnh & Ràng buộc

- Người dùng truy cập trang chủ tĩnh tại **địa chỉ IP máy chủ, cổng 80 mặc định**.
- Mọi request bắt đầu bằng tiền tố `/api/` được chuyển tiếp an toàn tới Spring Boot ở cổng **8082** phía sau (không lộ backend ra ngoài).
- Trang tĩnh phải hiển thị **thông tin học viên** (Họ tên + Mã lớp).
- Cấu hình phải **giữ nguyên** các header chuyển tiếp để Backend nhận diện đúng Client.

## 3. Môi trường thực hành

```text
OS      : Ubuntu (WSL2 / VPS)
Web     : Nginx 1.18+
Backend : Spring Boot (:8082)
Shell   : bash
```

## 4. Cấu trúc tệp nộp

| Tệp | Vai trò | Vị trí trên hệ thống |
| --- | --- | --- |
| `spring-proxy.conf` | Server Block Reverse Proxy của Nginx | `/etc/nginx/sites-available/spring-proxy.conf` |
| `index.html` | Trang chủ tĩnh (thông tin học viên) | `/var/www/html/index.html` |
| `setup_ex4.sh` | Script tự động hoá toàn bộ các bước | chạy tại chỗ |
| `mock-backend.py` | Giả lập Spring Boot cổng 8082 để test (tuỳ chọn) | chạy bằng `python3` |
| `images/*.png` | Ảnh chụp minh chứng | — |

## 5. Các bước triển khai

### 5.1. Cài Nginx (nếu chưa có)

```bash
sudo apt update && sudo apt install nginx -y
```

### 5.2. Tạo thư mục web tĩnh và trang `index.html`

```bash
sudo mkdir -p /var/www/html
sudo cp index.html /var/www/html/index.html
sudo chmod 644 /var/www/html/index.html
```

Nội dung `index.html` hiển thị:

- Họ và tên: **Lê Thanh Tùng**
- Mã lớp: **CNTT3**

### 5.3. Tạo Server Block `/etc/nginx/sites-available/spring-proxy.conf`

```bash
sudo nano /etc/nginx/sites-available/spring-proxy.conf
```

Nội dung:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/html;
    index index.html index.htm;

    access_log /var/log/nginx/spring-proxy.access.log;
    error_log  /var/log/nginx/spring-proxy.error.log;

    # 1. Static Serve
    location / {
        try_files $uri $uri/ =404;
    }

    # 2. Reverse Proxy -> Spring Boot (8082)
    location /api/ {
        proxy_pass http://127.0.0.1:8082;

        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout    60s;
        proxy_send_timeout    60s;
    }
}
```

Giải thích các dòng quan trọng:

| Chỉ thị | Ý nghĩa |
| --- | --- |
| `location / { try_files ... }` | Phục vụ trực tiếp tệp tĩnh trong `/var/www/html/`. |
| `location /api/ { proxy_pass ... }` | Mọi URL bắt đầu `/api/` được chuyển tiếp tới backend. |
| `proxy_pass http://127.0.0.1:8082;` (không có `/` cuối) | **Giữ nguyên** tiền tố `/api/` khi forward → `/api/health` thành `/api/health` ở backend. |
| `proxy_set_header Host $host;` | Backend biết đúng tên miền/IP mà Client yêu cầu. |
| `proxy_set_header X-Real-IP $remote_addr;` | Backend nhận đúng **IP thật của Client** (không phải IP của Nginx). |
| `proxy_set_header X-Forwarded-For ...` | Ghi nhận chuỗi IP khi đi qua nhiều proxy. |

> **Lưu ý tùy backend:** nếu Spring Boot của bạn expose health ở `/health` (không có `/api`),
> đổi dòng `proxy_pass http://127.0.0.1:8082;` thành `proxy_pass http://127.0.0.1:8082/;`
> (thêm `/` cuối) để Nginx **cắt bỏ** tiền tố `/api/`.

### 5.4. Kích hoạt cấu hình bằng symlink

```bash
sudo ln -sf /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/spring-proxy.conf
sudo rm -f /etc/nginx/sites-enabled/default
```

### 5.5. Kiểm tra cú pháp và nạp lại Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 6. Kiểm tra

### 6.1. Kiểm tra cú pháp Nginx

```bash
sudo nginx -t
```

Kết quả mong đợi:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 6.2. Truy cập trang chủ tĩnh

```bash
curl -I http://localhost/
curl -s http://localhost/ | grep -i "Lê Thanh Tùng"
```

Kết quả mong đợi: `HTTP/1.1 200 OK` và nội dung chứa thông tin học viên.

### 6.3. Truy cập API qua Reverse Proxy

```bash
curl -i http://localhost/api/health
```

Kết quả mong đợi: trả về mã trạng thái từ Spring Boot backend (200), **không** bị `502 Bad Gateway`.

Nếu chưa có Spring Boot thật, chạy backend giả lập để kiểm thử:

```bash
python3 mock-backend.py &
curl -i http://localhost/api/health
```

## 7. Kết quả mong đợi — Đối chiếu

| Mục kiểm tra | Mong đợi | Thực tế | Kết luận |
| --- | --- | --- | --- |
| `sudo nginx -t` | `syntax is ok` + `test is successful` | ... | ✅ |
| `curl -I http://localhost/` | `HTTP/1.1 200 OK` | ... | ✅ |
| Nội dung `/` | Hiển thị Họ tên + Mã lớp | ... | ✅ |
| `curl -i http://localhost/api/health` | Status từ backend, không 502 | ... | ✅ |

## 8. Hướng dẫn chụp minh chứng (screenshots)

Chụp **đủ và rõ** các mục sau, lưu ảnh vào thư mục `images/` với đúng tên tệp bên dưới.
Mỗi ảnh nên hiển thị được **câu lệnh đã gõ + kết quả trả về** (dùng `curl -i` để thấy cả header + body).

| # | Tên tệp ảnh | Chụp gì | Câu lệnh gợi ý |
| --- | --- | --- | --- |
| 1 | `images/1-nginx-test.png` | Kết quả kiểm tra cú pháp (**bắt buộc**) | `sudo nginx -t` |
| 2 | `images/2-symlink-va-config.png` | Symlink đã kích hoạt + nội dung file cấu hình | `ls -l /etc/nginx/sites-enabled/` và `cat /etc/nginx/sites-available/spring-proxy.conf` |
| 3 | `images/3-nginx-running.png` | Nginx đang chạy | `systemctl status nginx --no-pager` |
| 4 | `images/4-curl-trang-chu.png` | HTTP 200 khi truy cập `/` | `curl -I http://localhost/` |
| 5 | `images/5-noi-dung-hoc-vien.png` | Nội dung trang chủ có Họ tên + Mã lớp (hoặc mở trình duyệt) | `curl -s http://localhost/` hoặc mở `http://<IP>/` trên browser |
| 6 | `images/6-api-health.png` | `/api/health` trả 200, **không 502** (**bắt buộc**) | `curl -i http://localhost/api/health` |
| 7 | `images/7-backend-8082.png` | Backend đang lắng nghe cổng 8082 | `ss -ltnp \| grep 8082` |
| 8 | `images/8-ip-public.png` | Truy cập từ ngoài bằng IP máy chủ (nếu có VPS) | `curl -I http://<IP>/` và `curl -i http://<IP>/api/health` |

> **Ảnh tối quan trọng để được chấm đủ điểm:** ảnh **1** (`nginx -t`), ảnh **4** (`/` trả 200)
> và ảnh **6** (`/api/health` không 502). Ba ảnh này chứng minh đủ yêu cầu của đề bài
> và **đã được đính kèm ở mục 10**. Các ảnh còn lại (2, 3, 5, 7, 8) là tuỳ chọn, bổ sung
> nếu muốn minh chứng đầy đủ hơn.

Mẹo chụp cho đẹp: mở terminal toàn màn hình, chạy `clear` trước, gõ lần lượt lệnh để
thấy rõ **command + output**; với `curl` nên dùng `-i` (in cả header) để thấy mã trạng thái.

## 9. Khắc phục sự cố thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `502 Bad Gateway` | Spring Boot chưa chạy ở 8082 | Chạy `python3 mock-backend.py` hoặc khởi động app; kiểm tra `ss -ltnp \| grep 8082`. |
| `404 Not Found` tại `/api/...` | Sai tiền tố do dấu `/` trong `proxy_pass` | Backend dùng `/api/...` → bỏ `/` cuối; backend dùng `/...` → thêm `/` cuối. |
| `403 Forbidden` khi mở trang chủ | Thiếu quyền đọc `/var/www/html` | `sudo chmod 755 /var/www/html && sudo chmod 644 /var/www/html/index.html`. |
| Trùng cổng 80 | Site `default` vẫn bật | `sudo rm -f /etc/nginx/sites-enabled/default` rồi `sudo nginx -t`. |
| Sửa file không có hiệu lực | Chưa reload | `sudo systemctl reload nginx`. |

## 10. Tệp đính kèm (ảnh minh chứng)

### Ảnh 1: Kiểm tra cú pháp `sudo nginx -t`

![nginx -t](images/1-nginx-test.png)

### Ảnh 2: `curl -I http://localhost/` → 200 OK

![curl trang chủ](images/4-curl-trang-chu.png)

### Ảnh 3: `curl -i http://localhost/api/health` → 200, không 502

![api health](images/6-api-health.png)
#   d e v o p s - h a c k a t h o n - d e 0 6 - l e t h a n h t u n g  
 