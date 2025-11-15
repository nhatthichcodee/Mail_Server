# HƯỚNG DẪN CẤU HÌNH MAIL SERVER CANHNHAT.TECH

**Trạng thái:** ✅ Đã kiểm chứng và hoạt động 100%  
**Ngày cập nhật:** 15/11/2025

---

## PHẦN 1: CHUẨN BỊ THÔNG TIN

### 1.1 Thông tin cần có
- **Domain:** canhnhat.tech
- **Nhà cung cấp DNS:** inet
- **VPS IP:** [IP_VPS_CUA_BAN] (ví dụ: 172.104.62.246)
- **Hostname:** mail.canhnhat.tech
- **Email test:** test@canhnhat.tech

---

## PHẦN 2: CẤU HÌNH DNS TẠI INET

### 2.1 Đăng nhập inet
1. Truy cập https://inet.vn
2. Đăng nhập với tài khoản
3. Tìm domain **canhnhat.tech** → **Quản lý DNS**

### 2.2 Thêm các bản ghi DNS sau

**Bản ghi A (mail server):**
```
Tên:     mail
Loại:    A
Giá trị: [IP_VPS_CUA_BAN]
TTL:     3600
```

**Bản ghi MX (Mail Exchange):**
```
Tên:     @ (để trống)
Loại:    MX
Giá trị: mail.canhnhat.tech
Ưu tiên: 10
TTL:     3600
```

**Bản ghi SPF:**
```
Tên:     @ (để trống)
Loại:    TXT
Giá trị: v=spf1 mx a ~all
TTL:     3600
```

**Bản ghi A cho domain chính:**
```
Tên:     @ (để trống)
Loại:    A
Giá trị: [IP_VPS_CUA_BAN]
TTL:     3600
```

### 2.3 Kiểm tra DNS
```bash
nslookup mail.canhnhat.tech
```

---

## PHẦN 3: CẤU HÌNH VPS

### 3.1 Kết nối SSH
```bash
ssh root@[IP_VPS_CUA_BAN]
```

### 3.2 Cập nhật hệ thống
```bash
apt update && apt upgrade -y
```

### 3.3 Cài đặt công cụ cần thiết
```bash
apt install python3 python3-pip git curl wget nano sqlite3 -y
```

### 3.4 Clone project
```bash
cd /home
git clone https://github.com/lettaidev/server_mail.git
cd server_mail
```

### 3.5 Cài đặt Python dependencies
```bash
pip3 install flask mail-parser gunicorn
```

### 3.6 Cài đặt SSL Certificate
```bash
apt install certbot python3-certbot-nginx -y

certbot certonly --standalone -d mail.canhnhat.tech -d canhnhat.tech
```

**Kết quả:**
```
Certificate: /etc/letsencrypt/live/mail.canhnhat.tech/fullchain.pem
Key:         /etc/letsencrypt/live/mail.canhnhat.tech/privkey.pem
```

---

## PHẦN 4: CẤU HÌNH FLASK API SERVICE

### 4.1 Tạo systemd service
```bash
cat > /etc/systemd/system/mail-api.service << 'EOF'
[Unit]
Description=Mail API Service
After=network.target

[Service]
User=root
WorkingDirectory=/home/server_mail
Environment="WEBHOOK_SECRET=LETTAI_SECRET6"
Environment="DATABASE_PATH=/home/server_mail/emails.db"
Environment="FLASK_DEBUG=False"
Environment="PORT=5000"
Environment="EMAIL_EXPIRY_HOURS=3"
ExecStart=/usr/bin/python3 -m gunicorn -w 4 -b 127.0.0.1:5000 only_api:create_app()
Restart=always
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

### 4.2 Khởi động service
```bash
systemctl daemon-reload
systemctl start mail-api
systemctl enable mail-api

# Kiểm tra
systemctl status mail-api
curl http://127.0.0.1:5000/api/health
```

**Output mong đợi:**
```json
{"status":"healthy","timestamp":1763195019,"version":"1.0.0"}
```

---

## PHẦN 5: CẤU HÌNH POSTFIX

### 5.1 Cài đặt Postfix
```bash
apt install postfix -y
```

Chọn: **Internet Site** → tên miền: **canhnhat.tech**

### 5.2 Tạo virtual aliases
```bash
cat > /etc/postfix/virtual << 'EOF'
@canhnhat.tech webhook
EOF

postmap /etc/postfix/virtual
```

### 5.3 Tạo transport mapping
```bash
cat > /etc/postfix/transport << 'EOF'
canhnhat.tech webhook:
EOF

postmap /etc/postfix/transport
```

### 5.4 Tạo webhook script
```bash
cat > /usr/local/bin/postfix-webhook.sh << 'EOF'
#!/bin/bash
/usr/bin/curl -X POST "http://127.0.0.1:5000/webhook" \
  -H "X-Secret: LETTAI_SECRET6" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @- 2>/dev/null
exit 0
EOF

chmod +x /usr/local/bin/postfix-webhook.sh
```

### 5.5 Cập nhật main.cf
```bash
cat >> /etc/postfix/main.cf << 'EOF'
virtual_alias_domains = canhnhat.tech
virtual_alias_maps = hash:/etc/postfix/virtual
transport_maps = hash:/etc/postfix/transport
EOF
```

### 5.6 Cập nhật master.cf
```bash
cat >> /etc/postfix/master.cf << 'EOF'
webhook  unix  -       n       n       -       -       pipe
  flags=Xhq user=mail argv=/usr/local/bin/postfix-webhook.sh
EOF
```

### 5.7 Khởi động Postfix
```bash
postfix check
postfix reload
postfix start
systemctl status postfix
```

---

## PHẦN 6: CẤU HÌNH OPENDK IM

### 6.1 Cài đặt OpenDKIM
```bash
apt install opendkim opendkim-tools -y
```

### 6.2 Tạo DKIM keys
```bash
mkdir -p /etc/opendkim/keys/canhnhat.tech
cd /etc/opendkim/keys/canhnhat.tech

opendkim-genkey -b 2048 -d canhnhat.tech -D /etc/opendkim/keys/canhnhat.tech -s default -v
```

### 6.3 Xem public key
```bash
cat /etc/opendkim/keys/canhnhat.tech/default.txt
```

**Lấy giá trị từ `v=DKIM1` đến hết và thêm vào DNS inet:**

```
Tên:     default._domainkey
Loại:    TXT
Giá trị: v=DKIM1; h=sha256; k=rsa; p=[FULL_KEY_VALUE]
TTL:     3600
```

### 6.4 Đặt quyền
```bash
chown opendkim:opendkim /etc/opendkim/keys/canhnhat.tech/*
chmod 400 /etc/opendkim/keys/canhnhat.tech/default.private
```

### 6.5 Cấu hình OpenDKIM
```bash
cat > /etc/opendkim.conf << 'EOF'
Domain                  canhnhat.tech
KeyFile                 /etc/opendkim/keys/canhnhat.tech/default.private
Selector                default
Socket                  inet:8891@localhost
EOF
```

### 6.6 Cập nhật Postfix main.cf
```bash
cat >> /etc/postfix/main.cf << 'EOF'

# DKIM Configuration
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
EOF
```

### 6.7 Khởi động OpenDKIM
```bash
systemctl start opendkim
systemctl enable opendkim
systemctl restart postfix
```

---

## PHẦN 7: CẤU HÌNH NGINX

### 7.1 Cài đặt Nginx
```bash
apt install nginx -y
```

### 7.2 Tạo cấu hình Nginx
```bash
cat > /etc/nginx/sites-available/mail.canhnhat.tech << 'EOF'
server {
    listen 80;
    server_name mail.canhnhat.tech canhnhat.tech;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name mail.canhnhat.tech canhnhat.tech;
    
    ssl_certificate /etc/letsencrypt/live/mail.canhnhat.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mail.canhnhat.tech/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    client_max_body_size 20M;
    
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
EOF
```

### 7.3 Kích hoạt cấu hình
```bash
ln -s /etc/nginx/sites-available/mail.canhnhat.tech /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl restart nginx
systemctl enable nginx
```

---

## PHẦN 8: CẤU HÌNH FIREWALL

### 8.1 Cài đặt và bật UFW
```bash
apt install ufw -y

ufw allow 22/tcp
ufw allow 25/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 587/tcp
ufw allow 110/tcp
ufw allow 143/tcp
ufw allow 993/tcp
ufw allow 995/tcp

ufw enable
ufw status
```

---

## PHẦN 9: KIỂM TRA HỆ THỐNG

### 9.1 Kiểm tra services
```bash
systemctl status postfix opendkim mail-api nginx
postfix check
```

### 9.2 Kiểm tra DNS
```bash
dig mail.canhnhat.tech A
dig canhnhat.tech MX
dig canhnhat.tech TXT
dig default._domainkey.canhnhat.tech TXT
```

### 9.3 Test API
```bash
curl -s https://mail.canhnhat.tech/api/health | python3 -m json.tool
curl -s https://mail.canhnhat.tech/api/email/test@canhnhat.tech | python3 -m json.tool
```

### 9.4 Kiểm tra database
```bash
sqlite3 /home/server_mail/emails.db "SELECT COUNT(*) FROM emails;"
```

---

## PHẦN 10: TEST NHẬN EMAIL

### 10.1 Gửi email test
1. Mở Gmail
2. Soạn email mới
3. Gửi tới: **test@canhnhat.tech**

### 10.2 Kiểm tra trên VPS
```bash
# Xem logs
tail -50 /var/log/syslog | grep postfix

# Kiểm tra email
curl -s https://mail.canhnhat.tech/api/email/test@canhnhat.tech | python3 -m json.tool

# Đếm email trong database
sqlite3 /home/server_mail/emails.db "SELECT COUNT(*) FROM emails;"
```

---

## PHẦN 11: CẤU HÌNH AUTO-RENEWAL SSL

```bash
certbot renew --dry-run

systemctl enable certbot.timer
systemctl start certbot.timer
systemctl status certbot.timer
```

---

## PHẦN 12: API ENDPOINTS

### Health Check
```bash
GET https://mail.canhnhat.tech/api/health
```

### Lấy danh sách email
```bash
GET https://mail.canhnhat.tech/api/email/test@canhnhat.tech
```

### Lấy email theo ID
```bash
GET https://mail.canhnhat.tech/api/inbox/{email_id}
```

### Xóa email theo ID
```bash
DELETE https://mail.canhnhat.tech/api/inbox/{email_id}
```

### Xóa tất cả email
```bash
DELETE https://mail.canhnhat.tech/api/email/test@canhnhat.tech
```

---

## PHẦN 13: TROUBLESHOOTING - FIX LỖI CHI TIẾT

### ❌ LỖI 1: EMAIL KHÔNG ĐƯỢC NHẬN

**Triệu chứng:** Email gửi đến test@canhnhat.tech nhưng không xuất hiện trong database

#### 🔍 Bước 1: Kiểm tra DNS
```bash
nslookup -type=MX canhnhat.tech
```

**Kết quả mong đợi:**
```
Server:  8.8.8.8
Address: 8.8.8.8#53

canhnhat.tech   mail exchanger = 10 mail.canhnhat.tech.
```

**Nếu không có MX record:**
1. Đăng nhập inet.vn
2. Quản lý DNS domain canhnhat.tech
3. Thêm bản ghi MX: `@ → mail.canhnhat.tech (priority 10)`
4. Đợi 5-10 phút để DNS cập nhật

#### 🔍 Bước 2: Kiểm tra DNS A record
```bash
nslookup mail.canhnhat.tech
```

**Kết quả mong đợi:**
```
Server:  8.8.8.8
Address: 8.8.8.8#53

mail.canhnhat.tech      canonical name = mail.canhnhat.tech.
Address: 172.104.62.246
```

**Nếu không resolve:**
1. Kiểm tra inet.vn - bản ghi A `mail → [IP_VPS]`
2. Xác nhận IP VPS chính xác
3. Đợi DNS propagation 5-10 phút

#### 🔍 Bước 3: Kiểm tra Postfix status
```bash
systemctl status postfix
```

**Nếu status "exited (not running)":**
```bash
postfix start
systemctl enable postfix
systemctl restart postfix
```

#### 🔍 Bước 4: Kiểm tra Postfix configuration
```bash
postfix check
```

**Nếu có lỗi cú pháp:**
```bash
# Xem chi tiết lỗi
postfix check
# Sửa file main.cf tương ứng
nano /etc/postfix/main.cf
# Sau sửa, reload
postfix reload
```

#### 🔍 Bước 5: Kiểm tra virtual aliases
```bash
postmap -q @canhnhat.tech /etc/postfix/virtual
```

**Kết quả mong đợi:** `webhook`

**Nếu không có kết quả:**
```bash
# Kiểm tra file virtual
cat /etc/postfix/virtual

# Nếu thiếu, thêm:
cat > /etc/postfix/virtual << 'EOF'
@canhnhat.tech webhook
EOF

# Postmap lại
postmap /etc/postfix/virtual
postfix reload
```

#### 🔍 Bước 6: Kiểm tra transport mapping
```bash
postmap -q canhnhat.tech /etc/postfix/transport
```

**Kết quả mong đợi:** `webhook:`

**Nếu không có kết quả:**
```bash
# Kiểm tra file transport
cat /etc/postfix/transport

# Nếu thiếu, thêm:
cat > /etc/postfix/transport << 'EOF'
canhnhat.tech webhook:
EOF

# Postmap lại
postmap /etc/postfix/transport
postfix reload
```

#### 🔍 Bước 7: Kiểm tra webhook script
```bash
ls -la /usr/local/bin/postfix-webhook.sh
cat /usr/local/bin/postfix-webhook.sh
```

**Nếu không tồn tại hoặc thiếu quyền:**
```bash
cat > /usr/local/bin/postfix-webhook.sh << 'EOF'
#!/bin/bash
/usr/bin/curl -X POST "http://127.0.0.1:5000/webhook" \
  -H "X-Secret: LETTAI_SECRET6" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @- 2>/dev/null
exit 0
EOF

chmod +x /usr/local/bin/postfix-webhook.sh
```

#### 🔍 Bước 8: Xem Postfix logs
```bash
tail -100 /var/log/syslog | grep postfix
```

**Các lỗi thường gặp:**

**Lỗi:** `unknown virtual alias table type: hash:/etc/postfix/virtual`
- **Fix:** Kiểm tra `/etc/postfix/main.cf` có dòng `virtual_alias_maps = hash:/etc/postfix/virtual`
- Nếu thiếu, thêm vào main.cf rồi `postfix reload`

**Lỗi:** `connect to 127.0.0.1:5000: Connection refused`
- **Fix:** Flask API không chạy, xem lỗi API (phần dưới)

**Lỗi:** `permission denied: /usr/local/bin/postfix-webhook.sh`
- **Fix:** `chmod +x /usr/local/bin/postfix-webhook.sh`

#### 🔍 Bước 9: Test webhook manual
```bash
# Tạo test email
cat > /tmp/test.eml << 'EOF'
From: test@gmail.com
To: test@canhnhat.tech
Subject: Test Email
Date: Mon, 15 Nov 2025 10:00:00 +0700

Test content
EOF

# Gửi đến API
curl -X POST "http://127.0.0.1:5000/webhook" \
  -H "X-Secret: LETTAI_SECRET6" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @/tmp/test.eml
```

**Nếu có lỗi:** Kiểm tra Flask API (phần dưới)

---

### ❌ LỖI 2: API KHÔNG HOẠT ĐỘNG

**Triệu chứng:** Không thể kết nối tới API hoặc API trả về lỗi

#### 🔍 Bước 1: Kiểm tra service status
```bash
systemctl status mail-api
```

**Nếu "inactive (dead)":**
```bash
systemctl start mail-api
systemctl enable mail-api
systemctl status mail-api
```

#### 🔍 Bước 2: Kiểm tra logs
```bash
journalctl -u mail-api -n 100
```

**Lỗi:** `ModuleNotFoundError: No module named 'flask'`
- **Fix:** Cài đặt dependencies
```bash
pip3 install flask mail-parser gunicorn
```

**Lỗi:** `Address already in use`
- **Fix:** Port 5000 bị sử dụng
```bash
lsof -i :5000
kill -9 <PID>
systemctl restart mail-api
```

**Lỗi:** `FileNotFoundError: /home/server_mail/emails.db`
- **Fix:** Database chưa tạo
```bash
cd /home/server_mail
python3 -c "from only_api import create_app; app = create_app(); print('DB created')"
```

#### 🔍 Bước 3: Test API trực tiếp
```bash
curl -v http://127.0.0.1:5000/api/health
```

**Kết quả mong đợi:** HTTP 200 với JSON `{"status":"healthy",...}`

**Nếu Connection refused:**
```bash
netstat -tlnp | grep 5000
```

**Nếu không thấy 5000:**
- Khởi động lại service
```bash
systemctl restart mail-api
```

#### 🔍 Bước 4: Kiểm tra file only_api.py
```bash
cd /home/server_mail
python3 -m py_compile only_api.py
```

**Nếu có lỗi syntax:**
```bash
python3 only_api.py
```
- Xem lỗi và sửa file

#### 🔍 Bước 5: Kiểm tra quyền file
```bash
ls -la /home/server_mail/only_api.py
ls -la /home/server_mail/emails.db
```

**Nếu không có read permission:**
```bash
chmod 644 /home/server_mail/only_api.py
chmod 666 /home/server_mail/emails.db
```

#### 🔍 Bước 6: Kiểm tra service file
```bash
cat /etc/systemd/system/mail-api.service
```

**Kiểm tra các dòng quan trọng:**
```
WorkingDirectory=/home/server_mail     # Đúng
ExecStart=/usr/bin/python3 -m gunicorn -w 4 -b 127.0.0.1:5000 only_api:create_app()
```

**Nếu sai, sửa:**
```bash
nano /etc/systemd/system/mail-api.service
systemctl daemon-reload
systemctl restart mail-api
```

#### 🔍 Bước 7: Test database
```bash
sqlite3 /home/server_mail/emails.db "SELECT COUNT(*) FROM emails;"
```

**Nếu lỗi "no such table":**
```bash
# Xóa DB cũ
rm /home/server_mail/emails.db
# Khởi động lại API để tạo DB mới
systemctl restart mail-api
```

---

### ❌ LỖI 3: NGINX KHÔNG HOẠT ĐỘNG / HTTPS LỖI

**Triệu chứng:** 502 Bad Gateway, SSL error, hoặc không thể truy cập https

#### 🔍 Bước 1: Kiểm tra Nginx syntax
```bash
nginx -t
```

**Nếu có lỗi:**
```bash
# Xem chi tiết
nginx -T

# Sửa file cấu hình
nano /etc/nginx/sites-available/mail.canhnhat.tech

# Reload
systemctl reload nginx
```

#### 🔍 Bước 2: Kiểm tra Nginx status
```bash
systemctl status nginx
```

**Nếu "inactive":**
```bash
systemctl start nginx
systemctl enable nginx
```

#### 🔍 Bước 3: Kiểm tra SSL certificate
```bash
ls -la /etc/letsencrypt/live/mail.canhnhat.tech/
```

**Nếu không tồn tại:**
```bash
certbot certonly --standalone -d mail.canhnhat.tech -d canhnhat.tech
```

#### 🔍 Bước 4: Kiểm tra certificate hợp lệ
```bash
openssl x509 -in /etc/letsencrypt/live/mail.canhnhat.tech/fullchain.pem -text -noout
```

**Kiểm tra dòng:**
- `Issuer: C = US, O = Let's Encrypt` - OK
- `Not Before` và `Not After` - Xác nhận còn hạn

**Nếu hết hạn:**
```bash
certbot renew --force-renewal
systemctl reload nginx
```

#### 🔍 Bước 5: Kiểm tra Nginx logs
```bash
tail -50 /var/log/nginx/error.log
tail -50 /var/log/nginx/access.log
```

**Lỗi:** `502 Bad Gateway`
- **Fix:** Flask API không chạy, kiểm tra phần "API không hoạt động"

**Lỗi:** `upstream timed out`
- **Fix:** Tăng timeout trong Nginx config
```bash
nano /etc/nginx/sites-available/mail.canhnhat.tech
# Tăng các dòng proxy_*_timeout
systemctl reload nginx
```

#### 🔍 Bước 6: Test HTTPS từ client
```bash
curl -v https://mail.canhnhat.tech/api/health
```

**Nếu SSL certificate error:**
```bash
# Kiểm tra certificate chain
openssl s_client -connect mail.canhnhat.tech:443 -showcerts
```

#### 🔍 Bước 7: Kiểm tra firewall
```bash
ufw status
```

**Nếu 443 chưa mở:**
```bash
ufw allow 443/tcp
ufw reload
```

---

### ❌ LỖI 4: SPF / DKIM KHÔNG HOẠT ĐỘNG

**Triệu chứng:** Email bị spam, DKIM/SPF validation fails

#### 🔍 Bước 1: Kiểm tra SPF record
```bash
dig canhnhat.tech TXT +short | grep spf
```

**Kết quả mong đợi:** `v=spf1 mx a ~all`

**Nếu không có:**
1. inet.vn → Quản lý DNS
2. Thêm TXT record: `@ → v=spf1 mx a ~all`
3. Đợi DNS propagate

#### 🔍 Bước 2: Kiểm tra DKIM record
```bash
dig default._domainkey.canhnhat.tech TXT +short
```

**Kết quả mong đợi:** `v=DKIM1; h=sha256; k=rsa; p=...`

**Nếu không có:**
```bash
# Xem public key
cat /etc/opendkim/keys/canhnhat.tech/default.txt

# Copy toàn bộ từ v=DKIM1... đến hết
# Thêm vào inet.vn: TXT record `default._domainkey → [copied value]`
```

#### 🔍 Bước 3: Kiểm tra OpenDKIM service
```bash
systemctl status opendkim
```

**Nếu "inactive":**
```bash
systemctl start opendkim
systemctl enable opendkim
systemctl restart postfix
```

#### 🔍 Bước 4: Kiểm tra OpenDKIM logs
```bash
tail -50 /var/log/syslog | grep opendkim
```

**Lỗi:** `bind(): Address already in use`
- **Fix:** Port 8891 bị sử dụng
```bash
lsof -i :8891
kill -9 <PID>
systemctl restart opendkim
```

**Lỗi:** `Unable to open key file`
- **Fix:** Quyền file sai
```bash
chown opendkim:opendkim /etc/opendkim/keys/canhnhat.tech/*
chmod 400 /etc/opendkim/keys/canhnhat.tech/default.private
chmod 444 /etc/opendkim/keys/canhnhat.tech/default.txt
```

#### 🔍 Bước 5: Test DKIM
Gửi email từ Gmail tới test@canhnhat.tech:
1. Mở email nhận được
2. Click "..." → "Show original"
3. Tìm dòng: `dkim=pass` hoặc `dkim=fail`

**Nếu dkim=fail:**
- Kiểm tra lại DNS DKIM record
- Xác nhận key value đúng

#### 🔍 Bước 6: Kiểm tra Postfix milter config
```bash
grep milter /etc/postfix/main.cf
```

**Kết quả mong đợi:**
```
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

**Nếu thiếu:**
```bash
cat >> /etc/postfix/main.cf << 'EOF'
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
EOF

postfix reload
```

---

### ❌ LỖI 5: FIREWALL CHẶN EMAIL/API

**Triệu chứng:** Không thể gửi email, API không thể truy cập từ ngoài

#### 🔍 Bước 1: Kiểm tra firewall status
```bash
ufw status
```

#### 🔍 Bước 2: Mở ports cần thiết
```bash
# SMTP
ufw allow 25/tcp

# HTTP/HTTPS
ufw allow 80/tcp
ufw allow 443/tcp

# Optional (POP3, IMAP)
ufw allow 110/tcp
ufw allow 143/tcp
ufw allow 993/tcp
ufw allow 995/tcp

# Reload
ufw reload
```

#### 🔍 Bước 3: Kiểm tra port mở
```bash
netstat -tlnp
```

**Các port cần thấy:**
- `:25` → postfix
- `:5000` → flask (127.0.0.1)
- `:80` → nginx
- `:443` → nginx
- `:8891` → opendkim (127.0.0.1)

**Nếu thiếu, restart service tương ứng**

---

### ❌ LỖI 6: DATABASE LỖI / EMAIL KHÔNG LƯU

**Triệu chứng:** Email nhận được nhưng không xuất hiện trong database

#### 🔍 Bước 1: Kiểm tra database tồn tại
```bash
ls -la /home/server_mail/emails.db
```

**Nếu không tồn tại:**
```bash
# Khởi động Flask để tạo DB
systemctl restart mail-api
sleep 2
# Hoặc tạo manual:
cd /home/server_mail
python3 << 'EOF'
from only_api import create_app
app = create_app()
print("Database created at /home/server_mail/emails.db")
EOF
```

#### 🔍 Bước 2: Kiểm tra database integrity
```bash
sqlite3 /home/server_mail/emails.db ".tables"
sqlite3 /home/server_mail/emails.db ".schema emails"
```

**Nếu lỗi:**
```bash
# Backup cũ
cp /home/server_mail/emails.db /home/server_mail/emails.db.bak

# Xóa và tạo mới
rm /home/server_mail/emails.db
systemctl restart mail-api
```

#### 🔍 Bước 3: Kiểm tra dữ liệu
```bash
sqlite3 /home/server_mail/emails.db "SELECT COUNT(*) FROM emails;"
sqlite3 /home/server_mail/emails.db "SELECT * FROM emails LIMIT 5;"
```

#### 🔍 Bước 4: Kiểm tra quyền file
```bash
ls -la /home/server_mail/emails.db
```

**Nếu quyền sai:**
```bash
chmod 666 /home/server_mail/emails.db
```

#### 🔍 Bước 5: Test webhook với curl
```bash
curl -X POST "http://127.0.0.1:5000/webhook" \
  -H "X-Secret: LETTAI_SECRET6" \
  -H "Content-Type: application/octet-stream" \
  -d "From: test@gmail.com\nTo: test@canhnhat.tech\nSubject: Test\n\nTest body"
```

**Nếu có lỗi:**
```bash
journalctl -u mail-api -n 20
```

---

### ⚡ QUICK FIX CHECKLIST

```bash
# Tất cả services chạy?
systemctl status postfix opendkim mail-api nginx

# Tất cả ports mở?
ufw status | grep ALLOW

# DNS ok?
nslookup -type=MX canhnhat.tech
nslookup mail.canhnhat.tech

# API respond?
curl -s http://127.0.0.1:5000/api/health

# Database ok?
sqlite3 /home/server_mail/emails.db "SELECT COUNT(*) FROM emails;"

# Logs?
tail -20 /var/log/syslog | grep -i "postfix\|opendkim\|nginx"
journalctl -u mail-api -n 20
```

**Nếu vẫn lỗi:**
```bash
# Restart tất cả
systemctl restart postfix opendkim mail-api nginx

# Reload configs
postfix reload
nginx -s reload
systemctl reload opendkim

# Xem logs chi tiết
journalctl -xeu mail-api
tail -100 /var/log/syslog
```

---

## PHẦN 14: BIẾN MÔI TRƯỜNG (Tùy chỉnh)

Trong `/etc/systemd/system/mail-api.service`:

```
WEBHOOK_SECRET=LETTAI_SECRET6         # Secret key
DATABASE_PATH=/home/server_mail/emails.db  # DB path
FLASK_DEBUG=False                     # Debug mode
PORT=5000                            # API port
EMAIL_EXPIRY_HOURS=3                 # Lưu email (giờ)
```

---

## CHECKLIST HOÀN THÀNH

✅ **DNS Configuration:**
- [ ] A record: mail → IP_VPS
- [ ] A record: @ → IP_VPS
- [ ] MX record: @ → mail.canhnhat.tech
- [ ] SPF record: v=spf1 mx a ~all
- [ ] DKIM record: default._domainkey

✅ **VPS Services:**
- [ ] Postfix chạy
- [ ] OpenDKIM chạy
- [ ] Mail API chạy
- [ ] Nginx chạy

✅ **Security:**
- [ ] SSL certificate cài đặt
- [ ] HTTPS hoạt động
- [ ] Firewall mở ports

✅ **Email Functionality:**
- [ ] API health check OK
- [ ] Nhận email từ Gmail
- [ ] Email lưu database
- [ ] Lấy email qua API

---

**✅ Setup hoàn thành! Mail server sẵn sàng hoạt động.**
