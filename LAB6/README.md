# BÁO CÁO THỰC HÀNH LAB 6: XÁC THỰC ACTIVE DIRECTORY VÀ CHÍNH SÁCH TRUY CẬP TRÊN PFSENSE

- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Họ và tên:** Nguyễn Đăng Khoa
- **MSSV:** 1150080099
- **Lớp:** 11CNPM2
- **Link video:** https://youtu.be/fenSvuaSFCM
- **Tệp báo cáo Word đính kèm:** [`11CNPM2-LAB6_1150080099-NguyenDangKhoa.docx`](11CNPM2-LAB6_1150080099-NguyenDangKhoa.docx)
- **Tệp báo cáo gốc:** [`Lab 6.docx`](../Lab%206.docx)
- **Tệp cấu hình pfSense xuất kèm:** [`config.xml`](config.xml)

---

## 1. Môi trường triển khai và Kiến trúc mạng

Mô hình mạng riêng của LAB 6 tách biệt hoàn toàn với LAB 5, sử dụng tường lửa pfSense CE 2.7.2 quản lý 4 phân vùng mạng (`WAN`, `MGMT`, `LAN`, `DMZ`) kết hợp máy chủ Active Directory Domain Controller (`DC01`) và máy chủ Web mô phỏng (`DMZ-WEB`):

| Thiết bị / Interface | Tên Interface pfSense | Chế độ mạng ảo | Địa chỉ IPv4 / Subnet | Gateway / DNS | Vai trò kiến trúc |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **pfSense WAN** | `em0` | NAT (`LAB6-WAN`) | `192.168.154.130/24` (DHCP) | Gateway NAT | Kết nối ra Internet bên ngoài |
| **pfSense MGMT** | `em1` | Host-Only (`VMnet1`) | `192.168.56.254/24` | — | Kênh quản trị WebGUI riêng cho máy thật |
| **pfSense LAN** | `em2` | Internal (`lab6-lan`) | `10.6.0.1/24` | — | Gateway mạng nội bộ `LAN-DOMAIN` |
| **pfSense DMZ** | `em3` | Internal (`lab6-dmz`) | `172.16.6.1/24` | — | Gateway phân vùng máy chủ `DMZ` |
| **Máy thật quản trị** | `VMnet1` | Host-Only | `192.168.56.1/24` | Không GW/DNS | Truy cập quản trị `https://192.168.56.254` |
| **DC01** | `LAN-DOMAIN` | Internal (`lab6-lan`) | `10.6.0.10/24` | GW `10.6.0.1` / DNS `10.6.0.10` | Active Directory (`lab6.test`) + DNS Server |
| **Client-A** | `LAN-DOMAIN` | Internal (`lab6-lan`) | `10.6.0.21/24` | GW `10.6.0.1` / DNS `10.6.0.10` | Máy trạm đại diện tài khoản `sv_a` (`User_Group_A`) |
| **Client-B** | `LAN-DOMAIN` | Internal (`lab6-lan`) | `10.6.0.22/24` | GW `10.6.0.1` / DNS `10.6.0.10` | Máy trạm đại diện tài khoản `sv_b` (`User_Group_B`) |
| **DMZ-WEB** | `DMZ-SWITCH` | Internal (`lab6-dmz`) | `172.16.6.10/24` (+ `.20`, `.30`, `.40`) | GW `172.16.6.1` | Máy chủ Nginx mô phỏng 4 nhóm website |

---

## 2. Cấu hình Tích hợp Active Directory, Captive Portal, Alias & Schedule

### 2.1. Máy chủ xác thực LDAP (`LAB6-AD`)
- **Descriptive name:** `LAB6-AD`
- **Hostname / IP:** `10.6.0.10` | **Port:** `389` (`TCP - Standard`)
- **Base DN:** `DC=lab6,DC=test`
- **Authentication Containers:** `OU=LAB6-Users,DC=lab6,DC=test`
- **Bind DN:** `CN=svc_pfsense_ldap,CN=Users,DC=lab6,DC=test` (Tài khoản dịch vụ đặc quyền tối thiểu)
- **Attributes:** `samAccountName` / `cn` / `memberOf`
- **Kết quả kiểm thử `Diagnostics -> Authentication`:**
  - `sv_a`: Xác thực thành công (`User sv_a authenticated successfully`), thuộc nhóm `User_Group_A`.
  - `sv_b`: Xác thực thành công (`User sv_b authenticated successfully`), thuộc nhóm `User_Group_B`.

### 2.2. Captive Portal (`LAB6_USERS`)
- **Zone name:** `LAB6_USERS` trên cổng **LAN**
- **Authentication Method:** `Use an Authentication backend` -> `LAB6-AD`
- **Trạng thái phiên (`Status -> Captive Portal`):** Ghi nhận đầy đủ phiên đăng nhập hợp lệ của `sv_a` (`10.6.0.21`) và `sv_b` (`10.6.0.22`).

### 2.3. Hệ thống Firewall Aliases & Schedules
- **Aliases:**
  - `CLIENT_A` / `GroupA_Clients`: `10.6.0.21`
  - `CLIENT_B` / `GroupB_Clients`: `10.6.0.22`
  - `POLICY_CLIENTS`: `10.6.0.21, 10.6.0.22`
  - `GENERAL_SITE` (`172.16.6.10`), `JOB_SITE` (`172.16.6.20`), `SOCIAL_SITE` (`172.16.6.30`), `RESTRICTED_SITE` (`172.16.6.40`)
  - `WEB_PORTS`: `80, 443`
- **Schedules:**
  - `WORK_HOURS`: `08:00–17:00`, Thứ 2 – Thứ 6 (Giờ hành chính)
  - `LUNCH_WINDOW` / `LUNCH_BREAK`: `11:30–13:00`, Thứ 2 – Thứ 6 (Giờ nghỉ trưa mở khóa mạng xã hội cho `Client-B`)
  - `AFTER_HOURS`: Ngoài giờ hành chính (`17:30–23:59`)

---

## 3. Bảng kết quả kiểm thử tổng hợp theo kịch bản

| STT / Kịch bản | Nguồn -> Đích | Dự đoán | Thực tế | Nhận xét & Đối chiếu Rule / Log |
| :--- | :--- | :---: | :---: | :--- |
| **LDAP `sv_a` / `sv_b`** | `pfSense` -> `10.6.0.10:389` | Pass | **PASS** | Xác thực thành công và nhận diện đúng nhóm AD `User_Group_A`, `User_Group_B` |
| **Captive Portal** | `Client-A (.21)`, `Client-B (.22)` -> `:8002` | Pass | **PASS** | Chuyển hướng HTTP sang cổng 8002 và cấp phiên truy cập qua `LAB6-AD` |
| **1. `JOB_SITE`** | `Client-A (10.6.0.21)` -> `172.16.6.20:80` | Block | **BLOCK** | Khớp Rule ưu tiên #1 (`Block CLIENT_A -> JOB_SITE`); vẫn truy cập được `GENERAL_SITE (.10)` |
| **2. `RESTRICTED_SITE`** | `Client-A & B (.21, .22)` -> `172.16.6.40:80` | Block | **BLOCK** | Khớp Rule ưu tiên #2 (`Block POLICY_CLIENTS -> RESTRICTED_SITE`) có ghi nhận Firewall Log |
| **3. `SOCIAL_SITE` theo giờ** | `Client-B (10.6.0.22)` -> `172.16.6.30:80` | Allow (Trưa) / Block (Ngoài giờ) | **PASS** | Trong `LUNCH_WINDOW` khớp Rule #3 (`Pass`); ngoài khung giờ rơi xuống Rule #4 (`Block`) |
| **4. `WORK_HOURS`** | `POLICY_CLIENTS` -> `Any:WEB_PORTS` | Allow (Giờ làm) / Block (Ngoài giờ) | **PASS** | Trong `WORK_HOURS` cho phép HTTP/HTTPS; ngoài giờ rơi xuống Rule cuối (`Explicit Deny`) |
| **5. Cô lập `DMZ`** | `DMZ-WEB (172.16.6.10)` -> `DC01 (10.6.0.10)` | Block | **BLOCK** | Rule trên tab `DMZ` (`Block DMZ net -> LAN net`) chặn đứng kết nối khởi tạo từ DMZ vào LAN |
| **6–7. `Reject/Log/State`** | `Client-A/B` -> Các rule kiểm thử | Ghi Log & State | **PASS** | `Reject` trả lỗi `RST/ICMP` tức thì, `Block` chờ timeout; `Status -> System Logs -> Firewall` ghi nhận chính xác |

---

## 4. Trả lời 12 câu hỏi phân tích chuyên sâu

1. **Vì sao LAB 6 phải dùng topology riêng thay vì tiếp tục sử dụng trực tiếp topology LAB 5?**
   - LAB 6 triển khai thêm Captive Portal trên mạng LAN, tích hợp Domain Controller (`AD DS + DNS` tại `10.6.0.10`) và tách riêng cổng quản trị `MGMT` (`192.168.56.254/24`) để tránh tự khóa quyền truy cập WebGUI khi bật Captive Portal hoặc viết rule chặn thử nghiệm trên LAN. Ngoài ra, việc tách topology/config riêng đảm bảo không làm hỏng cấu hình NAT và bộ rule đang dùng của LAB 5.
2. **LDAP được dùng cho chức năng nào trên pfSense? Firewall rule chuẩn có thể dùng username AD làm Source trực tiếp không?**
   - LDAP trên pfSense được dùng để xác thực người dùng (Authentication) cho Captive Portal, VPN hoặc đăng nhập quản trị WebGUI. Bộ lọc gói tin chuẩn (`pf`) ở tầng L3/L4 chỉ khớp theo IP, Port, Interface và Schedule chứ **không** thể dùng trực tiếp chuỗi username AD (`sv_a`, `sv_b`) làm trường Source trong Firewall Rule. Do đó các chính sách được ánh xạ thông qua IP tĩnh/Alias của từng máy trạm.
3. **Vì sao Captive Portal được đặt trên LAN nhưng WebGUI quản trị lại tách sang MGMT?**
   - Captive Portal đặt trên LAN để bắt buộc các máy trạm người dùng (`Client-A`, `Client-B`) phải xác thực tài khoản AD trước khi đi qua pfSense. Việc tách WebGUI sang card `MGMT` (`Host-Only 192.168.56.0/24`) giúp người quản trị trên máy thật luôn truy cập thẳng vào giao diện cấu hình pfSense mà không bị vướng màn hình đăng nhập Captive Portal hay bị ảnh hưởng bởi các rule Block trên LAN.
4. **Search Scope Entire Subtree có ý nghĩa gì khi pfSense truy vấn Active Directory?**
   - `Entire Subtree` cho phép pfSense tìm kiếm đệ quy toàn bộ cây thư mục bắt đầu từ `Base DN` / `Authentication Container` trở xuống tất cả các OU con bên trong, đảm bảo tìm thấy đối tượng user và group dù được phân cấp trong nhiều Organizational Unit (OU).
5. **Tại sao nên dùng service account thay vì Domain Administrator làm LDAP bind?**
   - Tuân thủ nguyên tắc **Đặc quyền tối thiểu (Principle of Least Privilege)**: thao tác LDAP Bind của pfSense chỉ cần quyền đọc thông tin user/group trong thư mục AD. Vì mật khẩu tài khoản Bind phải lưu trong tệp cấu hình `config.xml` của firewall, sử dụng tài khoản dịch vụ quyền thấp (`svc_pfsense_ldap`) giúp ngăn chặn nguy cơ lộ thông tin điều khiển toàn bộ Domain nếu tệp cấu hình firewall bị rò rỉ.
6. **Trong Kịch bản 3, vì sao cần một rule Block SOCIAL_SITE nằm dưới rule Pass có Schedule?**
   - Trên pfSense, khi một Schedule (`LUNCH_WINDOW`) đang ở trạng thái không hoạt động (ngoài giờ nghỉ trưa), rule `Pass` gắn với Schedule đó chỉ bị vô hiệu hóa tạm thời (xem như không tồn tại) chứ **không** tự động đảo ngược thành lệnh `Block`. Nếu không đặt một rule `Block SOCIAL_SITE` cố định nằm ngay bên dưới, gói tin sẽ trôi xuống các rule `Pass` tổng quát hơn ở phía dưới (như rule cho phép Web trong giờ hành chính) và người dùng vẫn truy cập được mạng xã hội.
7. **Vì sao Client-A/B dùng DNS 10.6.0.10 nhưng rule Internet của client không cần cho DNS tới 10.6.0.10?**
   - `Client-A` (`10.6.0.21/24`), `Client-B` (`10.6.0.22/24`) và máy chủ `DC01/DNS` (`10.6.0.10/24`) nằm cùng một mạng con LAN (`10.6.0.0/24`). Các gói tin truy vấn DNS giữa Client và DC01 được chuyển mạch trực tiếp ở tầng liên kết dữ liệu (Layer 2 Switch) trong nội bộ subnet mà không đi qua cổng Default Gateway (`pfSense LAN 10.6.0.1`), nên tường lửa pfSense không chặn luồng này.
8. **Giải thích first-match bằng một ví dụ cụ thể trong ruleset tổng hợp.**
   - Nguyên tắc `First-match wins` nghĩa là pfSense duyệt danh sách luật từ trên xuống dưới và thực thi ngay luật đầu tiên khớp với gói tin. Ví dụ trong bảng luật tổng hợp: Luật #1 (`Block CLIENT_A -> JOB_SITE 172.16.6.20`) được đặt trên Luật #6 (`Pass POLICY_CLIENTS -> Any WEB_PORTS`). Khi `Client-A` truy cập `172.16.6.20:80`, gói tin khớp ngay Luật #1 và bị chặn lập tức. Nếu đặt Luật #6 lên trên Luật #1, gói tin của `Client-A` sẽ khớp Luật #6 trước và đi qua thành công, làm Luật #1 trở nên vô tác dụng.
9. **Vì sao sau khi đổi rule cần tạo kết nối mới hoặc Reset States?**
   - pfSense là tường lửa kiểm tra gói tin có trạng thái (`Stateful Firewall`). Khi một luồng kết nối đã được cho phép trước đó, mục trạng thái (`State entry`) của nó vẫn lưu trong `State Table` và các gói tin thuộc luồng đó sẽ đi tắt qua bước kiểm tra rule. Do đó, sau khi thêm hoặc sửa rule chặn, phải đóng kết nối cũ / tạo phiên mới hoặc vào `Diagnostics -> States -> Reset States` để buộc gói tin phải được đối chiếu lại với bộ rule mới.
10. **Mô hình JOB_SITE/SOCIAL_SITE/RESTRICTED_SITE trong DMZ khác Web Category Filtering thật như thế nào?**
    - Trong bài lab, các nhóm trang web được mô phỏng bằng các địa chỉ IP tĩnh cố định (`.20`, `.30`, `.40`) trên máy chủ `DMZ-WEB` và được lọc ở tầng mạng/giao vận (`L3/L4`) qua Firewall Alias. Trong thực tế, Web Category Filtering thương mại (như trên Sophos XG, FortiGate hoặc pfSense kèm SquidGuard/pfBlockerNG) hoạt động ở tầng ứng dụng (`L7`), kết hợp kiểm tra tên miền (FQDN), trường SNI của TLS, phân tích URL/HTTP Header và cơ sở dữ liệu phân loại hàng triệu tên miền cập nhật liên tục.
11. **Block và Reject khác nhau ra sao về hành vi phía client?**
    - Hành động `Block` âm thầm hủy (`drop`) gói tin mà không gửi bất kỳ phản hồi nào về máy trạm, khiến trình duyệt hoặc công cụ kiểm thử của client phải chờ hết thời gian chờ (`Connection Timed Out`). Ngược lại, hành động `Reject` vừa từ chối gói tin vừa chủ động gửi gói phản hồi báo lỗi ngay lập tức (gói `TCP RST` đối với giao thức TCP hoặc bản tin `ICMP Unreachable` đối với UDP/ICMP), giúp phía client nhận thông báo lỗi `Connection refused` tức thì.
12. **Nếu một rule Block có log nhưng Firewall Log không xuất hiện bản ghi, cần kiểm tra theo thứ tự nào?**
    - Trình tự kiểm tra chuẩn:
      1. Kiểm tra trên chính rule đó đã tích chọn `Log packets that are handled by this rule` và đã bấm **Save + Apply Changes** chưa.
      2. Kiểm tra thứ tự rule (`Rule Order`) xem có rule nào nằm phía trên đã khớp trước (`First-match`) hay không.
      3. Kiểm tra bảng trạng thái (`Diagnostics -> States`) xem luồng kết nối có đang đi theo một State cũ được tạo trước khi bật rule Block hay không (nếu có phải thực hiện `Reset States`).
      4. Kiểm tra đúng Interface, chiều lưu lượng (Inbound vào interface), địa chỉ IP nguồn/đích trong Alias và bộ lọc hiển thị trên trang `Status -> System Logs -> Firewall`.
