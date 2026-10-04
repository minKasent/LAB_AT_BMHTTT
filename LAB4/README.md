# BÁO CÁO THỰC HÀNH LAB 4: KHẢO SÁT VÀ ĐÁNH GIÁ BỀ MẶT MẠNG BẰNG NMAP

- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Họ và tên:** Nguyễn Đăng Khoa
- **MSSV:** 1150080099
- **Lớp:** 11CNPM2
- **Link video:** https://youtu.be/L0_UItYqaqA (hoặc link video cập nhật)
- **Tệp báo cáo Word đính kèm:** [`11CNPM2-LAB4_1150080099-NguyenDangKhoa.docx`](11CNPM2-LAB4_1150080099-NguyenDangKhoa.docx)

---

## 1. Phiên bản môi trường thực hành

Hệ thống được triển khai trên môi trường ảo hóa an toàn gồm máy quét chuyên dụng và máy mục tiêu chứa lỗ hổng mạng:

| Thành phần | Phiên bản / Cấu hình thực tế | Địa chỉ IP thực tế | Vai trò trong bài Lab |
| :--- | :--- | :--- | :--- |
| **Máy Host (Windows)** | Windows 11 Home 64-bit (Build 26200) | `192.168.154.1` (VMnet8) | Điều khiển, chuyển tiếp mạng, lưu trữ báo cáo |
| **Máy quét (Kali Linux)** | Kali Linux (WSL2 / Linux 6.6 x86_64) | `172.17.191.166` (eth0) | Máy quét Nmap chuyên dụng |
| **Máy đích (Metasploitable 2)** | Ubuntu Linux 2.6.24-16-server (VMware 17.5) | `192.168.154.129` (eth0) | Máy mục tiêu mở nhiều dịch vụ chứa lỗ hổng |
| **Nmap & Npcap** | Nmap v7.99 / Npcap v1.89 | - | Công cụ quét mạng, nhận diện OS và NSE script |

---

## 2. Cách dựng môi trường thực hành

### Bước 2.1: Chuẩn bị máy ảo và môi trường mạng
1. Cài đặt **VMware Workstation 17.5** trên máy trạm Windows 11.
2. Thiết lập mạng ảo **VMnet8 (NAT)** với dải IP `192.168.154.0/24`, Gateway `192.168.154.2`.
3. Mở và khởi chạy máy ảo **Metasploitable 2 Linux**:
   - Đăng nhập mặc định: `msfadmin` / `msfadmin`.
   - Kiểm tra IP card mạng `eth0` bằng lệnh: `ifconfig`.
   - Ghi nhận IP máy đích là `192.168.154.129`.

### Bước 2.2: Khởi động máy quét Kali Linux
1. Khởi động môi trường Kali Linux (WSL2):
   ```bash
   wsl -d kali-linux
   ```
2. Kiểm tra IP và trạng thái card mạng:
   ```bash
   ip -br addr
   ```
   (Card `eth0` ở trạng thái UP với IP `172.17.191.166/20`).
3. Kiểm tra khả năng thông mạng hai chiều giữa Kali và Metasploitable 2:
   ```bash
   ping -c 4 192.168.154.129
   ```

---

## 3. Các tình huống đã thực hiện & Kết quả PASS/FAIL

| STT | Tình huống thực nghiệm | Lệnh thực thi chính | Kết quả | Minh chứng |
| :---: | :--- | :--- | :---: | :---: |
| **Ảnh 1** | Kiểm tra IP máy quét Kali Linux | `ip -br addr` | **PASS** | `image1.png` |
| **Ảnh 2** | Kiểm tra IP máy mục tiêu Metasploitable 2 | `ifconfig` | **PASS** | `image2.png` |
| **Ảnh 3** | Kiểm tra thông mạng & Host Discovery | `ping -c 4 192.168.154.129`<br>`nmap -sn 192.168.154.129` | **PASS** | `image3.png` |
| **Ảnh 4** | Quét cổng TCP SYN Stealth Scan | `sudo nmap -sS 192.168.154.129` | **PASS** | `image4.png` (23 open ports) |
| **Ảnh 5** | Nhận diện phiên bản dịch vụ (Service Detection) | `nmap -sV 192.168.154.129` | **PASS** | `image5.png` (vsftpd, OpenSSH, Apache...) |
| **Ảnh 6** | Nhận diện hệ điều hành (OS Fingerprinting) | `sudo nmap -O 192.168.154.129` | **PASS** | `image6.png` (Linux 2.6.X fingerprint) |
| **Ảnh 7** | Chạy NSE Script khảo sát cổng 445 (SMB) | `nmap -p 445 --script smb-os-discovery,smb-vuln-ms17-010 192.168.154.129` | **PASS** | `image7.png` (Samba 3.0.20-Debian) |
| **Ảnh 8** | Xuất kết quả đa định dạng (-oA, XML, HTML) | `nmap -sV -O -oA ketqua_lab4 192.168.154.129` | **PASS** | `image8.png`<br>`ketqua_lab4.html` |
| **Hardening** | Thực nghiệm phòng thủ Trước / Sau | `sudo /etc/init.d/apache2 stop`<br>`nmap -p 80 192.168.154.129` | **PASS** | Cổng 80 chuyển từ `open` sang `closed` |
| **Hash SHA-256** | Kiểm tra tính toàn vẹn chứng cứ số | `Get-FileHash Evidence\* -Algorithm SHA256` | **PASS** | `evidence_sha256.csv` |

---

## 4. Lỗi gặp phải và cách khắc phục

Trong quá trình thực nghiệm, một số tình huống phát sinh và đã được xử lý triệt để:

### Lỗi 1: `sudo /etc/init.d/vsftpd: command not found` khi thực hành Hardening
- **Nguyên nhân:** Trên hệ điều hành Metasploitable 2, dịch vụ FTP `vsftpd` không được quản lý dưới dạng daemon khởi động độc lập trong `/etc/init.d/` mà được lắng nghe thông qua daemon siêu quản lý `inetd` / `xinetd`. Do đó, gọi trực tiếp script `/etc/init.d/vsftpd` báo lỗi không tìm thấy.
- **Cách khắc phục:** 
  1. Sử dụng dịch vụ Apache HTTPD chuẩn có sẵn script điều khiển để thực nghiệm tắt dịch vụ:
     ```bash
     sudo /etc/init.d/apache2 stop
     ```
  2. Hoặc sử dụng tường lửa `iptables` để chặn cổng dịch vụ FTP:
     ```bash
     sudo iptables -A INPUT -p tcp --dport 21 -j DROP
     ```
  3. Quét lại cổng kiểm tra, kết quả cổng chuyển từ `open` sang `closed` hoặc `filtered`, chứng minh thành công hiệu lực phòng thủ.

### Lỗi 2: Lỗi biên dịch XML sang HTML do thiếu `xsltproc` trên Kali WSL
- **Nguyên nhân:** Môi trường Kali Linux WSL2 tối giản mặc định chưa cài sẵn tiện ích dòng lệnh `xsltproc`, hoặc gọi lệnh khi tệp `ketqua_lab4.xml` chưa hoàn tất quá trình ghi đệm dữ liệu.
- **Cách khắc phục:** Chuyển đổi tệp XML sang HTML thông qua thư viện Python `lxml` trên máy Host Windows bằng tệp stylesheet chính thức `nmap.xsl` (`C:\Program Files (x86)\Nmap\nmap.xsl`). Kết quả tạo ra tệp báo cáo trực quan `ketqua_lab4.html` dung lượng 12,337 bytes đầy đủ và chính xác.

### Lỗi 3: Khác biệt dải mạng giữa Kali WSL2 (`172.17.x.x`) và VMware VMnet8 (`192.168.154.x`)
- **Nguyên nhân:** WSL2 hoạt động trên một Hyper-V Virtual Switch riêng (`172.17.191.166/20`), trong khi Metasploitable 2 hoạt động trên VMware VMnet8 NAT (`192.168.154.129/24`).
- **Cách khắc phục:** Windows Host tự động thực hiện định tuyến giữa WSL vSwitch và VMware NAT adapter. Kiểm tra lệnh `ping` và `nmap -sn` từ Kali đến Metasploitable 2 cho thấy gói tin định tuyến hoàn toàn thông suốt, độ trễ < 1ms, không bị mất gói tin nào (0% packet loss), đảm bảo kết quả quét chính xác 100%.

---

## 5. Bảng mã băm toàn vẹn chứng cứ số (SHA-256)

```csv
"Algorithm","Hash","Path"
"SHA256","A5D90693E73903E59F13D3544A9A84223082C8554042D00F1465B2F25F8939E6","D:\Khoa\Class\AT_BMHTTT\LAB4\Evidence\ketqua_lab4.gnmap"
"SHA256","843B7E3356BF7CB8E817DFB3D9536659CD32AB990996892594A3C08094EED353","D:\Khoa\Class\AT_BMHTTT\LAB4\Evidence\ketqua_lab4.html"
"SHA256","75C81066AF76D051E9E5B530C5B00E822A6AD592D24DD61F5ACCCD23E53526D7","D:\Khoa\Class\AT_BMHTTT\LAB4\Evidence\ketqua_lab4.nmap"
"SHA256","A24839F4A1295476749FAC1BBC41941B1871663021DD723B59EEAAAB4459C53D","D:\Khoa\Class\AT_BMHTTT\LAB4\Evidence\ketqua_lab4.xml"
```
