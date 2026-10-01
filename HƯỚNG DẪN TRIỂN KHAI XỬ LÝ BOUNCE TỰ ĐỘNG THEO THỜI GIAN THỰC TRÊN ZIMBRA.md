# HƯỚNG DẪN TRIỂN KHAI XỬ LÝ BOUNCE TỰ ĐỘNG THEO THỜI GIAN THỰC TRÊN ZIMBRA (POSTFIX LOG MONITORING)

---

## I. MỤC TIÊU & TỔNG QUAN GIẢI PHÁP

Khi triển khai các chiến dịch gửi mail số lượng lớn qua máy chủ Zimbra Mail, việc xuất hiện các địa chỉ email không tồn tại (**Hard Bounce - Lỗi 5xx**) là điều khó tránh khỏi. Nếu tiếp tục gửi lại các địa chỉ này, IP và Domain của hệ thống sẽ nhanh chóng bị liệt vào danh sách đen (Blacklist).

Giải pháp này sử dụng một **Python Service** chạy ngầm theo thời gian thực (Real-time) để:
1. Đọc luồng file `/var/log/zimbra.log` liên tục (tương tự cơ chế `tail -f`).
2. Sử dụng **Biểu thức chính quy (Regex)** để bắt sự kiện thư bị trả về với trạng thái `status=bounced` (Hard Bounce 5xx, User unknown, Recipient rejected).
3. Bóc tách chính xác địa chỉ email nhận bị lỗi và thông tin chi tiết về mã lỗi.
4. Ghi lại dữ liệu đệm tại máy chủ Zimbra và đồng thời phát tín hiệu **Webhook / HTTP API** gửi dữ liệu trực tiếp về cơ sở dữ liệu CRM/ERP/Mail Marketing của hệ thống.

---

## II. KIẾN TRÚC HỆ THỐNG (ARCHITECTURE)

```
[Luồng Email Gửi Đi]
        │
        ▼
┌──────────────────┐        Gửi thất bại (Lỗi 5xx)         ┌──────────────────────┐
│   Zimbra MTA     │ ────────────────────────────────────► │ /var/log/zimbra.log  │
│    (Postfix)     │                                       └──────────┬───────────┘
└──────────────────┘                                                  │
                                                             Tailing Real-time
                                                                      │
                                                                      ▼
                                                           ┌──────────────────────┐
                                                           │  bounce_monitor.py   │
                                                           │   (Python Service)   │
                                                           └──────────┬───────────┘
                                                                      │
                                                      Xử lý Regex & Bóc tách Email
                                                                      │
                                          ┌───────────────────────────┴───────────────────────────┐
                                          ▼                                                       ▼
                               ┌─────────────────────┐                                 ┌────────────────────┐
                               │  Bounced Email Log  │                                 │   Webhook / API    │
                               │  (Local Text File)  │                                 │ (CRM/DB System)    │
                               └─────────────────────┘                                 └────────────────────┘
```

---

## III. HƯỚNG DẪN TRIỂN KHAI CHI TIẾT

### BƯỚC 1: ĐIỀU KIỆN TIÊN QUYẾT & CHUẨN BỊ MÔI TRƯỜNG

Thực hiện đăng nhập máy chủ Zimbra bằng quyền **root**.

1. Kiểm tra môi trường Python 3:
   ```bash
   python3 --version
   ```

2. Cài đặt thư viện `requests` (phục vụ gọi API Webhook về hệ thống):
   ```bash
   pip3 install requests
   ```

3. Tạo thư mục chứa script hệ thống:
   ```bash
   mkdir -p /opt/zimbra/scripts
   ```

---

### BƯỚC 2: XÂY DỰNG SCRIPT PYTHON (`bounce_monitor.py`)

Tạo file script xử lý log chính:

```bash
nano /opt/zimbra/scripts/bounce_monitor.py
```

Nội dung mã nguồn script:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import re
import time
import logging
import requests

# ==================== CẤU HÌNH HỆ THỐNG ====================
LOG_FILE = "/var/log/zimbra.log"
OUTPUT_BLACKLIST_FILE = "/opt/zimbra/scripts/bounced_emails.txt"
SCRIPT_LOG = "/var/log/bounce_monitor.log"

# Cấu hình API endpoint của bạn (để gửi email bị bounce về DB / CRM)
ENABLE_API_CALL = True
API_ENDPOINT = "https://api.yourdomain.com/v1/email/suppression-list"
API_KEY = "YOUR_SECRET_API_KEY"

# ==================== REGEX PATTERNS ====================
# Pattern 1: Bắt log Postfix dạng status=bounced và trích xuất to=<email>
# Ví dụ: to=<user@domain.com>, relay=..., status=bounced (host ... said: 550 ...)
PATTERN_BOUNCE_TO = re.compile(
    r'to=<(?P<email>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})>.*status=bounced',
    re.IGNORECASE
)

# Pattern 2: Bắt lý do lỗi chi tiết (550, 551, 552, 554, User unknown, etc.)
PATTERN_REASON = re.compile(
    r'said:\s*(?P<reason>5\d\d\s+.*|\bUser unknown\b.*|\bAddress rejected\b.*)',
    re.IGNORECASE
)

# Setup logging cho chính script
logging.basicConfig(
    filename=SCRIPT_LOG,
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)

def follow(file_handle):
    """
    Generator đọc file log theo dạng tail -f (real-time).
    Tự động xử lý trường hợp log-rotate.
    """
    file_handle.seek(0, os.SEEK_END)
    while True:
        line = file_handle.readline()
        if not line:
            time.sleep(0.1)  # Tạm dừng 100ms trước khi đọc tiếp
            continue
        yield line

def handle_bounced_email(email, line_content):
    """
    Xử lý khi phát hiện email bị Hard Bounce.
    """
    # 1. Trích xuất lý do lỗi nếu có
    reason_match = PATTERN_REASON.search(line_content)
    reason = reason_match.group('reason').strip() if reason_match else "Hard Bounce (5xx)"

    log_msg = f"BOUNCE DETECTED -> Email: {email} | Reason: {reason}"
    print(log_msg)
    logging.info(log_msg)

    # 2. Ghi email vào file đệm địa phương (Suppression File)
    try:
        with open(OUTPUT_BLACKLIST_FILE, "a") as f:
            f.write(f"{email}\t{reason}\t{time.strftime('%Y-%m-%d %H:%M:%S')}\n")
    except Exception as e:
        logging.error(f"Failed to write to local blacklist file: {e}")

    # 3. Gọi API về Database / CRM của hệ thống
    if ENABLE_API_CALL:
        payload = {
            "email": email,
            "status": "bounced",
            "bounce_type": "Hard",
            "reason": reason,
            "timestamp": time.strftime('%Y-%m-%d %H:%M:%S')
        }
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {API_KEY}"
        }
        try:
            response = requests.post(API_ENDPOINT, json=payload, headers=headers, timeout=5)
            if response.status_code in [200, 201]:
                logging.info(f"API Success for {email}: {response.status_code}")
            else:
                logging.warning(f"API Failed for {email}: Status {response.status_code} - {response.text}")
        except Exception as e:
            logging.error(f"API Connection Error for {email}: {e}")

def main():
    logging.info("Starting Bounce Monitor service...")
    print(f"[*] Monitoring {LOG_FILE} for bounced emails...")

    while True:
        try:
            with open(LOG_FILE, "r", encoding="utf-8", errors="ignore") as f:
                log_lines = follow(f)
                for line in log_lines:
                    # Kiểm tra nhanh từ khóa trước khi chạy Regex nặng
                    if "status=bounced" in line:
                        match = PATTERN_BOUNCE_TO.search(line)
                        if match:
                            email = match.group('email').lower()
                            handle_bounced_email(email, line)

        except FileNotFoundError:
            logging.error(f"Log file {LOG_FILE} not found. Retrying in 5s...")
            time.sleep(5)
        except Exception as e:
            logging.error(f"Unexpected error in main loop: {e}. Restarting reader in 2s...")
            time.sleep(2)

if __name__ == "__main__":
    main()
```

Cấp quyền thực thi file:
```bash
chmod +x /opt/zimbra/scripts/bounce_monitor.py
```

---

### BƯỚC 3: THIẾT LẬP SYSTEMD SERVICE TỰ ĐỘNG CHẠY NGẦM

Tạo service quản lý để đảm bảo script luôn chạy ngầm và tự khởi động cùng OS hoặc tự khôi phục sau crash.

Tạo file `/etc/systemd/system/bounce-monitor.service`:
```bash
nano /etc/systemd/system/bounce-monitor.service
```

Nội dung file cấu hình:
```ini
[Unit]
Description=Real-time Postfix Bounce Log Monitor for Zimbra
After=network.target zimbra.service

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 /opt/zimbra/scripts/bounce_monitor.py
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Kích hoạt và vận hành service:
```bash
systemctl daemon-reload
systemctl enable bounce-monitor
systemctl start bounce-monitor
```

Kiểm tra trạng thái hoạt động:
```bash
systemctl status bounce-monitor
```

---

### BƯỚC 4: KIỂM THỬ THỰC TẾ (TESTING & VERIFICATION)

#### 1. Mở xem log trực tiếp của script:
```bash
tail -f /var/log/bounce_monitor.log
```

#### 2. Giả lập một bản ghi Bounce vào Zimbra Log:
Thực hiện lệnh bơm trực tiếp 1 dòng log giả lập vào file `/var/log/zimbra.log` để test cơ chế nhận diện:
```bash
echo 'Oct  1 10:00:00 mail postfix/smtp[12345]: 4XxxxxX: to=<invalid_user_test123@gmail.com>, relay=gmail-smtp-in.l.google.com[142.250.157.26]:25, delay=1.2, delays=0.01/0.01/0.5/0.68, dsn=5.1.1, status=bounced (host gmail-smtp-in.l.google.com[142.250.157.26] said: 550-5.1.1 The email account that you tried to reach does not exist. 550-5.1.1 User unknown)' >> /var/log/zimbra.log
```

#### 3. Kiểm tra kết quả trích xuất:
Mở file đệm danh sách đen cục bộ:
```bash
cat /opt/zimbra/scripts/bounced_emails.txt
```
*Kết quả xuất hiện địa chỉ `invalid_user_test123@gmail.com` kèm lý do lỗi `550-5.1.1 User unknown` cùng thời gian xử lý.*

---

## IV. BẢO TRÌ VÀ NÂNG CẤP MỞ RỘNG (TÙY CHỌN)

### Chặn gửi lại ngay tại mức Postfix của Zimbra (MTA Level)
Nếu muốn Zimbra tự động **từ chối (Reject)** ngay lập tức khi bất kỳ tài khoản nào cố tình gửi lại cho danh sách email bị bounce đã lưu:

1. Tạo quy tắc `check_recipient_access` từ file danh sách đen:
   ```bash
   awk '{print $1 " REJECT Address previously bounced (Suppressed)"}' /opt/zimbra/scripts/bounced_emails.txt > /opt/zimbra/conf/postfix_reject_bounces
   postmap /opt/zimbra/conf/postfix_reject_bounces
   ```

2. Đưa vào cấu hình Zimbra MTA:
   ```bash
   su - zimbra
   zmprov mcf +zimbraMtaRestriction "check_recipient_access lmdb:/opt/zimbra/conf/postfix_reject_bounces"
   zmmtactl restart
   ```
