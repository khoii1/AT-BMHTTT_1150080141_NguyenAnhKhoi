# LAB 10 — REMOTE DNS ATTACK: REMOTE DNS CACHE POISONING TRONG MẠNG ẢO CÔ LẬP

## 1. Thông tin bài thực hành

- **Họ và tên:** Nguyễn Anh Khôi
- **Mã số sinh viên:** 1150080141
- **Mã lớp:** 11_ĐH_CNPM2
- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Bài thực hành:** LAB 8 — REMOTE DNS CACHE POISONING TRONG MÔI TRƯỜNG pfSense CÔ LẬP
- **Link Video Demo:** https://youtu.be/6TddKir67t8

---

## 2. Môi trường thực hành

### pfSense Firewall

- **Phiên bản:** pfSense Community Edition 2.7.2-RELEASE
- **WAN (em0 / Card 1):** Bridged / NAT (DHCP/Tĩnh)
- **MGMT (em1 / Card 2):** Host-Only Adapter
  - IP: `192.168.56.254/24`
  - Host: `192.168.56.1`
  - Mục đích: Quản trị WebGUI độc lập, chống khóa quản trị (Anti-lockout).
- **LAN (em2 / Card 3):** Internal Network `lab-lan` (`10.6.0.1/24`)
- **DMZ (em3 / Card 4):** Internal Network `lab-dmz` (`172.16.6.1/24`)
- **DNSLAB (em4 / Card 5):** Internal Network `dns-lab` (`10.9.0.1/24`)
- **ATTACKNET (em5 / Card 6):** Internal Network `attack-net` (`10.10.0.1/24`) — Không gán Upstream Gateway để bảo đảm cách ly nội bộ.

### DNS Server (DNS-SERVER)

- **Hệ điều hành:** Ubuntu Server 24.04 LTS
- **Network:** Internal Network `dns-lab`
- **IP:** `10.9.0.10/24`
- **Gateway:** `10.9.0.1`
- **Dịch vụ:**
  - BIND9 (`named`) đóng vai trò Caching Resolver chuẩn mực, forward zone `example.lab10.test` sang `172.16.6.20`.
  - Script `vuln_resolver_lab10.py` mô phỏng resolver cấu hình yếu với cổng nguồn cố định `UDP 33333`, phục vụ việc tái hiện điều kiện tấn công Kaminsky trong môi trường kiểm thử.

### Máy nạn nhân (VICTIM)

- **Hệ điều hành:** Ubuntu Desktop / Server 24.04 LTS
- **Network:** Internal Network `dns-lab`
- **IP:** `10.9.0.100/24`
- **Gateway:** `10.9.0.1`
- **DNS Server:** `10.9.0.10`

### Máy tấn công (ATTACKER - Kali Linux)

- **Hệ điều hành:** Kali Linux
- **Network:** Internal Network `attack-net` (tách card `dns-lab`, chỉ dùng card `attack-net`)
- **IP:** `10.10.0.200/24`
- **Gateway:** `10.10.0.1`
- **Promiscuous Mode:** Deny (đảm bảo không bắt trộm gói tin)
- **Công cụ:** `kaminsky_remote_lab10.py`, `evil_dns_lab10.py` (cổng 53) và HTTP Server `fake_index.html` (cổng 80).

### AUTH-DNS / APP-TEST

- **IP:** `172.16.6.20/24`
- **Gateway:** `172.16.6.1`
- Chạy script `auth_dns_relay_lab10.py` đóng vai trò Authoritative DNS hợp lệ (UDP 53) và Race Relay (UDP 5300) phục vụ kiểm thử không gian Transaction ID mà không làm flood nghẽn mạng.

### DMZ Web Server (DMZ-WEB)

- **IP:** `172.16.6.10/24`
- **Gateway:** `172.16.6.1`
- Phục vụ website hợp lệ đối chiếu trước và sau khi DNS Cache Poisoning.

---

## 3. Mô hình mạng (Topology)

Mô hình sử dụng pfSense để phân đoạn các mạng LAN, DMZ, DNSLAB và ATTACKNET. Máy Kali được đặt trong ATTACKNET, tách biệt hoàn toàn với DNSLAB để mô phỏng tình huống attacker thực hiện tấn công từ xa mà không thể sniff trực tiếp lưu lượng DNS giữa Resolver và Upstream AUTH-DNS.

| Thành phần          | Địa chỉ IP     | Interface pfSense | Vai trò                                              |
| ------------------- | -------------- | ----------------- | ---------------------------------------------------- |
| pfSense DNSLAB      | `10.9.0.1/24`  | em4               | Gateway DNSLAB                                       |
| DNS-SERVER          | `10.9.0.10`    | -                 | BIND9 / Vulnerable Resolver                          |
| VICTIM              | `10.9.0.100`   | -                 | DNS Client                                           |
| pfSense ATTACKNET   | `10.10.0.1/24` | em5               | Gateway mạng attacker                                |
| ATTACKER Kali       | `10.10.0.200`  | -                 | Remote Attacker + Evil DNS/Web                       |
| AUTH-DNS / APP-TEST | `172.16.6.20`  | -                 | Authoritative DNS (port 53) / Race Relay (port 5300) |
| DMZ-WEB             | `172.16.6.10`  | -                 | Website hợp lệ đối chiếu                             |
| HOST MGMT           | `192.168.56.1` | em1               | Quản trị WebGUI pfSense                              |

---

## 4. Mục tiêu thực hành

1. Hiểu sự khác nhau giữa Local DNS Spoofing (LAB trước) và Remote DNS Cache Poisoning (Kaminsky-style).
2. Phân đoạn ATTACKER khỏi DNSLAB bằng pfSense để attacker không thể sniff trực tiếp query hoặc transaction ID của Resolver.
3. Quan sát vai trò của Transaction ID (16-bit), UDP Source Port, DNS Cache và bản ghi NS/Glue trong quá trình poisoning.
4. Thực hiện mô phỏng Cache Poisoning trong mạng ảo cô lập bằng domain thử nghiệm `.test` và các dải IP private hard-code.
5. Kiểm chứng hậu quả bằng DNS Query và website mô phỏng từ máy nạn nhân VICTIM.
6. Phục hồi cấu hình BIND9 và đánh giá các biện pháp hardening như Source Port Randomization và DNSSEC.

---

## 5. Quy trình thực hiện

### Bước 1: Phân đoạn mạng và cấu hình Firewall pfSense

- Tạo snapshot cho pfSense và các máy ảo trước khi thực hiện.
- Thêm card mạng Adapter 6 trên VirtualBox cho pfSense, gắn vào Internal Network `attack-net`.
- Bật interface ATTACKNET (`10.10.0.1/24`), không gán Upstream Gateway.
- Chuyển máy Kali sang mạng `attack-net`, đặt IP `10.10.0.200/24`, tắt card mạng cũ và đặt `Promiscuous Mode = Deny`.
- Cấu hình firewall rule theo ma trận: cho phép Kali truy vấn DNS đến Resolver (cổng 53), gửi batch TXID sang Relay (UDP 5300); chặn toàn bộ lưu lượng không liên quan tới MGMT, LAN và WAN.

### Bước 2: Thiết lập Baseline từ DNS-SERVER và VICTIM

- Áp dụng cấu hình baseline cho BIND9 trên DNS-SERVER (`named.conf.options` và `named.conf.local`), forward zone `example.lab10.test` sang `172.16.6.20`.
- Chạy authoritative DNS hợp lệ `auth_dns_relay_lab10.py` trên máy DMZ (`172.16.6.20`).
- Từ máy VICTIM (`10.9.0.100`), xóa cache DNS và kiểm tra truy vấn baseline: tên miền `www.example.lab10.test` phân giải chính xác ra IP `172.16.6.10`.
- Chạy `tcpdump` trên Kali để xác nhận tính "Remote": Kali hoàn toàn không thấy gói tin giữa DNS-SERVER và AUTH-DNS.

### Bước 3: Mô phỏng Remote DNS Cache Poisoning (Kaminsky Simulation)

- Dừng dịch vụ BIND9 trên DNS-SERVER và khởi chạy resolver mô phỏng `vuln_resolver_lab10.py` (cố định source port UDP 33333).
- Trên Kali: Khởi chạy Evil DNS (`evil_dns_lab10.py`) trên cổng 53 và Web Server giả mạo trên cổng 80.
- Bật `tcpdump` trên DNS-SERVER để ghi nhận gói tin trao đổi qua cổng 33333.
- Chạy script tấn công `kaminsky_remote_lab10.py` trên Kali: liên tục gửi các truy vấn subdomain ngẫu nhiên tới resolver để buộc resolver hỏi upstream, đồng thời gửi các batch dự đoán TXID sang Race Relay.
- Xác nhận sự kiện `[PENDING]` và `[HIT]` trên AUTH-DNS, cùng thông báo `[CACHE] delegation poisoned` trên Resolver.

### Bước 4: Kiểm chứng kết quả trên VICTIM

- Xóa cache cục bộ trên máy VICTIM (`resolvectl flush-caches`).
- Thực hiện truy vấn lại `www.example.lab10.test`: kết quả chuyển hướng sang IP máy tấn công `10.10.0.200`.
- Sử dụng lệnh `curl`: tải thành công nội dung trang web giả mạo (Fake Site) từ máy Kali.

### Bước 5: Khôi phục BIND9 và Hardening

- Dừng resolver mô phỏng và khởi động lại dịch vụ BIND9 hiện đại.
- Cấu hình thu hẹp quyền recursion (`allow-recursion`), loại bỏ quyền phân giải đệ quy của Kali.
- Sử dụng `tcpdump` bắt gói tin để quan sát tính năng Source Port Randomization của BIND9: cổng UDP nguồn thay đổi ngẫu nhiên liên tục thay vì cố định 33333, làm vô hiệu hóa khả năng đoán gói tin.
- Xác nhận cơ chế DNSSEC Validation (`auto`) và cấu hình `validate-except` cho zone thử nghiệm.

---

## 6. Kết quả thực nghiệm (Bảng so sánh Baseline / Attack / Restore)

| Tiêu chí kiểm thử                            | Trước attack (Baseline)                         | Sau poison (Attack Phase)                    | Khôi phục & Hardening (Restore)             |
| -------------------------------------------- | ----------------------------------------------- | -------------------------------------------- | ------------------------------------------- |
| **Dịch vụ DNS Resolver**                     | BIND 9 hiện đại                                 | Resolver mô phỏng (`vuln_resolver_lab10.py`) | BIND 9 hiện đại                             |
| **Phân giải `www.example.lab10.test`**       | `172.16.6.10` (Hợp lệ)                          | `10.10.0.200` (Bị đầu độc)                   | `172.16.6.10` (Hợp lệ)                      |
| **DNS Upstream của Resolver**                | `172.16.6.20:53` (AUTH-DNS)                     | `10.10.0.200:53` (Evil DNS Kali)             | `172.16.6.20:53` (AUTH-DNS)                 |
| **Website hiển thị trên VICTIM**             | DMZ-WEB hợp lệ (hoặc timeout do pfSense cô lập) | LAB 10 Fake Site (Giao diện giả mạo từ Kali) | DMZ-WEB hợp lệ (Không mở Fake Site)         |
| **Cổng nguồn UDP Upstream Query**            | Ngẫu nhiên động (> 1024)                        | Cố định `33333` (Lỗ hổng mô phỏng)           | Ngẫu nhiên động (Source Port Randomization) |
| **Quyền Recursion của Kali (`10.10.0.200`)** | Bị chặn bởi ACL                                 | Được phép truy vấn                           | Bị chặn hoàn toàn bởi ACL `10.9.0.0/24`     |
| **Attacker sniff DNS-SERVER <-> AUTH-DNS**   | Không (pfSense cô lập subnet)                   | Không (Mô phỏng đoán TXID từ xa)             | Không (pfSense cô lập subnet)               |
| **Trạng thái DNSSEC Validation**             | `auto` (kèm `validate-except`)                  | Không hỗ trợ xác thực chữ ký số              | `auto` (kèm `validate-except`)              |

---

## 7. Phạm vi an toàn

Toàn bộ bài thực hành được triển khai trong môi trường máy ảo cô lập bằng VirtualBox và pfSense.

- Chỉ sử dụng các subnet nội bộ được cấp phép: `10.9.0.0/24`, `10.10.0.0/24` và `172.16.6.0/24`.
- Sử dụng domain thử nghiệm chuẩn RFC 2606 (`.test`), không sử dụng website hay domain thật trên Internet.
- Không thực hiện tấn công trên Wi-Fi, mạng trường học, mạng doanh nghiệp hoặc Internet.
- Không sử dụng các script mô phỏng trên hệ thống không thuộc quyền quản trị.

---

## 8. Kết luận

Qua bài thực hành, sinh viên đã nắm vững:

1. Cơ chế tấn công Remote DNS Cache Poisoning theo kỹ thuật Kaminsky: việc dùng subdomain ngẫu nhiên giúp attacker vượt qua cơ chế lưu đệm (cache hit) của Resolver để liên tục kích hoạt truy vấn upstream.
2. Tầm quan trọng của không gian dự đoán: khi cổng nguồn bị cố định (chỉ còn 16-bit Transaction ID = 65,536 khả năng), nguy cơ bị đầu độc bộ nhớ đệm tăng lên rất cao.
3. Hiệu quả của các giải pháp phòng thủ hiện đại:
   - **Source Port Randomization:** Tăng thêm ~16-bit entropy từ cổng nguồn, mở rộng không gian tìm kiếm lên $2^{32}$, khiến việc đoán mò trở nên bất khả thi trong thực tế.
   - **Recursion ACL:** Giới hạn quyền truy vấn đệ quy, ngăn kẻ tấn công bên ngoài lợi dụng Resolver làm bàn đạp.
   - **DNSSEC:** Cung cấp tính toàn vẹn và xác thực nguồn gốc dữ liệu DNS thông qua chữ ký số, giải quyết triệt để vấn đề giả mạo bản ghi DNS.
