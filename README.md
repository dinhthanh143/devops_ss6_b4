# Báo cáo: Quản lý tiến trình nền với nohup và tín hiệu Kill

## 1. Mục tiêu
- Khởi chạy tiến trình chạy nền độc lập với phiên làm việc Terminal / SSH bằng công cụ `nohup` và ký tự `&`.
- Giám sát tiến trình hệ thống bằng các công cụ `ps aux`, `pgrep`.
- Quản lý và dừng tiến trình an toàn bằng lệnh `kill` với tín hiệu chuẩn `SIGTERM` (15) và `SIGKILL` (9).

---

## 2. Các bước triển khai

### Bước 1: Tạo kịch bản giám sát `loop-monitor.sh`
Tạo script thực hiện ghi nhận thời gian hệ thống vào `/tmp/monitor.log` sau mỗi 5 giây:

```bash
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF
```

Cấp quyền thực thi cho kịch bản:
```bash
chmod +x loop-monitor.sh
```

---

### Bước 2: Khởi chạy tiến trình chạy nền độc lập với `nohup`
Chạy kịch bản dưới nền (background) và chuyển hướng đầu ra để tiến trình không bị ảnh hưởng khi ngắt kết nối SSH (tín hiệu SIGHUP):

```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

**Output:**
```plaintext
[1] 14820
```
*(Tiến trình được đưa vào background với job ID `[1]` và Process ID `14820`).*

---

### Bước 3: Tra cứu định danh tiến trình (PID)
Sử dụng lệnh `ps aux` kết hợp `grep` hoặc `pgrep` để tìm PID:

```bash
ps aux | grep loop-monitor.sh
```

**Output:**
```plaintext
devops   14820  0.0  0.1   7236  3120 pts/0    S    22:10   0:00 /bin/bash ./loop-monitor.sh
devops   14855  0.0  0.0   6480  2112 pts/0    S+   22:11   0:00 grep --color=auto loop-monitor.sh
```

Hoặc dùng `pgrep`:
```bash
pgrep -f loop-monitor.sh
```

**Output:**
```plaintext
14820
```

---

### Bước 4: Kiểm tra tệp tin log được ghi liên tục
Kiểm tra nội dung tệp tin `/tmp/monitor.log` để xác nhận tiến trình đang ghi dữ liệu đều đặn mỗi 5 giây:

```bash
tail -n 10 /tmp/monitor.log
```

**Output:**
```plaintext
System time: Mon Oct  5 22:10:05 UTC 2026
System time: Mon Oct  5 22:10:10 UTC 2026
System time: Mon Oct  5 22:10:15 UTC 2026
System time: Mon Oct  5 22:10:20 UTC 2026
System time: Mon Oct  5 22:10:25 UTC 2026
System time: Mon Oct  5 22:10:30 UTC 2026
System time: Mon Oct  5 22:10:35 UTC 2026
System time: Mon Oct  5 22:10:40 UTC 2026
System time: Mon Oct  5 22:10:45 UTC 2026
System time: Mon Oct  5 22:10:50 UTC 2026
```

---

### Bước 5: Dừng tiến trình an toàn bằng tín hiệu `SIGTERM` (15)
Gửi tín hiệu yêu cầu dừng tiến trình an toàn (Graceful Shutdown) tới PID `14820`:

```bash
kill -15 14820
```

Kiểm tra lại trạng thái tiến trình sau khi gửi tín hiệu kill:
```bash
ps aux | grep loop-monitor.sh
```

**Output:**
```plaintext
devops   14910  0.0  0.0   6480  2048 pts/0    S+   22:12   0:00 grep --color=auto loop-monitor.sh
```
*(Tiến trình có PID `14820` đã dừng hoàn toàn và không còn tồn tại trong danh sách tiến trình).*

---

## 3. Tổng kết kiến thức tín hiệu (Signals)

| Tín hiệu | Mã số | Mục đích | Ứng dụng |
| :--- | :--- | :--- | :--- |
| **SIGHUP** | `1` | Hangup (ngắt kết nối terminal) | `nohup` bỏ qua tín hiệu này để tiến trình tiếp tục sống. |
| **SIGINT** | `2` | Interrupt từ bàn phím | Tương đương tổ hợp phím `Ctrl + C`. |
| **SIGTERM** | `15` | Termination request | Tín hiệu dừng mặc định và an toàn, cho phép tiến trình dọn dẹp tài nguyên trước khi thoát. |
| **SIGKILL** | `9` | Kill ngay lập tức | Buộc kernel dừng tiến trình ngay lập tức (không thể chặn hay bỏ qua), chỉ dùng khi SIGTERM không phản hồi. |
