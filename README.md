# ictu-cloud-security-lab

Thực hành học phần **An toàn điện toán đám mây** (ICTU) trên OpenNebula miniONE v6.10.3.

> **CHỈ DÙNG TRONG LAB.** Mật khẩu trong `lib/lab.env` được công khai, dành cho môi trường cá nhân, cô lập.
> Không dùng lại ở nơi khác.

## Môi trường

VM `ONE-Lab` chạy Ubuntu 24.04 (hoặc biến thể Xubuntu 24.04) + OpenNebula miniONE **v6.10.3**, bật nested virtualization.

| Cấu hình | Tối thiểu (máy 8 GB RAM) | Khuyến nghị (máy ≥ 16–24 GB) |
|---|---|---|
| RAM VM | 4 GB | 12 GB |
| vCPU | 2–3 | 6 |
| Ổ đĩa | 60 GB (cấp phát động) | 80 GB |

Chọn một trong ba cách tạo máy ảo:

| Hypervisor | Hướng dẫn |
|---|---|
| **VirtualBox** (Windows, kể cả Windows 10) | [Hướng dẫn từng bước bên dưới](#cài-đặt-trên-virtualbox-từng-bước) |
| Hyper-V (Windows 11) | `00-host/setup-onelab.ps1` |
| VMware Workstation | `00-host/VMWARE.md` |

## Cài đặt trên VirtualBox từng bước

Hướng dẫn này dành cho máy Windows. Máy chỉ có 8 GB RAM vẫn làm được nhưng sát giới hạn: tại một thời điểm chỉ chạy một VM lab, và nên đóng các ứng dụng nặng trên Windows.

### Bước 1. Tắt Hyper-V trên Windows

VirtualBox chỉ bật được nested VT-x khi Windows **không** giữ hypervisor. Nếu bạn dùng WSL2, Docker Desktop hoặc Windows Sandbox thì chúng sẽ ngừng hoạt động ở chế độ khởi động này (xem cách giữ cả hai ở `00-host/VMWARE.md`, mục 3: boot entry kép).

1. Vào BIOS/UEFI, bật Intel VT-x (hoặc AMD-V). Kiểm tra trong Task Manager > Performance > CPU: *Virtualization: Enabled*.
2. Mở *Turn Windows features on or off* và **bỏ tích**: Hyper-V, Virtual Machine Platform, Windows Hypervisor Platform, Windows Sandbox.
3. *Windows Security > Device security > Core isolation*: tắt **Memory integrity**.
4. Mở PowerShell (Administrator), chạy lệnh sau rồi khởi động lại máy:

   ```powershell
   bcdedit /set hypervisorlaunchtype off
   ```

5. Kiểm tra: lệnh `systeminfo` không còn dòng *"A hypervisor has been detected"*.

### Bước 2. Tạo mạng host-only

Mạng host-only cho phép máy Windows truy cập Sunstone và SSH vào VM.

1. Mở VirtualBox, vào quản lý Host-only Networks (*Tools > Network* trong bản 7.x, *File > Host Network Manager* trong bản cũ).
2. Tạo một mạng, đặt IPv4 là `192.168.50.1`, mặt nạ `255.255.255.0`, **tắt DHCP**.

Dải này khớp với `VM_LAB_IP=192.168.50.10` trong `lib/lab.env`. Nếu bạn dùng dải khác (ví dụ `192.168.56.0/24`), hãy sửa `VM_LAB_IP` trong `lib/lab.env`.

### Bước 3. Tạo VM `ONE-Lab`

Tạo VM mới trong VirtualBox (kiểu *Linux*, *Ubuntu 64-bit*) với các thông số:

| Mục | Giá trị |
|---|---|
| ISO | Ubuntu 24.04 hoặc Xubuntu 24.04 Minimal (nhẹ hơn, hợp máy 8 GB) |
| RAM | 4096 MB trở lên (xem bảng ở mục Môi trường) |
| CPU | 2–3 vCPU với máy 4 luồng |
| Ổ đĩa | VDI, 60–80 GB, **Dynamically allocated** |
| Adapter 1 | NAT (ra Internet) |
| Adapter 2 | Host-only (chọn mạng ở Bước 2) |

Bật nested virtualization khi VM đang **tắt**. Có thể tích *Settings > System > Processor > Enable Nested VT-x/AMD-V*, hoặc chạy:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "ONE-Lab" --nested-hw-virt on
```

Nếu ô *Enable Nested VT-x/AMD-V* bị xám, Hyper-V hoặc VBS vẫn còn hoạt động. Hãy quay lại Bước 1.

### Bước 4. Cài hệ điều hành và chuẩn bị VM

1. Cài Ubuntu/Xubuntu bình thường, tạo user thường có quyền sudo, đặt hostname là `one-lab`.
2. Sau khi vào được hệ thống, nên chụp snapshot VirtualBox tên `clean-os`.
3. Cập nhật và cài các gói cần thiết (`xmllint` do `check.sh` yêu cầu):

   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo apt install -y git curl openssh-server libxml2-utils cpu-checker
   ```

4. Đặt IP tĩnh cho card host-only. Tên card có thể khác, hãy xem bằng `ip -br a` (thường `enp0s3` là NAT, `enp0s8` là host-only). Card này **không đặt gateway** để giữ Internet qua card NAT. Mỗi lệnh gõ trên **một dòng**:

   ```bash
   sudo nmcli con add type ethernet ifname enp0s8 con-name hostonly ipv4.method manual ipv4.addresses 192.168.50.10/24 ipv4.never-default yes
   sudo nmcli con up hostonly
   ```

   Nếu thiếu `nmcli`, chạy `sudo apt install -y network-manager`. Nếu đã lỡ tạo connection `hostonly`, dùng `sudo nmcli con mod hostonly ...` với các tham số trên thay cho `con add`.

5. Kiểm tra mạng:

   ```bash
   ip -br a                    # enp0s8 có 192.168.50.10/24, enp0s3 vẫn có 10.0.2.15/24
   ip route                    # chỉ có một dòng default (qua enp0s3)
   ping -c 2 8.8.8.8
   ```

   Từ Windows chạy `ping 192.168.50.10` hoặc `ssh <user>@192.168.50.10`.

6. Kiểm tra nested KVM (`install.sh` cũng kiểm tra điều này):

   ```bash
   grep -c -E 'vmx|svm' /proc/cpuinfo    # phải lớn hơn 0
   ls -l /dev/kvm && kvm-ok
   free -h                               # nếu Swap = 0 nên tạo swap 2 GB (xem Khắc phục sự cố)
   ```

### Bước 5. Cài miniONE

Chụp snapshot `before-minione`, sau đó chạy trong VM:

```bash
git clone https://github.com/leeDUongk/ictu-cloud-security-lab.git
cd ictu-cloud-security-lab
bash 01-minione/install.sh      # 10–20 phút, log ở ~/minione-install.log
bash 01-minione/check.sh        # 6 mục PASS là đạt
```

### Bước 6. Sửa địa chỉ FireEdge cho Sunstone

miniONE có thể ghi địa chỉ NAT (`10.0.2.15`) vào cấu hình, khiến Sunstone báo *"FireEdge public endpoint is not working"* khi bạn vào từ Windows. Đặt lại địa chỉ host-only:

```bash
sudo grep -n fireedge_endpoint /etc/one/sunstone-server.conf
sudo sed -i 's|^:public_fireedge_endpoint:.*|:public_fireedge_endpoint: http://192.168.50.10:2616|' /etc/one/sunstone-server.conf
```

Dòng `:private_fireedge_endpoint:` giữ nguyên `http://localhost:2616`. Sau đó khởi động lại dịch vụ đang phục vụ cổng 80. Xem dịch vụ đó bằng `sudo ss -tlnp | grep -E ':(80|2616)\b'`, thường là `apache2`:

```bash
sudo systemctl restart apache2
```

Trên trình duyệt Windows, đăng xuất Sunstone, nhấn Ctrl+F5 rồi đăng nhập lại.

### Bước 7. Đăng nhập Sunstone

Từ Windows mở `http://192.168.50.10/` (hoặc `http://192.168.50.10:2616/`), đăng nhập `oneadmin` với mật khẩu trong `lib/lab.env`. Chụp snapshot `after-minione` để có điểm quay lại.

## Kiểm tra thủ công cài đặt

`01-minione/check.sh` kiểm tra 6 tiêu chí, exit code bằng số tiêu chí chưa đạt. Có thể kiểm tra thủ công:

| Tiêu chí | Lệnh | Kết quả đạt |
|---|---|---|
| Dịch vụ `opennebula` | `systemctl is-active opennebula` | `active` |
| Dịch vụ `opennebula-fireedge` | `systemctl is-active opennebula-fireedge` | `active` |
| Host 0 | `sudo -u oneadmin -H onehost list` | cột `STAT` là `on` |
| Virtual network | `sudo -u oneadmin -H onevnet list` | có ít nhất 1 dòng |
| Bridge `minionebr` | `ip -4 addr show minionebr` | có `172.16.100.1` |
| Web | `curl -s -o /dev/null -w '%{http_code}\n' http://localhost/` | `200`, `301` hoặc `302` (hoặc thử cổng 2616) |

## Nội dung repo

| Thư mục | Mục đích |
|---|---|
| `00-host/` | Tạo máy ảo `ONE-Lab` trên Hyper-V hoặc VMware (VirtualBox: xem README này) |
| `01-minione/` | `install.sh` cài miniONE; `check.sh` kiểm tra hệ thống |
| `bai02-kien-truc/` | `check.sh` kiểm tra báo cáo Bài thực hành 2 (6 mục) |
| `bai03-iaas/` | `check.sh` kiểm tra báo cáo Bài thực hành 3 (5 mục) |
| `bai04-users-acl/` | `check.sh` kiểm tra báo cáo Bài thực hành 4 (6 mục) |
| `lib/` | `common.sh` (hàm dùng chung), `lab.env` (cấu hình và mật khẩu lab) |

Repo này chỉ chứa script và cấu hình để kéo về chạy trong terminal. Tài liệu lab (hướng dẫn, lý thuyết, mẫu báo cáo)
được phát riêng cho sinh viên, không nằm trong repo.

## Kiểm tra báo cáo

```bash
bash bai0X-.../check.sh <đường dẫn REPORT.md>
```

`check.sh` in `[PASS]`/`[FAIL]` cho từng mục của báo cáo và trả exit code bằng số mục chưa điền. Script chỉ kiểm tra
mục báo cáo đã có nội dung thật (từ 20 ký tự trở lên, không phải placeholder), không chấm đúng sai. Tiêu đề mục phải
đúng dạng `## 1.`, `## 2.`... như file mẫu.

## Khắc phục sự cố

| Triệu chứng | Nguyên nhân và cách xử lý |
|---|---|
| Ô *Enable Nested VT-x/AMD-V* bị xám | Hyper-V/VBS còn hoạt động, làm lại Bước 1 |
| `install.sh` báo thiếu vmx/svm hoặc `/dev/kvm` | Chưa bật `--nested-hw-virt on`, hoặc Windows còn giữ hypervisor |
| `nmcli` báo `invalid or not allowed setting ' ipv4'` | Lệnh bị ngắt dòng khi dán. Gõ lại trên một dòng, không dùng `\` |
| Mất Internet sau khi đặt IP host-only | Card host-only bị đặt gateway. Thêm `ipv4.never-default yes` |
| Cài đặt bị treo hoặc `Killed` | Thường do hết RAM. Kiểm tra `sudo dmesg \| grep -i oom`, thêm swap, đóng ứng dụng trên Windows |
| `check.sh` fail ở Host 0 | Đợi vài phút cho `onehost` monitor xong rồi chạy lại. Xem `/var/log/one/monitor.log` |
| Sunstone báo *FireEdge public endpoint is not working* | Làm Bước 6 |

Tạo swap 2 GB khi `free -h` cho thấy Swap = 0:

```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Thu nhỏ và export máy ảo

Trong VM, dọn dẹp rồi ghi số 0 vào vùng trống để file OVA nhỏ hơn. Không xoá `/var/lib/one` và `/var/log/one`.

```bash
sudo apt clean
sudo apt autoremove --purge -y
sudo rm -rf /var/lib/apt/lists/*
sudo journalctl --vacuum-size=50M
sudo rm -rf /var/crash/* /tmp/* /var/tmp/* ~/.cache/*
sudo systemctl stop 'opennebula*' apache2
sudo dd if=/dev/zero of=/zero.fill bs=1M status=progress     # kết thúc bằng lỗi "No space left", bình thường
sync && sudo rm -f /zero.fill && sync
sudo poweroff
```

Trên Windows (PowerShell), tuỳ chọn thu nhỏ đĩa rồi export:

```powershell
cd "C:\Program Files\Oracle\VirtualBox"
.\VBoxManage.exe modifymedium disk "D:\duong\dan\ONE-Lab.vdi" --compact
.\VBoxManage.exe export "ONE-Lab" --output "D:\ONE-Lab.ova"
```

Sau khi import OVA trên máy khác: tạo lại mạng host-only `192.168.50.1/24` (tắt DHCP), gán cho Adapter 2, bật lại
`--nested-hw-virt on` khi VM tắt, rồi chạy `bash 01-minione/check.sh`.

## Quy ước

- Chạy bằng user thường có sudo trên VM `ONE-Lab`.
- Lệnh OpenNebula chạy qua hàm `one()` (= `sudo -u oneadmin -H`). Có thể thêm vào `~/.bashrc`:
  `one() { sudo -u oneadmin -H "$@"; }`
- Mọi mật khẩu và IP nằm trong `lib/lab.env`.
- Sinh viên chỉ clone/pull repo này để lấy script.
