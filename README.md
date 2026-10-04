# LAB_AT_BMHTTT - BÁO CÁO THỰC HÀNH AN TOÀN VÀ BẢO MẬT HỆ THỐNG THÔNG TIN

- **Họ và tên:** Nguyễn Đăng Khoa
- **MSSV:** 1150080099
- **Lớp:** 11CNPM2

---

## Danh mục các bài thực hành

### 1. [LAB 1: Bắt gói tin Telnet và SSH](LAB1/README.md)
- Khảo sát sự khác biệt giữa giao thức truyền tin văn bản rõ (Telnet) và giao thức mã hóa an toàn (SSH) qua Wireshark.

### 2. [LAB 3: Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin](LAB3/README.md)
- Phân loại 5 nguồn đe dọa an toàn thông tin.
- Kiểm thử cơ chế bảo vệ thời gian thực (Real-time Protection) của Microsoft Defender với chuỗi EICAR.
- Tấn công mật khẩu, kiểm toán xác thực qua Windows Event Log (Event ID 4624, 4625, 4648) và thực hiện Credential Rotation.
- Phát hiện Backdoor, kỹ thuật bám trụ (Persistence Run key) và định danh tiến trình mở cổng mạng với Microsoft Sysinternals (Sysmon, Autoruns, Process Explorer).
- Giám sát lưu lượng mạng (Sniffing), phân tích truyền tin văn bản rõ HTTP và mã hóa TLS/HTTPS trên Wireshark.
- Đánh giá nguy cơ suy giảm tính sẵn sàng (DoS qua `local_load_test.py`, phân tích đa nguồn DDoS và Mail Bombing).
- Nhận diện kỹ thuật lừa đảo (Phishing email) và đối chiếu 6 kịch bản Social Engineering.
- Quy trình phục hồi hệ thống sạch và bảng băm toàn vẹn chứng cứ số SHA-256.
- **Báo cáo Word:** [`LAB3/11CNPM2-LAB3_1150080099-NguyenDangKhoa.docx`](LAB3/11CNPM2-LAB3_1150080099-NguyenDangKhoa.docx)
- **Link video:** https://youtu.be/L0_UItYqaqA

### 3. [LAB 4: Khảo sát và đánh giá bề mặt mạng bằng Nmap](LAB4/README.md)
- Khảo sát cấu hình mạng và định tuyến giữa máy quét Kali Linux (WSL2) và máy đích Metasploitable 2 (VMware VMnet8).
- Kiểm tra kết nối ICMP ping và phát hiện máy chủ hoạt động (Host Discovery với `nmap -sn`).
- Quét cổng TCP SYN Stealth Scan (`nmap -sS`), nhận diện các cổng dịch vụ mở nguy hiểm.
- Nhận diện chi tiết phiên bản dịch vụ (`nmap -sV`) và đối chiếu lỗ hổng bảo mật đã biết.
- Nhận diện hệ điều hành mục tiêu qua mẫu vân tay TCP/IP fingerprinting (`nmap -O`).
- Sử dụng Nmap Scripting Engine (NSE) khảo sát dịch vụ SMB và kiểm tra lỗ hổng trên cổng 445 (`smb-os-discovery`, `smb-vuln-ms17-010`).
- Xuất và lưu trữ kết quả quét đa định dạng (`-oA`, `.nmap`, `.xml`, `.gnmap`, `.html`).
- Thực nghiệm củng cố bảo mật (Hardening Before / After): giảm bề mặt tấn công qua kiểm soát dịch vụ và cấu hình tường lửa.
- Trả lời 10 câu hỏi phân tích lý thuyết và tình huống phòng thủ chuyên sâu.
- **Báo cáo Word:** [`LAB4/11CNPM2-LAB4_1150080099-NguyenDangKhoa.docx`](LAB4/11CNPM2-LAB4_1150080099-NguyenDangKhoa.docx)
- **Link video:** https://youtu.be/7dzlYyW29kQ
