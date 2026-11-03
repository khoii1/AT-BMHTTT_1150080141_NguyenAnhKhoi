# LAB 9 — LOCAL DNS ATTACK: DNS SPOOFING VÀ CACHE POISONING TRONG MẠNG ẢO CÔ LẬP

## 1. Thông tin bài thực hành

- **Họ và tên:** Nguyễn Anh Khôi
- **Mã số sinh viên:** 1150080141
- **Mã lớp:** 11_ĐH_CNPM2
- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Bài thực hành:** LAB 9 (Kế thừa pfSense LAB 8) — Local DNS Attack: Xây dựng Recursive DNS Resolver BIND9, thực thi tấn công HOSTS-file redirection, Direct DNS Spoofing và DNS Cache Poisoning trong mạng ảo cô lập; đánh giá cơ chế phòng thủ DNSSEC, DNS COOKIE và phân đoạn mạng.
- **Link Video Demo:** https://youtu.be/XyNU5y_eL8s

---

## 2. Môi trường thực hành

### pfSense Firewall (pfSense - LAB 9)

- **Phiên bản:** pfSense Community Edition 2.7.2-RELEASE (Kế thừa từ LAB 8)
- **WAN (em0 / Card 1):** Bridged / NAT (DHCP/Tĩnh 192.168.250.0/24)
- **MGMT (em1 / Card 2):** Host-Only Adapter
  - IP: `192.168.56.254/24` (Host: `192.168.56.1`)
  - Mục đích: Quản trị WebGUI độc lập, chống khóa quản trị (Anti-lockout)
- **LAN (em2 / Card 3):** Internal Network `lab-lan` (`10.6.0.1/24` - Kế thừa LAB 8 LAN-DOMAIN, được cô lập an toàn)
- **DMZ (em3 / Card 4):** Internal Network `lab-dmz` (`172.16.6.1/24` - Kế thừa LAB 8)
- **DNSLAB (em4 / Card 5 - OPT2):** Internal Network `dns-lab`
  - IP: `10.9.0.1/24` (Static IPv4, **không** gán Upstream Gateway để tránh lỗi phân loại WAN-type)

### DNS Server BIND9 (DNS-SERVER)

- **Tên máy:** `DNS-SERVER`
- **Hệ điều hành:** Ubuntu Server 24.04 LTS
- **Network Interface:** Internal Network `dns-lab`
  - IP tĩnh: `10.9.0.10/24`
  - Gateway: `10.9.0.1`
- **Dịch vụ cài đặt:**
  - BIND9 (`named`): Đóng vai trò Authoritative cho zone `lab9.test` và Recursive/Caching Resolver cho mạng `10.9.0.0/24`.
  - Cấu hình chuyển tiếp: Chuyển tiếp truy vấn `external.lab9.test` tới Authoritative DNS mô phỏng trong DMZ (`172.16.6.20`).
  - Thiết lập đặc thù bài Lab: `send-cookie no;` riêng cho server `172.16.6.20` để tái lập kịch bản on-path spoofing.

### Máy nạn nhân (VICTIM)

- **Tên máy:** `VICTIM`
- **Hệ điều hành:** Ubuntu Desktop / Server 24.04 LTS
- **Network Interface:** Internal Network `dns-lab`
  - IP tĩnh: `10.9.0.100/24`
  - Gateway: `10.9.0.1`
  - DNS Server: `10.9.0.10` (Trỏ duy nhất về BIND9 qua Netplan, cấm chỉnh sửa đè `/etc/resolv.conf`)

### Máy tấn công (ATTACKER - Kali)

- **Tên máy:** `ATTACKER-KALI`
- **Hệ điều hành:** Kali Linux
- **Network Interface:** Internal Network `dns-lab`
  - IP tĩnh: `10.9.0.200/24`
  - VirtualBox Adapter Setting: **Promiscuous Mode = Allow VMs** (bắt buộc để bắt gói tin unicast giữa các VM)
- **Công cụ & Dịch vụ:**
  - Python3 Scapy (`dns_spoof_lab9.py`): Thực thi giả mạo phản hồi DNS ở cả 2 chế độ `victim` và `resolver`.
  - Fake Web Server: Python HTTP Server chạy cổng `80`, phục vụ trang web giả mạo `fake_index.html`.
  - Wireshark & Tcpdump: Phân tích bắt gói tin và đối soát Transaction ID/UDP Port.

### DMZ Authoritative DNS Simulator (APP-TEST / AUTH-DNS)

- **Tên máy:** `APP-TEST` (Kế thừa LAB 8)
- **Network Interface:** Internal Network `lab-dmz`
  - IP tĩnh: `172.16.6.20/24`
- **Dịch vụ:** Script Python `auth_dns_delayed.py` đóng vai trò Authoritative DNS trả lời `www.external.lab9.test.` -> `172.16.6.10`, cấu hình độ trễ nhân tạo `--delay 1.5` giây để tạo điều kiện quan sát cuộc đua DNS (Race Condition).

### DMZ Web Server (DMZ-WEB)

- **Tên máy:** `DMZ-WEB` (Kế thừa LAB 7/8)
- **Network Interface:** Internal Network `lab-dmz`
  - IP tĩnh: `172.16.6.10/24`
- **Dịch vụ:** Web server nội bộ hợp lệ cho tên miền `www.external.lab9.test`.

---

## 3. Mô hình mạng (Topology)

```text
                                  INTERNET
                                      │
                                      ▼
                            ┌───────────────────┐
                            │   WAN (DHCP/NAT)  │
┌──────────────────┐        ├───────────────────┤
│ Máy thật Admin   │◄──────►│ MGMT 192.168.56.1 │ (pfSense: 192.168.56.254)
└──────────────────┘        ├───────────────────┤
                            │  pfSense - LAB 9  │
                            └─────────┬─────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
   LAN 10.6.0.1/24             DMZ 172.16.6.1/24          DNSLAB 10.9.0.1/24
(Internal: lab-lan)          (Internal: lab-dmz)         (Internal: dns-lab)
   [Cô lập an toàn]                   │                           │
                              ┌───────┴───────┐           ┌───────┼───────┐
                              ▼               ▼           ▼       ▼       ▼
                       ┌─────────────┐ ┌─────────────┐ ┌──────┐┌──────┐┌────────┐
                       │   DMZ-WEB   │ │  AUTH-DNS   │ │ DNS- ││VICTIM││ATTACKER│
                       │             │ │  (APP-TEST) │ │SERVER││Ubuntu││ (Kali) │
                       │172.16.6.10  │ │172.16.6.20  │ │.0.10 ││.0.100││ .0.200 │
                       └─────────────┘ └─────────────┘ └──────┘└──────┘└────────┘
```
