# BÁO CÁO THỰC HÀNH LAB 5: CẤU HÌNH TƯỜNG LỬA PFSENSE

- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Họ và tên:** Nguyễn Đăng Khoa
- **MSSV:** 1150080099
- **Lớp:** 11CNPM2
- **Tệp báo cáo Word đính kèm:** [`11CNPM2-LAB5_1150080099-NguyenDangKhoa.docx`](11CNPM2-LAB5_1150080099-NguyenDangKhoa.docx)
- **Tệp báo cáo gốc:** [`Lab 5.docx`](../Lab%205.docx)

---

## 1. Môi trường triển khai và Kiến trúc mạng

Mô hình hệ thống mạng phòng thủ phân vùng (Defense-in-Depth) với tường lửa pfSense CE 2.7.2 đóng vai trò Gateway/Firewall định tuyến và kiểm soát truy cập giữa 3 vùng mạng:

| Vùng mạng / Card | Tên Interface pfSense | Chế độ VMware | Địa chỉ IP / Subnet | Vai trò kiến trúc |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** (Card 1) | `em0` | NAT / Bridged | `192.168.154.130/24` (DHCP) | Kết nối ra Internet bên ngoài |
| **LAN** (Card 2) | `em1` | Host-only (`VMnet1`) | `10.0.0.1/8` (Static) | Mạng nội bộ tin cậy (Quản trị WebGUI & Client test) |
| **DMZ / OPT1** (Card 3) | `em2` | LAN Segment (`pvn`) | `172.16.0.1/16` (Static) | Vùng máy chủ công cộng (Web Server cách ly) |
| **Máy Host Windows** | `VMnet1 Adapter` | Host-only | `10.0.0.100/8` (Static) | Trình duyệt quản trị pfSense WebGUI |
| **Client LAN (Ubuntu)** | `ens33` | Host-only (`VMnet1`) | `10.0.0.2/8` & `10.0.0.3/8` | Máy trạm kiểm thử quy tắc lọc gói LAN |
| **DMZ Web Server** | `ens33` | LAN Segment (`pvn`) | `172.16.0.2/16` | Máy chủ Web dịch vụ nội bộ DMZ |

---

## 2. Các tình huống thực nghiệm & Kết quả kiểm thử

| STT | Tình huống thực nghiệm | Cấu hình Firewall Rule / Lệnh thực thi | Kết quả thực tế |
| :---: | :--- | :--- | :---: |
| **Tình huống 1** | Khởi tạo pfSense, phân bổ 3 Interface & Truy cập WebGUI | Console pfSense gán IP `em0`, `em1`, `em2`; truy cập WebGUI `http://10.0.0.1` từ máy Host | **PASS** - Dashboard hiển thị đầy đủ 3 interface UP |
| **Tình huống 2** | Kiểm tra cơ chế Stateful Inspection & Reset State Table | Disable 2 default rule trên LAN -> ping fail; Thêm rule Pass `LAN subnets -> Any` -> Reset State Table -> ping 8.8.8.8 thành công | **PASS** - Chứng minh cơ chế duy trì trạng thái của bảng State |
| **Tình huống 3** | Quản lý truy cập Internet theo địa chỉ Host cụ thể | Rule 1: Pass `10.0.0.2 -> Any`<br>Rule 2: Block `LAN subnets -> Any` | **PASS** - Host `10.0.0.2` ra mạng OK; Host `10.0.0.3` bị chặn 100% packet loss |
| **Tình huống 4** | Cô lập phân vùng DMZ (Ngăn chặn Lateral Movement) | Tab OPT1:<br>Rule 1: Block `OPT1 subnets -> LAN subnets`<br>Rule 2: Pass `OPT1 subnets -> Any` | **PASS** - DMZ ping LAN `10.0.0.2` bị Drop; DMZ ping Internet `8.8.8.8` thành công |
| **Tình huống 5** | Cấu hình NAT Port Forwarding (Public Web Server) | Bỏ tick *Block private networks* trên WAN; Cấu hình NAT: `WAN:8080 -> 172.16.0.2:80` | **PASS** - Máy Host truy cập `http://192.168.154.130:8080` hiển thị Web Server DMZ |

---

## 3. Danh mục ảnh minh chứng thực hành

- **Ảnh 0:** Xác thực tính toàn vẹn bộ cài `pfSense-CE-2.7.2-RELEASE-amd64.iso` qua mã băm SHA-256 trên PowerShell.
- **Ảnh 1:** Màn hình Console pfSense sau khi gán IP (WAN DHCP, LAN `10.0.0.1/8`, OPT1 `172.16.0.1/16`).
- **Ảnh 2:** Giao diện WebGUI pfSense Dashboard hiển thị trạng thái hệ thống và 3 interface UP.
- **Ảnh 3:** Giao diện cấu hình Rules tab LAN (đã Disable 2 default rules) và thao tác Reset State Table.
- **Ảnh 4:** Kết quả kiểm thử trên máy trạm LAN chứng minh trạng thái Before (Fail) và After (Pass) khi tạo rule mới.
- **Ảnh 5:** Bảng cấu hình Rule LAN quản lý theo IP: Rule Pass cho `10.0.0.2` đặt trên Rule Block toàn bộ LAN.
- **Ảnh 6:** Kết quả kiểm thử đối chiếu: máy `10.0.0.2` ping Internet thành công, máy `10.0.0.3` bị chặn hoàn toàn.
- **Ảnh 7:** Kiểm thử cô lập DMZ: máy DMZ ping sang LAN bị chặn (Drop), ping ra Internet thành công.
- **Ảnh 8:** Truy cập dịch vụ Web DMZ qua địa chỉ WAN và cổng chuyển tiếp (`http://192.168.154.130:8080`) kèm bảng cấu hình NAT Port Forward.

---

## 4. Các câu hỏi lý thuyết cốt lõi

1. **Ý nghĩa thứ tự kiểm tra luật (First-match wins):**
   - pfSense duyệt các luật từ trên xuống dưới theo thứ tự xuất hiện. Gói tin khớp với luật đầu tiên thỏa mãn điều kiện sẽ được thực thi ngay (Pass/Block) và dừng kiểm tra các luật phía dưới.
   - Do đó, các luật chi tiết, đặc thù (như cho phép riêng host `10.0.0.2`) bắt buộc phải đặt trên các luật tổng quát (chặn toàn bộ mạng `LAN subnets`).
2. **Cơ chế hoạt động của Stateful Inspection:**
   - Tường lửa tự động ghi nhận và theo dõi trạng thái các luồng kết nối (TCP SYN, ACK, FIN...) vào bảng trạng thái (State Table).
   - Khi gói tin chiều đi (Outbound) được phép, các gói tin phản hồi hợp lệ (Inbound Return Traffic) tự động được cho phép đi qua mà không cần viết thêm rule chiều về.
3. **Tại sao phải Reset State Table sau khi cập nhật luật chặn:**
   - Các kết nối đã được thiết lập trước đó vẫn còn phiên lưu trong State Table. Nếu không xóa bảng trạng thái, lưu lượng của phiên cũ vẫn tiếp tục được chuyển tiếp cho đến khi kết nối kết thúc hoặc hết thời gian timeout.
4. **Vai trò của Anti-Lockout Rule:**
   - Tự động duy trì quyền truy cập vào cổng quản trị WebGUI (Port 80/443) và SSH (Port 22) từ mạng LAN, ngăn ngừa trường hợp người quản trị vô tình viết nhầm luật chặn làm mất quyền điều khiển thiết bị.
5. **Cơ chế bảo vệ của tính năng Block Private Networks trên WAN:**
   - Ngăn chặn các gói tin mang địa chỉ IP thuộc dải riêng tư RFC 1918 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) từ phía ngoài Internet gửi vào WAN, phòng chống giả mạo IP (IP Spoofing). Trong môi trường lab ảo hóa, cần tạm thời tắt tính năng này để cho phép máy Host (vốn nằm trong dải NAT ảo) truy cập vào cổng WAN.
