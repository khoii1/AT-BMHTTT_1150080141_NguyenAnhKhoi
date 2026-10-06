# LAB 4 - KHẢO SÁT VÀ ĐÁNH GIÁ BỀ MẶT MẠNG BẰNG NMAP

## 1. Thông tin sinh viên

- Họ tên: Nguyễn Anh Khôi
- MSSV: 1150080141
- Tên lab: Lab 4 - Khảo sát và đánh giá bề mặt mạng bằng Nmap

## 2. Phiên bản và môi trường thực hành

- Phần mềm ảo hóa: VMware Workstation
- Kali Linux:
  - IP: 172.16.16.130/24
  - Vai trò: Máy quét
  - Nmap: 7.99
- Metasploitable 2:
  - IP: 172.16.16.129/24
  - Vai trò: Máy đích
- Windows 11:
  - IP: 172.16.16.128/24
  - Vai trò: Máy đích dùng kiểm tra before/after hardening
  - Windows build: 10.0.26200.8037
- Mạng sử dụng: VMware Host-only
- Subnet: 172.16.16.0/24

## 3. Cách dựng môi trường

1. Tạo máy ảo Kali Linux trên VMware và cài Nmap.
2. Import máy ảo Metasploitable 2 vào VMware.
3. Chuyển Kali, Metasploitable 2 và Windows 11 sang cùng mạng Host-only.
4. Kiểm tra địa chỉ IP:
   - Kali: `ip -br addr`
   - Metasploitable 2: `ifconfig`
   - Windows 11: `ipconfig`
5. Kiểm tra kết nối từ Kali đến Metasploitable 2 bằng lệnh:

   `ping -c 4 172.16.16.129`

6. Chỉ thực hiện quét trong mạng Host-only, không quét các hệ thống bên ngoài.

## 4. Các tình huống đã thực hiện

### 4.1. Host Discovery

Lệnh:

`sudo nmap -sn 172.16.16.0/24`

Kết quả phát hiện 4 host đang hoạt động trong mạng, bao gồm Kali Linux và Metasploitable 2.

**Kết quả: PASS**

### 4.2. TCP Connect Scan

Lệnh:

`nmap -sT 172.16.16.129`

Kết quả:
- 23 cổng open
- 977 cổng closed
- 0 cổng filtered

**Kết quả: PASS**

### 4.3. SYN Scan

Lệnh:

`sudo nmap -sS 172.16.16.129`

Kết quả:
- 23 cổng open
- 977 cổng closed
- 0 cổng filtered

Kết quả gần giống TCP Connect Scan.

**Kết quả: PASS**

### 4.4. FIN / Xmas / NULL Scan

Các lệnh:

`sudo nmap -sF 172.16.16.129`

`sudo nmap -sX 172.16.16.129`

`sudo nmap -sN 172.16.16.129`

Các kỹ thuật đều ghi nhận 23 cổng ở trạng thái `open|filtered` và 977 cổng đóng.

**Kết quả: PASS**

### 4.5. ACK Scan

Lệnh:

`sudo nmap -sA 172.16.16.129`

Kết quả cho thấy 1000 cổng ở trạng thái `unfiltered`.

**Kết quả: PASS**

### 4.6. UDP Scan

Lệnh:

`sudo nmap -sU --top-ports 20 172.16.16.129`

Phát hiện một số dịch vụ UDP như:
- 53/udp - domain
- 137/udp - netbios-ns

Một số cổng có trạng thái `open|filtered`.

**Kết quả: PASS**

### 4.7. Nhận diện dịch vụ và phiên bản

Lệnh:

`sudo nmap -sV 172.16.16.129`

Một số dịch vụ phát hiện được:
- FTP: vsftpd 2.3.4
- SSH: OpenSSH 4.7p1
- HTTP: Apache httpd 2.2.8
- MySQL: MySQL 5.0.51a
- SMB: Samba

**Kết quả: PASS**

### 4.8. Nhận diện hệ điều hành

Lệnh:

`sudo nmap -O 172.16.16.129`

Nmap dự đoán máy đích chạy Linux 2.6.X, khoảng phiên bản Linux 2.6.9 - 2.6.33.

**Kết quả: PASS**

### 4.9. Aggressive Scan

Lệnh:

`sudo nmap -A 172.16.16.129`

Kết quả thu được thông tin về dịch vụ, phiên bản, hệ điều hành, traceroute và các script mặc định.

**Kết quả: PASS**

### 4.10. Thu thập thông tin SMB

Lệnh:

`sudo nmap -p 445 --script smb-os-discovery 172.16.16.129`

Kết quả:
- OS: Unix (Samba 3.0.20-Debian)
- Computer name: metasploitable
- Domain: localdomain
- FQDN: metasploitable.localdomain

**Kết quả: PASS**

### 4.11. Kiểm tra MS17-010

Lệnh:

`sudo nmap -p 445 --script smb-vuln-ms17-010 172.16.16.129`

Cổng 445/tcp đang mở nhưng script không trả về trạng thái `VULNERABLE`.

Không đủ cơ sở để kết luận máy đích bị ảnh hưởng bởi MS17-010.

**Kết quả: PASS - không xác định được lỗ hổng**

### 4.12. Xuất kết quả

Đã xuất kết quả dưới các định dạng:

- `ket_qua.txt`
- `ket_qua.xml`
- `smb.txt`
- `bao_cao.html`

Các tùy chọn sử dụng:
- `-oN`
- `-oX`
- `-oG`
- `xsltproc`

**Kết quả: PASS**

### 4.13. Before/After Hardening

Máy đích: Windows 11 - 172.16.16.128

#### Before

Tắt Windows Defender Firewall và chạy:

`sudo nmap -sV 172.16.16.128 -oN before.txt`

Kết quả phát hiện:
- 135/tcp open - msrpc
- 139/tcp open - netbios-ssn
- 445/tcp open - microsoft-ds

#### Hardening

Bật lại Windows Defender Firewall.

#### After

Chạy lại:

`sudo nmap -sV 172.16.16.128 -oN after.txt`

Kết quả:
- 0 cổng open
- 1000 cổng filtered

Windows Defender Firewall đã hạn chế các kết nối từ máy Kali đến Windows 11.

**Kết quả: PASS**

## 5. Lỗi gặp phải và cách khắc phục

### Lỗi 1: Kali và Metasploitable 2 không cùng subnet

Ban đầu Kali nhận IP `192.168.231.132/24`, trong khi Metasploitable 2 có IP `172.16.16.129/24`.

**Cách khắc phục:**
Chuyển Network Adapter của cả hai máy sang cùng mạng VMware Host-only. Sau đó Kali nhận IP `172.16.16.130/24`.

### Lỗi 2: Lệnh `ipconfig` không chạy trên Metasploitable 2

Hệ thống báo:

`-bash: ipconfig: command not found`

**Cách khắc phục:**
Sử dụng lệnh Linux:

`ifconfig`

### Lỗi 3: Windows 11 không phản hồi ping

Ping từ Kali đến Windows 11 bị mất 100% gói tin và Nmap báo toàn bộ cổng `filtered`.

**Nguyên nhân:**
Windows Defender Firewall đang chặn lưu lượng từ mạng.

**Cách khắc phục:**
Tạm tắt Windows Defender Firewall để lấy kết quả Before. Sau đó bật lại firewall để thực hiện và chứng minh kết quả hardening.

### Lỗi 4: Script MS17-010 không trả về kết luận

Script `smb-vuln-ms17-010` không báo `VULNERABLE`.

**Cách xử lý:**
Không tự kết luận máy đã an toàn hoặc đã vá. Chỉ ghi nhận rằng script chưa cung cấp đủ thông tin để xác định lỗ hổng.

## 6. Kết luận

Môi trường thực hành đã được dựng thành công trên VMware Host-only. Các kỹ thuật quét Nmap trong nội dung chính của Lab 4 đã được thực hiện đầy đủ.

Kết quả cho thấy Metasploitable 2 mở nhiều dịch vụ mạng và cung cấp nhiều thông tin khi được quét bằng Nmap. Thử nghiệm before/after trên Windows 11 cũng cho thấy firewall có tác dụng hạn chế khả năng truy cập từ các máy khác trong mạng.

**Kết quả tổng thể: PASS**