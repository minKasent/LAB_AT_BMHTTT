# BÁO CÁO THỰC HÀNH LAB 7: XÂY DỰNG CHÍNH SÁCH WEB PROTECTION TRÊN MÔ HÌNH PFSENSE + WEB PROXY

- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Họ và tên:** Nguyễn Đăng Khoa
- **MSSV:** 1150080099
- **Lớp:** 11CNPM2
- **Tệp báo cáo Word đính kèm:** [`11CNPM2-LAB7_1150080099-NguyenDangKhoa.docx`](11CNPM2-LAB7_1150080099-NguyenDangKhoa.docx)
- **Tệp báo cáo gốc:** [`Lab 7.docx`](../Lab%207.docx)
- **Tệp cấu hình pfSense xuất kèm:** [`config.xml`](config.xml)

---

## 1. Kiến trúc hệ thống và Môi trường triển khai

LAB 7 kế thừa toàn bộ nền tảng mạng 4 phân vùng của LAB 6 (`pfSense-LAB6`, `DC01`, `Client-A`, `Client-B`, `DMZ-WEB`) và bổ sung máy chủ chuyên dụng **WEB-PROXY (`10.6.0.30/24`)** chạy trên Ubuntu Server 24.04 LTS. Do Netgate đã đánh dấu gói `Squid / SquidGuard / Lightsquid` trực tiếp trên pfSense là lỗi thời (deprecated), toàn bộ tầng lọc nội dung Web Layer 7 (`E2Guardian :8080 -> Squid :3128 + ClamAV`) được tách riêng sang máy chủ `WEB-PROXY`, trong khi `pfSense` đóng vai trò Firewall/Gateway kiểm soát luồng và chống bypass proxy:

| Thành phần | Địa chỉ IP / Mạng | Cấu hình & Dịch vụ | Vai trò trong kiến trúc LAB 7 |
| :--- | :--- | :--- | :--- |
| **pfSense-LAB6** | `MGMT`: `192.168.56.254/24`<br>`LAN`: `10.6.0.1/24`<br>`DMZ`: `172.16.6.1/24` | pfSense CE 2.7.2 | Firewall/Gateway ép lưu lượng Web từ Client đi qua `WEB-PROXY`, chặn kết nối trực tiếp `TCP 80/443` |
| **WEB-PROXY** | `10.6.0.30/24`<br>GW `10.6.0.1` / DNS `10.6.0.10` | Ubuntu Server 24.04 LTS<br>`E2Guardian :8080`<br>`Squid 127.0.0.1:3128`<br>`ClamAV daemon` | Lọc nội dung URL/Domain, trang cảnh báo Temporary Bypass (`300s`), quét virus tải tệp (`ClamAV`) và giới hạn tệp `>100 MB` |
| **DC01** | `10.6.0.10/24` | Windows Server AD DS + DNS | Quản lý miền và phân giải 13 bản ghi A thuộc zone `lab7.test` về `172.16.6.10` |
| **Client-A & Client-B** | `10.6.0.21/24`, `10.6.0.22/24` | Cấu hình Proxy `10.6.0.30:8080` | Máy trạm người dùng kiểm thử chính sách Web Protection và chống bypass |
| **DMZ-WEB** | `172.16.6.10/24` | Ubuntu + Nginx (13 Virtual Hosts) | Lưu trữ 13 website mô phỏng `*.lab7.test` và bộ tệp kiểm thử `10MB.bin`, `100MB.bin`, `110MB.bin`, `eicar.com.txt` |

---

## 2. Cấu hình pfSense Ép Lưu Lượng Web Qua Proxy (Chống Bypass)

### 2.1. Danh sách Firewall Aliases
- **`LAB7_CLIENTS`** (`Host`): `10.6.0.21, 10.6.0.22` (`Client-A`, `Client-B`)
- **`LAB7_PROXY`** (`Host`): `10.6.0.30` (`WEB-PROXY`)
- **`WEB_PORTS`** (`Port`): `80, 443`
- **`MAIL_PORTS`** (`Port`): `25, 465, 587, 110, 995, 143, 993`
- **`DMZ_WEB`** (`Host`): `172.16.6.10`

### 2.2. Bộ quy tắc Firewall Rules trên cổng LAN (Theo đúng thứ tự Top-Down)

| Thứ tự | Action | Source | Destination | Port | Ý nghĩa chính sách |
| :---: | :---: | :--- | :--- | :---: | :--- |
| **1** | `Pass` | `LAN net` | `10.6.0.10` | `TCP/UDP 53` | Cho phép máy trạm và Proxy truy vấn DNS về `DC01` |
| **2** | `Pass` | `LAB7_CLIENTS` | `LAB7_PROXY (10.6.0.30)` | `TCP 8080` | Cho phép `Client-A/B` gửi yêu cầu Web vào `E2Guardian` |
| **3** | `Pass` | `LAN net` | `Any` | `MAIL_PORTS` | Cho phép các giao thức Email theo yêu cầu bài gốc |
| **4** | **`Block`** | `LAB7_CLIENTS` | `Any` | `WEB_PORTS (80, 443)` | **Chặn kết nối Web trực tiếp từ Client để chống tắt Proxy (Anti-Bypass)** |
| **5** | `Pass` | `LAB7_PROXY` | `Any` | `WEB_PORTS (80, 443)` | Cho phép riêng `WEB-PROXY` kết nối HTTP/HTTPS ra ngoài |
| **6** | `Pass` | `LAB7_PROXY` | `172.16.6.10` | `TCP 80` | Cho phép `WEB-PROXY` truy cập 13 website mô phỏng tại `DMZ-WEB` |
| **7** | **`Block`** | `LAN net` | `Any` | `Any` | Luật chặn tường minh cuối bảng (Explicit Deny) |

---

## 3. Bảng kết quả kiểm thử 10 Kịch bản Web Protection

| Kịch bản | Đích kiểm thử (FQDN / URL) | Kỳ vọng | Thực tế | Giải thích cơ chế & Đối chiếu Log |
| :--- | :--- | :---: | :---: | :--- |
| **1. Baseline & Chống Bypass** | `http://normal.lab7.test` | Qua Proxy: Allow<br>Tắt Proxy: Block | **PASS (200 OK)**<br>**Direct: BLOCKED** | Qua proxy khớp `exceptionsitelist` (`*EXCEPTION*`). Khi tắt proxy trên trình duyệt, gói tin trực tiếp `TCP 80` bị chặn bởi Rule #4 trên pfSense LAN. |
| **2. Geo-Sim (Chặn vùng địa lý)** | `http://china.lab7.test`<br>`http://mideast.lab7.test` | Hard Block | **BLOCKED (403)** | Nằm trong `bannedsitelist` và `non-bypass list`; E2Guardian trả về trang chặn tùy biến `custom_block_template.html` và không cho mở khóa tạm thời. |
| **3. Job Search** | `http://jobs.lab7.test` | Hard Block | **BLOCKED (403)** | Thuộc danh sách `hard_block_sites.txt`; bị E2Guardian từ chối tuyệt đối (`*DENIED* Banned site`). |
| **4. Warning Content** | `http://weapons.lab7.test`<br>`http://politics.lab7.test`<br>`http://religion.lab7.test` | Warning + Temporary Bypass (`300s`) | **WARNING -> PASS (Bypass 300s)** | Nằm trong `warning_sites.txt` với `bypass = 300` trong `e2guardianf1.conf`. Khi bấm nút xác nhận, E2Guardian cấp mã `GBYPASS` mở khóa tạm thời 5 phút (`*GBYPASS*`). |
| **5. Education (Whitelist)** | `http://education.lab7.test` | Allow (Không cảnh báo) | **PASS (200 OK)** | Nằm trong `/etc/e2guardian/lists/exceptionsitelist`, được ưu tiên cho phép truy cập thẳng không qua bộ lọc chặn. |
| **6. Entertainment & Games** | `http://entertainment.lab7.test`<br>`http://games.lab7.test` | Trong lịch 60p: Allow<br>Ngoài lịch: Block | **Trong giờ: PASS**<br>**Ngoài giờ: BLOCKED** | Áp dụng chính sách khung giờ 60 phút (`Time-Window Policy` với Schedule `LAB7_ENT_60M`): chỉ hoạt động khi lịch đang Active; ngoài khung giờ bị từ chối. |
| **7. Streaming Media** | `http://streaming.lab7.test` | Warning + Temporary Bypass | **WARNING -> PASS (Bypass 300s)** | Hiển thị cảnh báo nội dung Streaming và cho phép người dùng xác nhận mở khóa tạm thời 5 phút. |
| **8. Specific Game** | `http://blockedgame.lab7.test` | Hard Block | **BLOCKED (403)** | Nằm trong danh sách chặn cứng (`hard_block_sites.txt`), không có tùy chọn bypass. |
| **9. Antivirus Scanning** | `http://download.lab7.test/files/eicar.com.txt` | Bị AV chặn | **BLOCKED (Virus Found)** | E2Guardian chuyển tệp qua `clamdscan.conf`; `clamav-daemon` phát hiện mẫu `Win.Test.EICAR_HDB-1 FOUND` và chặn tải xuống (`*INFECTED*`). |
| **10. Download Size (>100 MB)** | `10MB.bin`, `100MB.bin` (Allow)<br>`110MB.bin` (Block) | `<=100MB`: Allow<br>`110MB`: Block | **10/100MB: PASS**<br>**110MB: BLOCKED (413)** | Chỉ thị `reply_body_max_size 104857600 allow all` trên Squid cho phép tệp `<= 100 MB` nhưng chặn tệp `110MB.bin` với mã `TCP_DENIED/413 (ERR_TOO_BIG)`. |

---

## 4. Trả lời 10 câu hỏi phân tích chuyên sâu

1. **Vì sao LAB 7 không cài Squid/SquidGuard trực tiếp trên pfSense?**
   - Netgate đã chính thức đánh dấu các gói cài đặt `Squid`, `SquidGuard` và `Lightsquid` trên pfSense là lỗi thời (**deprecated**) do tồn tại nhiều lỗ hổng bảo mật từ nhánh phần mềm gốc (upstream) chưa được vá và khuyến nghị gỡ bỏ khỏi firewall. Do đó, việc tách dịch vụ Web Proxy (`Squid + E2Guardian + ClamAV`) sang máy chủ Ubuntu Server 24.04 LTS độc lập (`10.6.0.30`) giúp đảm bảo an toàn bảo mật, dễ cập nhật bản vá và tuân thủ nguyên tắc phân tách chức năng giữa Firewall/Gateway (`pfSense`) và Content Filtering Proxy.
2. **Firewall pfSense và Web Proxy khác nhau ở lớp nào và loại thông tin nào có thể kiểm tra?**
   - **Tường lửa pfSense** hoạt động chủ yếu ở tầng Mạng và Giao vận (**Layer 3 / Layer 4**), kiểm tra tiêu đề gói tin IP/TCP/UDP như địa chỉ IP nguồn, IP đích, cổng dịch vụ (Port), giao diện mạng (Interface), trạng thái kết nối (State) và lịch thời gian (Schedule).
   - **Web Proxy (`E2Guardian + Squid`)** hoạt động ở tầng Ứng dụng (**Layer 7**), có khả năng phân tích sâu nội dung giao thức HTTP/HTTPS như tên miền đầy đủ (`FQDN / Host header`), đường dẫn URL, phương thức HTTP, kích thước phản hồi (`Content-Length`), danh mục trang web và quét mã độc trực tiếp trên luồng dữ liệu tệp tải về (`ClamAV`).
3. **Vì sao phải block direct TCP/80,443 từ Client-A/B khi đã cấu hình proxy?**
   - Cấu hình Manual Proxy trên trình duyệt của máy trạm nằm dưới quyền kiểm soát của người dùng cuối. Nếu tường lửa pfSense không có Rule #4 chặn kết nối trực tiếp `TCP/80` và `TCP/443` từ `LAB7_CLIENTS` (`10.6.0.21, 10.6.0.22`) ra ngoài, người dùng chỉ cần tắt cấu hình Proxy trong trình duyệt là có thể đi thẳng qua Default Gateway (`pfSense`) để truy cập các trang web bị cấm, vô hiệu hóa hoàn toàn lớp bảo vệ Web Protection.
4. **Vì sao hostname/category filtering của website Internet thật có thể không ổn định nếu chỉ dựa vào IP?**
   - Các dịch vụ web hiện đại trên Internet sử dụng mạng phân phối nội dung (CDN như Cloudflare, Akamai), cân bằng tải đám mây và lưu trữ chia sẻ (Shared Hosting / Virtual Hosting): một địa chỉ IP có thể đại diện cho hàng trăm website thuộc các danh mục khác nhau, và ngược lại một tên miền có thể thay đổi địa chỉ IP liên tục theo vùng địa lý hoặc thời gian sống DNS (TTL ngắn). Vì vậy, lọc web theo địa chỉ IP ở tầng L3/L4 sẽ gây chặn nhầm hoặc bỏ lọt, đòi hỏi phải lọc theo tên miền (`FQDN / URL / SNI`) ở tầng Proxy L7.
5. **Temporary bypass khác hard block ở điểm nào?**
   - **Hard block (chặn cứng)** từ chối hoàn toàn quyền truy cập vào trang web (nằm trong `bannedsitelist` kèm `non-bypass list`), người dùng không có bất kỳ tùy chọn nào để tự mở khóa.
   - **Temporary bypass (cảnh báo / mở khóa tạm thời với `bypass = 300`)** hiển thị trang cảnh báo chính sách kèm nút xác nhận (`Proceed`); khi người dùng xác nhận, E2Guardian cấp mã thông báo (`GBYPASS` token) cho phép truy cập trang đó trong một khoảng thời gian ngắn (5 phút) đồng thời ghi nhận hành vi vào nhật ký kiểm toán.
6. **Vì sao EICAR qua HTTPS không thể quét nội dung nếu không có SSL MITM?**
   - Khi truy cập qua HTTPS mà không bật SSL MITM (HTTPS Inspection), trình duyệt chỉ gửi bản tin `HTTP CONNECT` tới Proxy để thiết lập đường hầm TCP mã hóa đầu-cuối (`TLS End-to-End Encryption`) trực tiếp giữa Client và Web Server đích. Lúc này Proxy chỉ nhìn thấy tên miền đích (qua `CONNECT / SNI`) còn toàn bộ đường dẫn URL, tiêu đề HTTP và nội dung tệp tải về (như chuỗi EICAR) đều bị mã hóa hoàn toàn, khiến `ClamAV` không thể đọc và đối chiếu chữ ký virus.
7. **CA dùng cho HTTPS inspection tạo ra rủi ro gì nếu private key bị lộ?**
   - Chứng chỉ Root CA của Proxy (`LAB7 E2Guardian Root CA`) đã được cài đặt vào kho `Trusted Root Certification Authorities` trên các máy trạm. Nếu khóa bí mật (`Private Key - ca.key`) của CA này bị lộ, kẻ tấn công có thể tự ý phát hành chứng chỉ giả mạo hợp lệ cho bất kỳ tên miền nào (ngân hàng, email, cổng nội bộ) và thực hiện tấn công nghe lén trung gian (`Man-in-the-Middle`) giải mã toàn bộ lưu lượng HTTPS của các máy trạm mà trình duyệt không hề cảnh báo.
8. **`maxcontentfilecachescansize` có phải luôn là hard download-size limit không? Giải thích.**
   - **Không.** Tham số `maxcontentfilecachescansize` trong E2Guardian chỉ quy định kích thước tối đa của đối tượng được lưu đệm vào đĩa để phục vụ quá trình quét nội dung (như quét virus ClamAV), chứ không phải là chính sách từ chối tải tệp theo dung lượng (những tệp vượt ngưỡng này có thể chỉ bị bỏ qua bước quét bộ đệm tùy phiên bản). Để áp đặt giới hạn chặn cứng tệp tải về lớn hơn 100 MB một cách chính xác, phải kết hợp chỉ thị `reply_body_max_size 104857600 allow all` trên tầng Squid Proxy.
9. **Vì sao Schedule 60 phút không đồng nghĩa với daily category quota của Sophos?**
   - Tính năng **Daily Category Quota** trên Sophos UTM/XG sử dụng bộ đếm thời gian tích lũy (`cumulative timer`) cho từng người dùng: người dùng có thể truy cập nhóm trang giải trí nhiều lần rải rác trong ngày miễn là tổng thời gian sử dụng chưa vượt quá 60 phút. Trong khi đó, **Schedule 60 phút** trên pfSense (`LAB7_ENT_60M`) là chính sách khung giờ cố định theo đồng hồ hệ thống (**Time-Window Policy**, ví dụ từ `12:00` đến `13:00`), không đếm thời gian sử dụng thực tế của từng phiên.
10. **Khi browser lỗi nhưng E2Guardian không có log, giả thuyết đầu tiên cần kiểm tra là gì?**
    - Giả thuyết đầu tiên cần kiểm tra là **yêu cầu từ trình duyệt của máy trạm chưa hề đi tới dịch vụ E2Guardian tại `10.6.0.30:8080`**. Nguyên nhân thường gặp nhất là:
      1. Trình duyệt chưa bật cấu hình Manual Proxy `10.6.0.30:8080` nên gửi kết nối trực tiếp và bị pfSense chặn ở Rule #4.
      2. Máy trạm không phân giải được DNS hoặc mất kết nối mạng tới `10.6.0.30`.
      3. Dịch vụ `e2guardian` trên cổng `8080` đang dừng hoạt động.
