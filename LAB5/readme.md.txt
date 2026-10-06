## 1. Thông tin sinh viên

- Họ tên: Nguyễn Anh Khôi
- MSSV: 1150080141
- Tên lab: Lab 5 -  Thiết lập Mô hình Tường lửa pfSense

# Báo cáo Bài Lab:

## 2. Phiên bản môi trường
* **Hệ thống ảo hóa:** Oracle VirtualBox / VMware Workstation
* **Tường lửa:** pfSense Community Edition (CE) 2.7.2-RELEASE (amd64)
* **Máy trạm LAN:** Windows Server (Domain Controller - IP: 10.0.0.2) và Ubuntu Server (LAN-Test - IP: 10.0.0.3)
* **Máy chủ DMZ:** Windows Server (DMZ-Web - IP: 172.16.0.2)

## 3. Cách dựng môi trường
* **Mô hình mạng:** 
  * Cổng **WAN**: Cấu hình chế độ Bridged Adapter hoặc NAT Network riêng biệt (không chồng lấn), nhận IP tự động từ mạng ngoài.
  * Cổng **LAN (10.0.0.1/8)**: Gắn vào card Host-only, cấp địa chỉ tĩnh cho Domain Controller (10.0.0.2) và Ubuntu LAN-Test (10.0.0.3).
  * Cổng **DMZ (172.16.0.1/16)**: Gắn vào Internal Network (`dmz-net`), đặt địa chỉ tĩnh cho máy chủ DMZ-Web (172.16.0.2).
* **Cấu hình ban đầu:** Truy cập WebGUI qua `https://10.0.0.1`, thiết lập các Interface, cấu hình Outbound NAT ở chế độ Hybrid và chuẩn hóa ruleset nền tảng.

## 4. Các tình huống đã thực hiện & Kết quả
1. **Tình huống 1 — Chặn ICMP nhưng vẫn cho phép Web/DNS:** 
   * *Kết quả:* **PASS** (Ping ra 8.8.8.8 thất bại, phân giải DNS và truy cập HTTPS/HTTP thành công).
2. **Tình huống 2 — Chỉ cho một host cụ thể ra Internet:** 
   * *Kết quả:* **PASS** (Máy Domain Controller 10.0.0.2 ra được Internet, máy Ubuntu 10.0.0.3 bị chặn hoàn toàn).
3. **Tình huống 3 — Cô lập DMZ khỏi LAN:** 
   * *Kết quả:* **PASS** (DMZ-Web không thể ping vào Domain Controller trong LAN nhưng vẫn kết nối được Internet).
4. **Tình huống 4 — Port Forward WAN → DMZ:** 
   * *Kết quả:* **PASS** (Truy cập dịch vụ IIS trên máy DMZ-Web từ cổng 8080 của WAN thành công).
5. **Tình huống 5 — Bật logging và đọc Firewall Log:** 
   * *Kết quả:* **PASS** (Xác định chính xác các bản ghi chặn gói tin trên hệ thống log của tường lửa).

## 5. Lỗi gặp phải và cách khắc phục
* **Lỗi 1 (Mạng LAN):** Máy Ubuntu ban đầu không nhận IP tĩnh do có xung đột giữa 2 file cấu hình Netplan (`00-installer-config.yaml` và `50-cloud-init.yaml`).
  * *Cách khắc phục:* Vô hiệu hóa file cấu hình cũ (`.bak`), chỉ giữ lại một file `50-cloud-init.yaml`, chỉnh lại đúng tên card mạng thực tế (`ens33`) và dùng khoảng trắng (Space) chuẩn cú pháp YAML.
* **Lỗi 2 (Tình huống kiểm thử stateful firewall):** Các kết nối cũ vẫn thông sau khi bật rule Block do tường lửa lưu trạng thái (State).
  * *Cách khắc phục:* Thực hiện thao tác xóa bảng trạng thái kết nối tại `Diagnostics → States → Reset States` sau mỗi lần thay đổi rule.