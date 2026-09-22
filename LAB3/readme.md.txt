# LAB 3 - NHẬN DIỆN VÀ ỨNG PHÓ CÁC MỐI ĐE DỌA ĐẾN AN TOÀN THÔNG TIN

## Thông tin sinh viên

- Họ và tên: [Điền họ tên]
- MSSV: [Điền MSSV]
- Lớp: [Điền mã lớp]
- Tên bài lab: Lab 3 - Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin

## Môi trường thực hành

- Nền tảng ảo hóa: VMware Workstation Pro
- Hệ điều hành: Windows 11 Pro 25H2
- OS Build thực tế: 26200.8037
- Python: 3.14.7
- Wireshark: 4.6.8
- Npcap: đã cài để hỗ trợ packet capture
- Sysmon: 15.22
- Autoruns: 14.3
- Process Explorer: 17.14
- Microsoft Defender Antivirus: bật trong quá trình thực hành
- Windows Firewall: bật trong quá trình thực hành
- Mạng VM: Host-only là cấu hình chính; NAT được sử dụng tạm thời khi cần tải công cụ hoặc tạo lưu lượng HTTPS.

## Cách dựng môi trường

Windows 11 Pro được cài trên VMware Workstation. Máy ảo được cấu hình tài nguyên theo yêu cầu bài lab và sử dụng mạng Host-only để giới hạn phạm vi thực hành.

Python, Wireshark/Npcap và bộ công cụ Sysinternals được cài để phục vụ việc thu thập và phân tích bằng chứng.

Các tình huống đã thực hiện
TH1 - Xác định tài sản, lỗ hổng, mối đe dọa và rủi ro

Đã xây dựng risk register theo quan hệ:

Asset → Vulnerability → Threat → Risk → Control

Đồng thời phân loại các nguồn đe dọa thành hành động vô ý, hành động cố ý, thảm họa/tác động môi trường, lỗi kỹ thuật và lỗi quản lý.

Kết quả: PASS

TH2 - EICAR và Microsoft Defender

Đã kiểm tra trạng thái Microsoft Defender và xác nhận Real-time Protection đang bật.

Tệp kiểm thử EICAR được tạo trong thư mục Evidence. Defender đã phát hiện:

Virus:DOS/EICAR_Test_File

và thực hiện cách ly thành công.

Kết quả: PASS

TH3 - Tấn công mật khẩu và nguy cơ keylogging

Đã bật Audit Logon Success/Failure và tạo tài khoản thử nghiệm:

lab3user

Đã thực hiện đăng nhập thành công và các lần đăng nhập sai có kiểm soát. Security Log ghi nhận các Event ID:

4624: đăng nhập thành công
4625: đăng nhập thất bại
4648: sử dụng credential rõ ràng

Đã thực hiện thay đổi credential của tài khoản thử nghiệm.

Kết quả: PASS

TH4 - Backdoor, Persistence và Listener

Đã cài Sysmon và kiểm tra log trong:

Microsoft-Windows-Sysmon/Operational

Đã thực hiện thử nghiệm persistence lành tính bằng Run value và Scheduled Task có tên LAB3_*.

Đã sử dụng Sysmon/Autoruns để kiểm tra dấu vết persistence.

Trạng thái: [PASS/PARTIAL - cập nhật theo bằng chứng H6, H7, H8 thực tế]

TH5 - Sniffing, MITM và Spoofing

Đã sử dụng Wireshark và Npcap để capture lưu lượng trong VM.

HTTP server cục bộ được chạy tại:

127.0.0.1:8080

HTTP được dùng để quan sát dữ liệu plaintext và HTTPS/TLS được dùng để so sánh khả năng quan sát nội dung.

Không thực hiện ARP poisoning, DNS spoofing, session hijacking, Wi-Fi giả mạo hoặc chèn chứng chỉ.

Kết quả: [PASS/FAIL - cập nhật theo capture thực tế]

TH6 - DoS, DDoS và Mail Bombing

Đã sử dụng local_load_test.py để tạo tải giới hạn trên:

127.0.0.1:8080

Script sử dụng 50 request và 5 worker, không tạo traffic gây tải tới hệ thống bên ngoài.

Đã phân tích ddos_sample.csv để quan sát nhiều SourceIP và phân tích mailbomb_sample.csv để thống kê sender và volume email.

Kết quả: [PASS/FAIL - cập nhật theo output thực tế]

TH7 - Social Engineering, Phishing và Spear Phishing

Đã phân tích offline file:

phishing_email.txt

Các dấu hiệu chính gồm cảm giác khẩn cấp, display name đáng tin giả, domain cần xác minh, Reply-To khác From và yêu cầu cung cấp thông tin xác thực.

Đã phân loại các tình huống trong social_engineering_cases.csv gồm Phishing, Spear Phishing, Watering Hole, Pretexting, Baiting và Quid Pro Quo.

Kết quả: [PASS/FAIL - cập nhật theo kết quả thực tế]

Lỗi gặp phải và cách khắc phục
1. Chạy lệnh PowerShell trong Command Prompt

Ban đầu các lệnh như New-Item, Get-Date và Out-File được chạy trong CMD nên báo lỗi.

Cách khắc phục: mở Windows PowerShell bằng quyền Administrator và chạy lại các lệnh.

2. Gói LAB3 giải nén sai cấp thư mục

Sau khi giải nén, lab3_assets nằm tại:

C:\LAB3\LAB3_Threats_Assets\lab3_assets

trong khi các lệnh trong bài sử dụng:

C:\LAB3\lab3_assets

Cách khắc phục: copy thư mục lab3_assets về đúng vị trí C:\LAB3\lab3_assets.

3. SHA-256 của gói tải về khác manifest trong tài liệu

Hash thực tế của gói tải về không trùng với giá trị trong tài liệu.

Cách xử lý: ghi nhận giá trị thực tế và không khẳng định file trùng manifest nếu chưa được xác minh.

4. runas không hoạt động với .\lab3user

Lệnh ban đầu báo lỗi khi lấy password hoặc xác thực.

Cách khắc phục: sử dụng tên máy đầy đủ:

runas /user:%COMPUTERNAME%\lab3user cmd.exe

Sau đó đăng nhập thành công và tạo được các event xác thực cần thiết.

5. Không tìm thấy Sysmon trong Event Viewer

Ban đầu không thấy nhánh Sysmon trong Event Viewer.

Cách khắc phục: kiểm tra lại cài đặt Sysmon, khởi động lại VM và mở đúng đường dẫn:

Applications and Services Logs → Microsoft → Windows → Sysmon → Operational

6. Wireshark không hiển thị interface capture

Wireshark báo không có packet capture driver.

Cách khắc phục: cài Npcap, sau đó mở lại Wireshark để sử dụng interface loopback.

Bằng chứng và log

Các output/log được lưu trong thư mục Evidence. Trước khi đưa lên GitHub, các file được kiểm tra và làm sạch để không chứa mật khẩu, token, cookie/session, email thật hoặc dữ liệu nhạy cảm.

File evidence_sha256.csv được tạo sau khi hoàn tất việc làm sạch các file Evidence để kiểm tra tính toàn vẹn.

Quy tắc an toàn khi thực hành

Không tắt Microsoft Defender để chạy mẫu kiểm thử.

Không phục hồi file EICAR đã bị quarantine để đưa lên repository.

Không chỉnh local_load_test.py sang mục tiêu khác ngoài 127.0.0.1:8080.

Không thực hiện DDoS, mail bombing, spoofing hoặc MITM chủ động trên mạng bên ngoài môi trường lab.

Không đưa installer, executable Sysinternals, Wireshark, Python hoặc dữ liệu nhạy cảm lên GitHub.