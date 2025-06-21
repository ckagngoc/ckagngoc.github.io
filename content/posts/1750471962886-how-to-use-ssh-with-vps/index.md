---
title: "How to use SSH with VPS"
date: 2025-06-21
draft: false
description: "Hướng dẫn sử dụng SSH với VPS: cài đặt, cấu hình, tạo key, mẹo và tài liệu tham khảo."
tags: ["ssh", "trick"]
---

# Hướng dẫn sử dụng SSH với VPS

## 1. SSH là gì?

**SSH** (Secure Shell) là giao thức mạng giúp bạn kết nối, quản lý và truyền dữ liệu giữa máy tính cá nhân và máy chủ từ xa (VPS) một cách an toàn. SSH mã hóa toàn bộ dữ liệu truyền đi, giúp bảo vệ thông tin đăng nhập và dữ liệu khỏi bị nghe lén.

---

## 2. Cài đặt SSH

### 2.1. Cài đặt SSH Server trên VPS (Ubuntu/Debian)

SSH server là dịch vụ chạy trên VPS, cho phép bạn kết nối từ xa.

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```
- `sudo apt update`: Cập nhật danh sách gói phần mềm.
- `sudo apt install openssh-server`: Cài đặt dịch vụ SSH server.
- `sudo systemctl enable ssh`: Thiết lập SSH tự khởi động cùng hệ thống.
- `sudo systemctl start ssh`: Khởi động dịch vụ SSH ngay lập tức.

### 2.2. Cài đặt SSH Client trên máy cá nhân

- **Windows 10 trở lên**: Đã tích hợp sẵn OpenSSH, bạn có thể dùng trực tiếp trong PowerShell hoặc Command Prompt.
- **Hoặc dùng phần mềm giao diện**:  
  - [PuTTY](https://www.putty.org/)  
  - [MobaXterm](https://mobaxterm.mobatek.net/)

- **Linux/macOS**: SSH client thường đã được cài sẵn. Kiểm tra bằng:
```bash
ssh -V
```
Nếu có kết quả phiên bản SSH, bạn đã sẵn sàng sử dụng.

---

## 3. Kết nối SSH tới VPS

Cú pháp cơ bản:
```bash
ssh username@ip_vps
```
- `username`: Tên tài khoản trên VPS (thường là `root`).
- `ip_vps`: Địa chỉ IP của VPS.

**Ví dụ:**
```bash
ssh root@123.45.67.89
```
Sau khi nhập lệnh, bạn sẽ được yêu cầu nhập mật khẩu của tài khoản trên VPS.

---

## 4. Tạo và sử dụng SSH Key (RSA)

Đăng nhập bằng SSH key giúp bảo mật hơn so với dùng mật khẩu.

### 4.1. Tạo SSH Key trên máy cá nhân

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
- `-t rsa`: Chọn thuật toán RSA.
- `-b 4096`: Độ dài key 4096 bit (bảo mật cao).
- `-C`: Thêm comment (thường là email).

Sau khi chạy, bạn sẽ được hỏi nơi lưu key (mặc định là `~/.ssh/id_rsa`) và đặt passphrase (mật khẩu bảo vệ key, có thể bỏ qua).

### 4.2. Copy public key lên VPS

Cách nhanh nhất:
```bash
ssh-copy-id username@ip_vps
```
- Lệnh này sẽ tự động copy nội dung file `~/.ssh/id_rsa.pub` lên VPS vào file `~/.ssh/authorized_keys`.

**Nếu không có ssh-copy-id, dùng lệnh thủ công:**
```bash
cat ~/.ssh/id_rsa.pub | ssh username@ip_vps "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 4.3. Đăng nhập bằng SSH Key

```bash
ssh -i ~/.ssh/id_rsa username@ip_vps
```
- `-i`: Chỉ định file private key để đăng nhập.

---

## 5. Cấu hình SSH cơ bản trên VPS

Mở file cấu hình SSH:
```bash
sudo nano /etc/ssh/sshd_config
```
**Một số tùy chọn nên chỉnh:**

- **Đổi port mặc định (giảm nguy cơ bị quét cổng):**
  ```
  Port 2222
  ```
- **Chỉ cho phép đăng nhập bằng key (tăng bảo mật):**
  ```
  PasswordAuthentication no
  ```
- **Chỉ cho phép user nhất định đăng nhập:**
  ```
  AllowUsers youruser
  ```

**Sau khi chỉnh sửa, lưu file và khởi động lại SSH:**
```bash
sudo systemctl restart ssh
```

---

## 6. Một số mẹo khi dùng SSH

- **Chuyển file giữa máy và VPS bằng SCP:**
  ```bash
  scp file.txt username@ip_vps:/path/to/dest/
  ```
  - `scp`: Lệnh copy file qua SSH.
  - `file.txt`: File cần gửi.
  - `/path/to/dest/`: Thư mục đích trên VPS.

- **Tạo SSH config để quản lý nhiều VPS dễ dàng:**
  Tạo file `~/.ssh/config` với nội dung:
  ```
  Host myvps
      HostName 123.45.67.89
      User root
      Port 2222
      IdentityFile ~/.ssh/id_rsa
  ```
  Kết nối nhanh bằng:
  ```bash
  ssh myvps
  ```

- **Tạo tunnel (forward port) để truy cập dịch vụ nội bộ trên VPS:**
  ```bash
  ssh -L 8080:localhost:80 username@ip_vps
  ```
  - Truy cập `localhost:8080` trên máy cá nhân sẽ chuyển tiếp tới `localhost:80` trên VPS.

---

## 7. Tài liệu tham khảo

- [DigitalOcean - SSH Essentials](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)
- [SSH Config File](https://linuxize.com/post/using-the-ssh-config-file/)
- [OpenSSH Manual](https://man.openbsd.org/ssh)

---