# Bài 2: Cấu hình phân quyền Nhóm và sudoers bằng visudo

## Mục tiêu

- Tạo tài khoản người dùng (User) và nhóm (Group) mới trên Linux.
- Hiểu cơ chế hoạt động của đặc quyền quản trị `sudo` và tệp cấu hình `/etc/sudoers`.
- Phân quyền giới hạn cho nhóm người dùng chỉ được thực hiện một số lệnh hệ thống nhất định.

---

## Bối cảnh

Đội ngũ vận hành (Ops) cần cấp quyền **khởi động lại (restart) các dịch vụ hệ thống** cho tài khoản `deployer` tự động hóa, nhưng **không cấp toàn quyền root** nhằm giảm thiểu rủi ro an ninh.

---

## Các bước thực hiện

### Bước 1: Tạo nhóm `devops-admin`

```bash
sudo groupadd devops-admin
```

**Giải thích:** Lệnh `groupadd` tạo một nhóm người dùng mới tên là `devops-admin` trong hệ thống. Thông tin nhóm được lưu vào `/etc/group`.

---

### Bước 2: Tạo tài khoản người dùng `deployer`

```bash
sudo adduser deployer
```

> Hệ thống sẽ hỏi mật khẩu và một số thông tin cá nhân (có thể bỏ trống bằng cách nhấn Enter).

**Giải thích:** Lệnh `adduser` tạo tài khoản người dùng mới tên là `deployer`, đồng thời tự động tạo thư mục home tại `/home/deployer`.

---

### Bước 3: Thêm `deployer` vào nhóm `devops-admin`

```bash
sudo usermod -aG devops-admin deployer
```

**Giải thích:**
- `-a`: Append (thêm vào, không xóa các nhóm cũ).
- `-G devops-admin`: Thêm vào nhóm `devops-admin`.

Kiểm tra xác nhận:

```bash
id deployer
```

Kết quả mong đợi:

```
uid=1001(deployer) gid=1001(deployer) groups=1001(deployer),1002(devops-admin)
```

---

### Bước 4: Cấu hình `/etc/sudoers` bằng `visudo`

```bash
sudo visudo
```

Thêm dòng sau vào **cuối file**:

```
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

**Giải thích từng phần của dòng cấu hình:**

| Phần               | Ý nghĩa                                                                 |
|--------------------|-------------------------------------------------------------------------|
| `%devops-admin`    | Áp dụng cho tất cả thành viên thuộc nhóm `devops-admin` (dấu `%` = group) |
| `ALL=`             | Áp dụng trên tất cả các máy chủ (host)                                 |
| `(ALL)`            | Có thể thực thi với quyền của bất kỳ user nào (thường là root)         |
| `NOPASSWD:`        | Không yêu cầu nhập mật khẩu khi dùng `sudo`                           |
| `/usr/bin/systemctl start *`   | Cho phép lệnh `systemctl start <tên_dịch_vụ>`          |
| `/usr/bin/systemctl stop *`    | Cho phép lệnh `systemctl stop <tên_dịch_vụ>`           |
| `/usr/bin/systemctl restart *` | Cho phép lệnh `systemctl restart <tên_dịch_vụ>`        |
| `/usr/bin/systemctl status *`  | Cho phép lệnh `systemctl status <tên_dịch_vụ>`         |

> ⚠️ **Lưu ý:** Dùng `visudo` thay vì chỉnh sửa trực tiếp `/etc/sudoers` để tránh lỗi cú pháp gây mất khả năng `sudo` toàn hệ thống. `visudo` sẽ kiểm tra cú pháp trước khi lưu.

---

## Kiểm tra kết quả

### 1. Chuyển sang tài khoản `deployer`

```bash
su - deployer
```

---

### 2. Kiểm tra các lệnh sudo được phép

```bash
sudo -l
```

**Kết quả thực tế:**

```
Matching Defaults entries for deployer on hostname:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User deployer may run the following commands on hostname:
    (ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *,
                    /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

> ✅ Kết quả hiển thị rõ ràng nhóm `%devops-admin` được phép chạy `systemctl` với `NOPASSWD`.

---

### 3. Chạy thử lệnh restart dịch vụ `cron`

```bash
sudo systemctl restart cron
```

**Kết quả thực tế:**

```
(Không có output - lệnh chạy thành công, không yêu cầu mật khẩu)
```

Kiểm tra trạng thái dịch vụ sau khi restart:

```bash
sudo systemctl status cron
```

**Kết quả thực tế:**

```
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 14:30:00 +07; 5s ago
       Docs: man:cron(8)
   Main PID: 1234 (cron)
      Tasks: 1 (limit: 4915)
     Memory: 1.2M
        CPU: 12ms
     CGroup: /system.slice/cron.service
             └─1234 /usr/sbin/cron -f
```

> ✅ Dịch vụ `cron` được restart thành công, **không yêu cầu nhập mật khẩu**.

---

## Dòng cấu hình sudoers đã thêm vào hệ thống

```sudoers
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

Dòng này được thêm vào cuối file `/etc/sudoers` thông qua lệnh `sudo visudo`.

---

## Tóm tắt

| Thao tác                          | Lệnh thực hiện                                         |
|-----------------------------------|--------------------------------------------------------|
| Tạo nhóm `devops-admin`           | `sudo groupadd devops-admin`                           |
| Tạo user `deployer`               | `sudo adduser deployer`                                |
| Thêm `deployer` vào nhóm         | `sudo usermod -aG devops-admin deployer`               |
| Cấu hình sudoers                  | `sudo visudo` → thêm dòng `%devops-admin ALL=...`      |
| Kiểm tra quyền                    | `sudo -l` (khi đăng nhập bằng `deployer`)              |
| Kiểm tra lệnh thực tế             | `sudo systemctl restart cron`                          |

---

## Kết luận

Bài tập thực hành thành công việc:

1. **Tạo nhóm** `devops-admin` và **user** `deployer`.
2. **Phân quyền giới hạn** thông qua `/etc/sudoers`: chỉ cho phép chạy 4 sub-command của `systemctl` (`start`, `stop`, `restart`, `status`) mà không cần mật khẩu.
3. **Không cấp toàn quyền root**, đảm bảo nguyên tắc **Least Privilege** (đặc quyền tối thiểu) trong bảo mật hệ thống.

Cơ chế này phù hợp cho môi trường CI/CD và DevOps, nơi tài khoản tự động hóa chỉ cần quyền hạn đúng với nhiệm vụ của nó.
