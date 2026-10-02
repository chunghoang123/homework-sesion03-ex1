# Bài 1: Thay đổi cổng SSH (SSH Port Hardening 22 -> 2222)

## 1. Mục tiêu
- Đổi cổng SSH mặc định 22 sang 2222 để tránh botnet quét cổng.
- Mở UFW 2222/tcp trước khi restart SSH để tránh lockout.
- Chỉ dùng user thường `devops` + `sudo`, không dùng `root` trực tiếp.

## 2. Môi trường
- Server: `221.121.1.244`, Ubuntu 22.04.5 LTS, hostname `test`
- User: `devops` (sudo), key: `~/.ssh/id_ed25519_wiservice`
- Từ máy cá nhân (Windows + OpenSSH):
```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519_wiservice devops@221.121.1.244
```

## 3. Các bước thực hiện (giữ nguyên 1 terminal cũ, mở terminal mới để test)

### Bước 1: Kiểm tra hiện trạng
```bash
whoami # -> devops
cat /etc/ssh/sshd_config | grep -i "^#\?Port"
sudo grep -R "^#\?Port" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/ 2>/dev/null
sudo sshd -T | grep -i port
sudo systemctl status ssh --no-pager | head -20
sudo ss -tlnp | grep ssh
sudo ufw status verbose
```

### Bước 2: Sửa Port sang 2222
```bash
sudo nano /etc/ssh/sshd_config
# Tìm #Port 22 -> sửa thành (bỏ #):
Port 2222
```
Nếu có drop-in override:
```bash
sudo grep -R Port /etc/ssh/sshd_config.d/
sudo nano /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
# sửa thành Port 2222
```
Kiểm tra cú pháp:
```bash
sudo sshd -t && echo OK
sudo sshd -T | grep -i port
# mong đợi: port 2222
```

### Bước 3: Mở UFW trước khi restart (bắt buộc)
```bash
sudo ufw allow 2222/tcp
sudo ufw enable
sudo ufw status numbered
# mong đợi: 22/tcp ALLOW (giữ tạm) + 2222/tcp ALLOW
```

### Bước 4: Restart SSH
```bash
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager | head -20
sudo ss -tlnp | grep 2222
sudo sshd -T | grep port
```

> Ubuntu dùng `ssh` (alias `sshd`). Không tắt terminal cũ.

### Bước 5: Test từ máy cá nhân (terminal mới)
```powershell
# Cổng mới - phải SUCCESS
ssh -i $env:USERPROFILE\.ssh\id_ed25519_wiservice -p 2222 devops@221.121.1.244 "whoami; echo SUCCESS-2222"

# Cổng cũ - phải FAIL
ssh -i $env:USERPROFILE\.ssh\id_ed25519_wiservice -p 22 devops@221.121.1.244 -o ConnectTimeout=5 -v
# mong đợi: Connection refused / Timeout
```

### Bước 6: Khóa hẳn cổng 22 (chỉ làm sau khi 2222 OK)
```bash
ssh -i ~/.ssh/id_ed25519_wiservice -p 2222 devops@221.121.1.244
sudo ufw delete allow 22/tcp
# nếu có OpenSSH profile:
# sudo ufw delete allow OpenSSH
sudo ufw deny 22/tcp
sudo ufw status verbose
```

## 4. Kết quả kiểm tra (dán log thực tế vào đây)

```text
# Dán output của:
# sudo sshd -T | grep port
# -> port 2222

# sudo ufw status verbose
# -> 2222/tcp ALLOW, 22/tcp DENY

# sudo ss -tlnp | grep ssh
# -> 0.0.0.0:2222 ...

# ssh -p 2222 devops@221.121.1.244 -> SUCCESS bằng key

# ssh -p 22 devops@221.121.1.244 -> Connection refused / Timeout
```

## 5. Lưu ý chống lockout
- Luôn `ufw allow 2222` trước `systemctl restart ssh`.
- Luôn giữ 1 session cũ, test bằng session mới.
- Nếu lỗi `sshd -t` báo sai cú pháp thì không restart.
- Nếu dùng DigitalOcean Cloud Firewall phải mở thêm Inbound 2222/tcp.

## 6. Đường dẫn nộp bài
```
homework/session_03/ex1/README.md
```
