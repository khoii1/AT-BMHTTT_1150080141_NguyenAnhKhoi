\# LAB 7 — TRIỂN KHAI GIẢI PHÁP WEB PROTECTION VỚI PFSENSE, SQUID PROXY, E2GUARDIAN VÀ CLAMAV ANTIVIRUS

\## 1. Thông tin bài thực hành

\- \*\*Họ và tên:\*\* Nguyễn Anh Khôi

\- \*\*Mã số sinh viên:\*\* 1150080141

\- \*\*Mã lớp:\*\* 11_ĐH_CNPM2

\- \*\*Môn học:\*\* An toàn và Bảo mật Hệ thống Thông tin

\- \*\*Bài thực hành:\*\* LAB 7 — Web Protection: Phân tách vai trò L3/L4 Firewall và L7 Content Filtering, kiểm soát truy cập danh mục, xác thực mở khóa tạm thời (Bypass), giới hạn dung lượng tải và ngăn chặn mã độc với ClamAV.

\---

\## 2. Môi trường thực hành

\### pfSense Firewall

\- \*\*Phiên bản:\*\* pfSense Community Edition 2.7.2-RELEASE

\- \*\*WAN (em0 / Card 1):\*\* NAT / Bridged Adapter (DHCP)

\- \*\*MGMT (em1 / Card 2):\*\* Host-Only Adapter

&#x20; - IP: `192.168.56.254/24`

&#x20; - Mục đích: Quản trị WebGUI độc lập, chống khóa quản trị (Anti-lockout)

\- \*\*LAN (em2 / Card 3):\*\* Internal Network `lab7-lan`

&#x20; - IP: `10.6.0.1/24`

\- \*\*DMZ (em3 / Card 4):\*\* Internal Network `lab7-dmz`

&#x20; - IP: `172.16.6.1/24`

\### Web Proxy Server (WebProxy-VoAnhKiet)

\- \*\*Tên máy:\*\* `WebProxy-VoAnhKiet`

\- \*\*Hệ điều hành:\*\* Ubuntu Server 24.04 LTS (CLI)

\- \*\*Network Interface 1:\*\* Internal Network `lab7-lan`

&#x20; - IP tĩnh: `10.6.0.30/24`

&#x20; - Gateway: `10.6.0.1`

&#x20; - DNS: `10.6.0.10` (Trỏ về DC)

\- \*\*Network Interface 2:\*\* Host-Only Adapter (`192.168.56.x` - Phục vụ chia sẻ file cấu hình)

\- \*\*Dịch vụ cài đặt:\*\*

&#x20; - \*\*Squid Proxy:\*\* Cổng `3128` (Xử lý cache và giới hạn kích thước file tải)

&#x20; - \*\*E2Guardian:\*\* Cổng `8080` (Lọc nội dung Lớp 7, danh mục Blacklist/Whitelist, cơ chế GBYPASS)

&#x20; - \*\*ClamAV Daemon (`clamav-daemon`):\*\* Quét virus/mã độc thời gian thực qua Unix domain socket

\### Windows Server — Domain Controller \& DNS (DC-VoAnhKiet)

\- \*\*Tên máy:\*\* `DC-VoAnhKiet`

\- \*\*IP tĩnh:\*\* `10.6.0.10/24` (Gateway: `10.6.0.1`)

\- \*\*Dịch vụ:\*\* Active Directory Domain Services, DNS Server phân giải tên miền Forward Lookup Zone `lab7.test` trỏ toàn bộ 13 website mô phỏng về Web Server DMZ (`172.16.6.10`).

\### DMZ Web Server (DMZ-Test-VoAnhKiet)

\- \*\*Tên máy:\*\* `DMZ-Test-VoAnhKiet`

\- \*\*Hệ điều hành:\*\* Ubuntu Server

\- \*\*Network:\*\* Internal Network `lab7-dmz`

\- \*\*IP tĩnh:\*\* `172.16.6.10/24` (Gateway: `172.16.6.1`)

\- \*\*Dịch vụ:\*\* Nginx Web Server thiết lập 13 Virtual Hosts:

&#x20; - `general.lab7.test`, `mideast.lab7.test`, `jobs.lab7.test`, `weapons.lab7.test`, `education.lab7.test`

&#x20; - `entertainment.lab7.test`, `games.lab7.test`, `streaming.lab7.test`, `blockedgame.lab7.test`

&#x20; - `download.lab7.test` (Chứa tệp kiểm thử mã độc chuẩn `eicar.com.txt` và file dữ liệu `110MB.bin`)

\### LAN Test Client (LAN-Test-VoAnhKiet)

\- \*\*Tên máy:\*\* `LAN-Test-VoAnhKiet`

\- \*\*Hệ điều hành:\*\* Ubuntu Server (CLI)

\- \*\*Network:\*\* Internal Network `lab7-lan`

\- \*\*IP tĩnh:\*\* `10.6.0.21/24` (Gateway: `10.6.0.1`)

\---

\## 3. Mô hình mạng (Topology)

````text

&#x20;                                 INTERNET

&#x20;                                     │

&#x20;                                     ▼

&#x20;                            ┌─────────────────┐

&#x20;                            │  WAN (DHCP/NAT) │

┌──────────────────┐         ├─────────────────┤

│ Máy thật Admin   │◄───────►│ MGMT 192.168.56.254 (Host-Only)

└──────────────────┘         ├─────────────────┤

&#x20;                            │ pfSense Firewall│

&#x20;                            └────────┬────────┘

&#x20;                                     │

&#x20;                ┌────────────────────┴────────────────────┐

&#x20;                │                                         │

&#x20;                ▼                                         ▼

&#x20;        LAN 10.6.0.1/24                           DMZ 172.16.6.1/24

&#x20;     (Internal: lab7-lan)                       (Internal: lab7-dmz)

&#x20;                │                                         │

&#x20;    ┌───────────┼───────────┐                             │

&#x20;    │           │           │                             ▼

┌────▼─────┐ ┌───▼────┐ ┌────▼───────┐           ┌───────────────────┐

│ Windows  │ │ Ubuntu │ │  Web Proxy │           │  DMZ Web Server   │

│ Domain DC│ │ Client │ │ Squid/E2G  │           │   172.16.6.10     │

│10.6.0.10 │ │10.6.0.21│ │ 10.6.0.30  │           │(13 Virtual Hosts) │

└──────────┘ └────────┘ └────┬───────┘           └───────────────────┘

&#x20;                            │ (Forward Web Traffic qua pfSense)

&#x20;                            └─────────────────────────────►



\# 4. CÁC CẤU HÌNH CHÍNH



\## 4.1. Quy tắc Tường lửa pfSense (Chống vượt rào và Điều phối Proxy)



\- \*\*Rule 1 (DNS):\*\* Pass LAN subnets → 10.6.0.10, Port UDP/53. Đảm bảo Client có thể phân giải tên miền lab7.test.

\- \*\*Rule 2 (Cho phép Proxy):\*\* Pass LAN subnets → 10.6.0.30, Port TCP/8080. Cho phép Client gửi request tới E2Guardian.

\- \*\*Rule 3 (WebProxy ra ngoài):\*\* Pass 10.6.0.30 → Any, Port TCP/80, 443. Cho phép WebProxy lấy dữ liệu từ Web Server DMZ.

\- \*\*Rule 4 (Anti-Bypass):\*\* Block LAN subnets → Any, Port TCP/80, 443. Chặn các kết nối web trực tiếp từ Client, buộc lưu lượng web phải đi qua Proxy.

\- \*\*Schedules (LAB7\_ENT\_60M):\*\* Tạo lịch biểu 60 phút để kiểm soát khung giờ giải trí và gắn vào luồng WebProxy.



\## 4.2. Cấu hình Squid Proxy (/etc/squid/squid.conf)



\- Cổng lắng nghe nội bộ: `http\_port 3128`.

\- Cấu hình ACL cho phép mạng nội bộ: `acl localnet src 10.6.0.0/24`.

\- Giới hạn kích thước file tải về trong Kịch bản 10: `reply\_body\_max\_size 100 MB all`.



\## 4.3. Cấu hình bộ lọc E2Guardian (e2guardian.conf và e2guardianf1.conf)



\- Cổng nhận kết nối từ Client: `filterport = 8080`.

\- Cổng Proxy đích: `proxyport = 3128`.

\- Mức báo cáo vi phạm chi tiết: `reportinglevel = 3`.

\- Cấu hình mở khóa tạm thời: `bypass = 300`, tương ứng hiệu lực 5 phút.

\- Tích hợp ClamAV Scanner: Kích hoạt `contentscanner = '/etc/e2guardian/contentscanners/clamdscan.conf'`.

\- Phân loại danh sách kiểm soát:

&#x20; - `bannedsitelist`: Nạp danh sách cấm tuyệt đối `hard\_block\_sites.txt` và danh sách cảnh báo `warning\_sites.txt`.

&#x20; - `exceptionsitelist`: Nạp danh sách trắng ngoại lệ `allow\_sites.txt`.



\## 4.4. Cấu hình dịch vụ quét virus ClamAV



\- Cấu hình quyền truy cập Socket cục bộ bằng chỉ thị `LocalSocketMode 666` trong `/etc/clamav/clamd.conf`.

\- Bổ sung người dùng `e2guardian` vào nhóm `clamav`.

\- Cấp quyền truy cập Socket `/var/run/clamav/clamd.ctl` để E2Guardian có thể giao tiếp với ClamAV Scanner.



\# 5. CÁC KỊCH BẢN VÀ KẾT QUẢ KIỂM THỬ



\## Kịch bản 01 — Kiểm tra cơ chế chống vượt rào (Anti-Bypass) trên pfSense



\*\*Mục tiêu:\*\* Chứng minh Client không thể kết nối web trực tiếp nếu không khai báo Proxy.



\*\*Thực thi:\*\*



```bash

curl -I --connect-timeout 5 http://172.16.6.10

````

\*\*Kết quả:\*\* Lệnh bị ngắt kết nối với lỗi `Connection timed out`. Rule Anti-Bypass trên pfSense hoạt động chính xác.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 02 — Chặn theo vị trí địa lý mô phỏng (Geo-Sim Hard Block)

\*\*Mục tiêu:\*\* Cấm tuyệt đối truy cập website khu vực Trung Đông `mideast.lab7.test`.

\*\*Thực thi:\*\*

```bash

curl -x http://10.6.0.30:8080 http://mideast.lab7.test/

```

\*\*Kết quả:\*\* Trả về trang `ACCESS DENIED` từ E2Guardian. Liên kết xác nhận rỗng (`href=""`), không có cơ chế bypass.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 03 — Chặn cứng website tuyển dụng (Job Search Hard Block)

\*\*Mục tiêu:\*\* Cấm nhân viên truy cập website tìm kiếm việc làm `jobs.lab7.test`.

\*\*Thực thi:\*\*

```bash

curl -x http://10.6.0.30:8080 http://jobs.lab7.test/

```

\*\*Kết quả:\*\* Trả về trang `ACCESS DENIED` với danh mục vi phạm `Category: Job Search / Banned Site`.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 04 — Cảnh báo và mở khóa tạm thời 5 phút (Warning \& Temporary Bypass)

\*\*Mục tiêu:\*\* Cảnh báo khi truy cập website vũ khí `weapons.lab7.test`, đồng thời cho phép người dùng xác nhận để mở khóa tạm thời trong 5 phút.

\*\*Thực thi:\*\*

\- \*\*Bước 1:\*\* Gửi yêu cầu truy cập website:

```bash

curl -x http://10.6.0.30:8080 http://weapons.lab7.test/

```

Kết quả trả về trang cảnh báo kèm token `GBYPASS=CD2C1AF889F44CA6B3946CBD9C3ADD1B1792495910`.

\- \*\*Bước 2:\*\* Kích hoạt token thông qua URL bypass. Website được mở khóa và yêu cầu truy cập tiếp theo trả về mã nguồn HTML gốc:

```html
<h1>Weapons Warning Site</h1>
```

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 05 — Ngoại lệ cho phép website giáo dục (Education Whitelist)

\*\*Mục tiêu:\*\* Đảm bảo website học tập `education.lab7.test` luôn được phép truy cập mà không bị cản trở bởi các quy tắc lọc thông thường.

\*\*Thực thi:\*\*

```bash

curl -x http://10.6.0.30:8080 http://education.lab7.test/

```

\*\*Kết quả:\*\* Trả về mã nguồn HTML gốc:

```html
<h1>Education Site</h1>
```

Mã trạng thái HTTP: `200 OK`.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 06 — Kiểm soát giải trí theo khung giờ 60 phút (Schedule Time-Window)

\*\*Mục tiêu:\*\* Chỉ cho phép truy cập `entertainment.lab7.test` và `games.lab7.test` trong khung giờ được quy định bởi Schedule.

\*\*Kết quả:\*\*

\- \*\*Trong khung giờ:\*\* Lệnh `curl -I` trả về `HTTP/1.1 200 OK`.

\- \*\*Ngoài khung giờ:\*\* pfSense ngắt luồng kết nối của WebProxy, lệnh trả về lỗi `504 Gateway Time-out` từ Squid.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 07 — Cảnh báo truyền thông và streaming (Streaming Warning \& Bypass)

\*\*Mục tiêu:\*\* Hiển thị cảnh báo khi người dùng truy cập website streaming `streaming.lab7.test`, kèm liên kết xác nhận mở khóa tạm thời.

\*\*Thực thi:\*\*

```bash

curl -x http://10.6.0.30:8080 http://streaming.lab7.test/

```

\*\*Kết quả:\*\* Trả về trang cảnh báo với danh mục `Category: Streaming / Multimedia`, kèm liên kết `Acknowledge` chứa mã `GBYPASS`.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 08 — Chặn cứng game trực tuyến (Games Hard Block)

\*\*Mục tiêu:\*\* Chặn triệt để website trò chơi trực tuyến `blockedgame.lab7.test` nằm trong danh sách cấm.

\*\*Thực thi:\*\*

```bash

curl -x http://10.6.0.30:8080 http://blockedgame.lab7.test/

```

\*\*Kết quả:\*\* Trả về giao diện `ACCESS DENIED`, danh mục `Category: Games / Hard Block`, không có liên kết mở khóa.

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 09 — Quét và ngăn chặn mã độc với ClamAV Antivirus

\*\*Mục tiêu:\*\* Tự động phát hiện và ngăn chặn tải xuống tệp chứa mã độc thử nghiệm chuẩn EICAR.

\*\*Thực thi:\*\*

```bash

curl -x http://10.6.0.30:8080 http://download.lab7.test/eicar.com.txt

```

\*\*Kết quả:\*\* E2Guardian phối hợp với ClamAV chặn tệp tin và trả về cảnh báo:

`Virus name: Win.Test.EICAR\_HDB-1 FOUND`

\*\*Đánh giá:\*\* PASS.

\## Kịch bản 10 — Giới hạn cứng kích thước tải xuống 100 MB (Hard Download Limit)

\*\*Mục tiêu:\*\* Ngăn người dùng tải tệp vượt quá giới hạn 100 MB nhằm hạn chế việc tiêu tốn tài nguyên và băng thông mạng.

\*\*Thực thi:\*\*

```bash

curl -I -x http://10.6.0.30:8080 http://download.lab7.test/110MB.bin

```

\*\*Kết quả:\*\* Squid Proxy từ chối yêu cầu và trả về mã lỗi:

`HTTP/1.1 413 Request Entity Too Large (ERR\_TOO\_BIG)`

\*\*Đánh giá:\*\* PASS.

\# 7. KIỂM TRA TÍNH TOÀN VẸN MINH CHỨNG

Khởi tạo và xác thực mã băm SHA-256 cho toàn bộ tệp minh chứng bằng PowerShell trên Windows:

```powershell

Get-ChildItem -Path .\\evidence\\\*.\* |

&#x20;   Get-FileHash -Algorithm SHA256 |

&#x20;   Export-Csv -Path .\\evidence\_sha256.csv -NoTypeInformation -Encoding UTF8

```

Lệnh trên thực hiện các công việc sau:

\- Liệt kê các tệp trong thư mục `evidence`.

\- Tính mã băm SHA-256 cho từng tệp.

\- Xuất kết quả ra tệp `evidence\_sha256.csv` để phục vụ kiểm tra tính toàn vẹn của minh chứng.

\# 8. CẤU TRÚC THƯ MỤC BÀN GIAO

```text

LAB7/

└── 11CNPM2\_LAB7\_1150080143\_VoAnhKiet.docx

```

\# 9. TỔNG KẾT

LAB 7 đã triển khai thành công mô hình bảo vệ web nhiều lớp, kết hợp giữa tường lửa mạng pfSense và máy chủ Proxy lọc nội dung chuyên dụng.

\*\*Phân tách trách nhiệm bảo mật rõ ràng:\*\* pfSense đảm nhiệm kiểm soát lưu lượng mạng ở lớp L3/L4 và thực thi chính sách chống vượt rào (Anti-Bypass). WebProxy đảm nhiệm phân tích lưu lượng ở lớp ứng dụng L7, lọc nội dung và phối hợp quét virus.

\*\*Quản trị chính sách linh hoạt:\*\* Hệ thống thực hiện phân loại website với các chính sách chặn cứng (Hard Block), cảnh báo có xác nhận mở khóa tạm thời (Temporary Bypass với GBYPASS) và danh sách ưu tiên cho phép truy cập (Whitelist).

\*\*Kiểm soát thời gian và tài nguyên:\*\* Tích hợp lịch biểu Schedule 60 phút để điều tiết lưu lượng giải trí, đồng thời áp dụng giới hạn kích thước tải xuống 100 MB bằng Squid Proxy.

\*\*Phòng chống phần mềm độc hại:\*\* Tích hợp ClamAV để quét luồng truyền tải HTTP và ngăn chặn tệp thử nghiệm mã độc EICAR trước khi tệp đến máy trạm.

Theo kết quả kiểm thử được ghi nhận, toàn bộ 10 kịch bản của LAB 7 đều đạt trạng thái PASS.
