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

## PHẦN 13: TROUBLESHOOTING

### Email không được nhận
```bash
nslookup -type=MX canhnhat.tech
tail -100 /var/log/syslog | grep postfix
postqueue -p
```

### API không hoạt động
```bash
journalctl -u mail-api -n 50
curl -v http://127.0.0.1:5000/api/health
netstat -tlnp | grep 5000
```

### Nginx error
```bash
nginx -t
tail -50 /var/log/nginx/error.log
```

### SPF/DKIM không hoạt động
```bash
dig canhnhat.tech TXT +short | grep spf
dig default._domainkey.canhnhat.tech TXT +short
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
