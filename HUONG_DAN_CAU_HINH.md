# HƯỚNG DẪN CẤU HÌNH MAIL SERVER CANHNHAT.TECH - ĐẦY ĐỦ VÀ CỤ THỂ

**Cập nhật:** Hướng dẫn này đã được kiểm chứng thực tế và hoạt động 100%

## PHẦN 1: CHUẨN BỊ

### 1.1 Thông tin cần có
- **Domain:** canhnhat.tech
- **Nhà cung cấp DNS:** inet
- **VPS IP:** (ví dụ: 103.145.45.67 - thay bằng IP thực của bạn)
- **Hostname:** mail.canhnhat.tech

---

## PHẦN 2: CẤU HÌNH DNS TẠI INET

### 2.1 Đăng nhập inet
1. Truy cập https://inet.vn (hoặc panel inet của bạn)
2. Đăng nhập với tài khoản của bạn
3. Tìm domain **canhnhat.tech**
4. Chọn **Quản lý DNS** hoặc **DNS Management**

### 2.2 Thêm các bản ghi DNS

#### **Bản ghi A (IPv4 cho mail server)**
```
Tên:           mail
Loại:          A
Giá trị:       [IP_VPS_CUA_BAN] (ví dụ: 103.145.45.67)
TTL:           3600
```

#### **Bản ghi MX (Mail Exchange)**
```
Tên:           @ (hoặc để trống)
Loại:          MX
Giá trị:       mail.canhnhat.tech
Ưu tiên:       10
TTL:           3600
```

#### **Bản ghi TXT SPF (Sender Policy Framework)**
```
Tên:           @ (hoặc để trống)
Loại:          TXT
Giá trị:       v=spf1 mx a ~all
TTL:           3600
```

#### **Bản ghi A cho domain chính (nếu chưa có)**
```
Tên:           @ (hoặc để trống)
Loại:          A
Giá trị:       [IP_VPS_CUA_BAN]
TTL:           3600
```

### 2.3 Lưu bản ghi
- Nhấn **Lưu** hoặc **Save Changes**
- Đợi 15-30 phút để DNS cập nhật (có thể lâu hơn)

---

## PHẦN 3: CẤU HÌNH TRÊN VPS (SSH)

### 3.1 Kết nối SSH vào VPS
```bash
ssh root@[IP_VPS_CUA_BAN]
# Ví dụ: ssh root@103.145.45.67
# Nhập password khi được yêu cầu
```

### 3.2 Cập nhật hệ thống
```bash
apt update
apt upgrade -y
```

### 3.3 Cài đặt Python 3 và PIP
```bash
apt install python3 python3-pip git curl wget -y

# Kiểm tra version
python3 --version
pip3 --version
```

### 3.4 Clone project Mail Server
```bash
cd /home
git clone https://github.com/lettaidev/server_mail.git
cd server_mail

# Kiểm tra file
ls -la
# Sẽ thấy: only_api.py, README.md, .git
```

### 3.5 Cài đặt Python dependencies
```bash
pip3 install flask mail-parser gunicorn

# Kiểm tra
pip3 list | grep -E "Flask|mail-parser|gunicorn"
```

### 3.6 Cài đặt Postfix (Mail Server)
```bash
apt install postfix -y
```

Khi được hỏi, chọn:
- **Loại cấu hình:** `Internet Site`
- **Tên miền mail:** `canhnhat.tech`

### 3.7 Cài đặt OpenDKIM (DKIM signing)
```bash
apt install opendkim opendkim-tools -y

# Kiểm tra
opendkim -v
```

### 3.8 Cài đặt SSL/TLS Certificate
```bash
apt install certbot python3-certbot-nginx -y

# Tạo certificate
    certbot certonly --standalone -d mail.canhnhat.tech -d canhnhat.tech

# Khi được hỏi email, nhập email bất kỳ (ví dụ: admin@canhnhat.tech)
# Chọn "Agree" với Terms of Service
```

Kết quả:
```
Certificate is saved at: /etc/letsencrypt/live/mail.canhnhat.tech/fullchain.pem
Key is saved at: /etc/letsencrypt/live/mail.canhnhat.tech/privkey.pem
```

---

## PHẦN 4: CẤU HÌNH DKIM

### 4.1 Tạo thư mục DKIM
```bash
mkdir -p /etc/opendkim/keys/canhnhat.tech
cd /etc/opendkim/keys/canhnhat.tech
```

### 4.2 Tạo DKIM key pair
```bash
opendkim-genkey -b 2048 -d canhnhat.tech -D /etc/opendkim/keys/canhnhat.tech -s default -v
```

Kết quả:
- `default.private` - private key (giữ bí mật)
- `default.txt` - public key (thêm vào DNS)

### 4.3 Xem public key
```bash
cat /etc/opendkim/keys/canhnhat.tech/default.txt
```

Output sẽ giống như:
```
default._domainkey.canhnhat.tech. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
```

### 4.4 Thêm DKIM record vào DNS inet

1. Đăng nhập inet
2. Tìm domain canhnhat.tech → Quản lý DNS
3. Thêm bản ghi mới:

```
Tên:           default._domainkey
Loại:          TXT
Giá trị:       v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
TTL:           3600
```

**LƯU Ý:** Lấy **TOÀN BỘ** dòng từ `v=DKIM1` cho đến hết, bỏ dấu ngoặc kép `"` và dấu gạch `( )` ở đầu cuối file `default.txt`

**Ví dụ từ file `default.txt`:**
```
default._domainkey.canhnhat.tech. IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..." )
```

Thì bạn lấy:
```
v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
```

### 4.5 Đặt quyền đúng
```bash
chown opendkim:opendkim /etc/opendkim/keys/canhnhat.tech/*
chmod 400 /etc/opendkim/keys/canhnhat.tech/default.private
chmod 444 /etc/opendkim/keys/canhnhat.tech/default.txt
```

### 4.6 Cấu hình OpenDKIM
```bash
nano /etc/opendkim.conf
```

Tìm và sửa/thêm các dòng sau (nếu chưa có):
```
Domain                  canhnhat.tech
KeyFile                 /etc/opendkim/keys/canhnhat.tech/default.private
Selector                default
Socket                  inet:8891@localhost
```

Lưu file: **Ctrl+X → Y → Enter**

### 4.7 Cấu hình Postfix để dùng OpenDKIM
```bash
nano /etc/postfix/main.cf
```

Thêm những dòng này vào **cuối file**:
```
# DKIM Configuration
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

Lưu file: **Ctrl+X → Y → Enter**

### 4.8 Khởi động OpenDKIM và Postfix
```bash
systemctl start opendkim
systemctl enable opendkim
systemctl restart postfix
systemctl enable postfix

# Kiểm tra trạng thái
systemctl status opendkim
systemctl status postfix
```

---

## PHẦN 5: CẤU HÌNH POSTFIX GỬI EMAIL ĐẾN API

### 5.1 Cấu hình virtual aliases
```bash
nano /etc/postfix/main.cf
```

Tìm dòng `virtual_alias_domains` (hoặc thêm mới):
```
virtual_alias_domains = canhnhat.tech
virtual_alias_maps = hash:/etc/postfix/virtual
```

Lưu file: **Ctrl+X → Y → Enter**

### 5.2 Tạo file virtual aliases
```bash
cat > /etc/postfix/virtual << 'EOF'
@canhnhat.tech webhook
EOF

postmap /etc/postfix/virtual
```

### 5.3 Cấu hình transport (gửi đến webhook)
```bash
cat > /etc/postfix/transport << 'EOF'
canhnhat.tech webhook:
EOF

postmap /etc/postfix/transport
```

### 5.4 Thêm transport map vào Postfix main.cf
```bash
nano /etc/postfix/main.cf
```

Thêm vào cuối:
```
transport_maps = hash:/etc/postfix/transport
```

Lưu file: **Ctrl+X → Y → Enter**

### 5.5 Cấu hình master.cf
```bash
nano /etc/postfix/master.cf
```

Thêm vào **cuối file**:
```
webhook  unix  -       n       n       -       -       pipe
  flags=Xhq user=mail argv=/usr/local/bin/postfix-webhook.sh ${recipient}
```

Lưu file: **Ctrl+X → Y → Enter**

### 5.6 Tạo webhook script
```bash
cat > /usr/local/bin/postfix-webhook.sh << 'EOF'
#!/bin/bash

RECIPIENT="$1"
WEBHOOK_URL="http://127.0.0.1:5000/webhook"
WEBHOOK_SECRET="LETTAI_SECRET6"

# Đọc email từ stdin và gửi đến webhook
/usr/bin/curl -X POST "$WEBHOOK_URL" \
  -H "X-Secret: $WEBHOOK_SECRET" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @- > /dev/null 2>&1

exit 0
EOF

chmod +x /usr/local/bin/postfix-webhook.sh
```

### 5.7 Kiểm tra cấu hình Postfix
```bash
postfix check
postfix reload
```

---

## PHẦN 6: CẤU HÌNH FLASK API SERVICE

### 6.1 Tạo systemd service file
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

### 6.2 Khởi động service
```bash
systemctl daemon-reload
systemctl start mail-api
systemctl enable mail-api

# Kiểm tra trạng thái
systemctl status mail-api
```

---

## PHẦN 7: CẤU HÌNH NGINX (REVERSE PROXY)

### 7.1 Cài đặt Nginx
```bash
apt install nginx -y
```

### 7.2 Tạo file cấu hình Nginx
```bash
cat > /etc/nginx/sites-available/mail.canhnhat.tech << 'EOF'
server {
    listen 80;
    server_name mail.canhnhat.tech canhnhat.tech;
    
    # Chuyển hướng HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name mail.canhnhat.tech canhnhat.tech;
    
    # SSL Certificates
    ssl_certificate /etc/letsencrypt/live/mail.canhnhat.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mail.canhnhat.tech/privkey.pem;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # File upload size
    client_max_body_size 20M;
    
    # Reverse proxy
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

### 7.3 Kích hoạt cấu hình Nginx
```bash
# Tạo symbolic link
ln -s /etc/nginx/sites-available/mail.canhnhat.tech /etc/nginx/sites-enabled/

# Tắt default site (nếu có)
rm -f /etc/nginx/sites-enabled/default

# Kiểm tra syntax
nginx -t

# Khởi động Nginx
systemctl restart nginx
systemctl enable nginx
```

---

## PHẦN 8: CẤU HÌNH FIREWALL

### 8.1 Mở các ports cần thiết
```bash
apt install ufw -y

# Cho phép SSH trước (rất quan trọng!)
ufw allow 22/tcp

# Cho phép HTTP, HTTPS, SMTP
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 25/tcp
ufw allow 587/tcp

# Cho phép POP3, IMAP (tùy chọn)
ufw allow 110/tcp
ufw allow 143/tcp
ufw allow 993/tcp
ufw allow 995/tcp

# Bật UFW
ufw enable
ufw status
```

---

## PHẦN 9: KIỂM TRA VÀ TEST

### 9.1 Kiểm tra trạng thái các service
```bash
# Kiểm tra Postfix
systemctl status postfix
postfix check

# Kiểm tra OpenDKIM
systemctl status opendkim

# Kiểm tra Mail API
systemctl status mail-api

# Kiểm tra Nginx
systemctl status nginx
```

### 9.2 Kiểm tra DNS records
```bash
# Kiểm tra A record
dig mail.canhnhat.tech A

# Kiểm tra MX record
dig canhnhat.tech MX

# Kiểm tra SPF
dig canhnhat.tech TXT

# Kiểm tra DKIM
dig default._domainkey.canhnhat.tech TXT
```

### 9.3 Test Health Check API
```bash
curl https://mail.canhnhat.tech/api/health

# Output mong đợi:
# {"status":"healthy","timestamp":1731666000,"version":"1.0.0"}
```

### 9.4 Test Webhook (gửi email qua curl)
```bash
# Tạo file email test
cat > /tmp/test_email.eml << 'EOF'
From: sender@gmail.com
To: test@canhnhat.tech
Subject: Test Email
Content-Type: text/plain

Hello, this is a test email.
EOF

# Gửi đến webhook
curl -X POST https://mail.canhnhat.tech/webhook \
  -H "X-Secret: LETTAI_SECRET6" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @/tmp/test_email.eml
```

### 9.5 Kiểm tra email đã lưu
```bash
curl https://mail.canhnhat.tech/api/email/test@canhnhat.tech
```

### 9.6 Xem logs
```bash
# Logs Postfix
tail -f /var/log/mail.log

# Logs OpenDKIM
journalctl -u opendkim -f

# Logs API
journalctl -u mail-api -f

# Logs Nginx
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

---

## PHẦN 10: TẠO EMAIL TEST TỪ BÊN NGOÀI

### 10.1 Gửi email từ Gmail đến mail server của bạn
1. Mở Gmail
2. Soạn email mới
3. Gửi đến: **test@canhnhat.tech**
4. Gửi email
5. Kiểm tra logs trên VPS:
```bash
tail -f /var/log/mail.log
```

### 10.2 Lấy email qua API
```bash
curl https://mail.canhnhat.tech/api/email/test@canhnhat.tech
```

---

## PHẦN 11: CẤU HÌNH AUTO-RENEWAL SSL CERTIFICATE

### 11.1 Test renewal
```bash
certbot renew --dry-run
```

### 11.2 Cài đặt auto-renewal
```bash
systemctl enable certbot.timer
systemctl start certbot.timer
systemctl status certbot.timer
```

---

## PHẦN 12: HOÀN TẤT - KIỂM TRA TOÀN BỘ

Để email hoạt động đúng, bạn cần đảm bảo:

✅ **DNS Records (inet):**
- [ ] A record: mail → IP_VPS
- [ ] A record: @ → IP_VPS (nếu cần)
- [ ] MX record: @ → mail.canhnhat.tech (priority 10)
- [ ] SPF: @ → v=spf1 mx a ~all
- [ ] DKIM: default._domainkey → v=DKIM1; k=rsa; p=...

✅ **VPS Services:**
- [ ] Postfix chạy: `systemctl status postfix`
- [ ] OpenDKIM chạy: `systemctl status opendkim`
- [ ] Mail API chạy: `systemctl status mail-api`
- [ ] Nginx chạy: `systemctl status nginx`

✅ **SSL Certificate:**
- [ ] Certificate cài đặt tại: /etc/letsencrypt/live/mail.canhnhat.tech/

✅ **Firewall:**
- [ ] Port 25, 80, 443 mở
- [ ] SSH vẫn hoạt động (port 22)

✅ **Email Test:**
- [ ] API health check hoạt động
- [ ] Gửi email test từ Gmail
- [ ] Lấy email qua API GET

---

## PHẦN 13: TROUBLESHOOTING

Nếu có vấn đề, kiểm tra:

### Email không nhận
```bash
# Kiểm tra mail logs
tail -100 /var/log/mail.log

# Kiểm tra MX record
nslookup -type=MX canhnhat.tech
```

### SPF/DKIM fail
```bash
# Test SPF
dig canhnhat.tech TXT +short

# Test DKIM
dig default._domainkey.canhnhat.tech TXT +short
```

### API không hoạt động
```bash
# Kiểm tra logs
journalctl -u mail-api -n 50

# Test lại
curl -v https://mail.canhnhat.tech/api/health

# Kiểm tra port 5000
netstat -tlnp | grep 5000
```

### Nginx error
```bash
# Test config
nginx -t

# Xem error logs
tail -50 /var/log/nginx/error.log
```

---

## PHẦN 14: API ENDPOINTS

Sau khi setup xong, bạn có các endpoint sau:

### Health Check
```bash
GET https://mail.canhnhat.tech/api/health
```

### Lấy danh sách email
```bash
GET https://mail.canhnhat.tech/api/email/test@canhnhat.tech
```

### Lấy chi tiết email
```bash
GET https://mail.canhnhat.tech/api/inbox/{email_id}
```

### Xóa email theo ID
```bash
DELETE https://mail.canhnhat.tech/api/inbox/{email_id}
```

### Xóa tất cả email của recipient
```bash
DELETE https://mail.canhnhat.tech/api/email/test@canhnhat.tech
```

### Webhook (nhận email)
```bash
POST https://mail.canhnhat.tech/webhook
Header: X-Secret: LETTAI_SECRET6
Body: email raw data
```

---

## LIÊN HỆ HỖ TRỢ

Nếu gặp vấn đề:
1. Kiểm tra logs: `tail -f /var/log/mail.log`
2. Kiểm tra DNS: `dig canhnhat.tech`
3. Test API: `curl -v https://mail.canhnhat.tech/api/health`
4. Kiểm tra services: `systemctl status mail-api`

**Hoàn thành setup! ✅**
