\# LAB3 - CÁC MỐI ĐE DỌA AN TOÀN THÔNG TIN



\## 1. Thông tin sinh viên



\- Họ và tên: Bùi Đức Anh

\- MSSV: 1150080126

\- Mã lớp: 11\_THMT

\- Tên bài Lab: LAB3 - Các mối đe dọa An toàn thông tin



\## 2. Môi trường thực hành



\- Máy ảo: VMware Workstation

\- Hệ điều hành: Windows 11 25H2 x64

\- OS Build: 26200.9445

\- RAM: 6 GB trở lên

\- CPU: 2 vCPU

\- Network Adapter: Host-only

\- Windows Defender: Bật Real-time Protection và Tamper Protection

\- PowerShell: 5.1

\- Sysmon: 15.22

\- Autoruns: 14.3

\- Process Explorer: 17.14

\- Wireshark: 4.6.8

\- Python: 3.14.7



\## 3. Link video thực hành



\- Video: \[BỔ SUNG LINK VIDEO TẠI ĐÂY]



\## 4. Nội dung đã thực hiện



\### TH1 - Nhận diện và phân loại mối đe dọa



\- Thực hiện phân tích và phân loại các mối đe dọa.

\- Xây dựng Risk Register.

\- Phân loại các mối đe dọa theo nội dung yêu cầu của bài Lab.

\- Hoàn thành phần trả lời TH1.



Kết quả: PASS



File bằng chứng:

\- TH1\_Answers.txt

\- TH1\_Risk\_Register.txt

\- TH1\_Threat\_Classification.txt



\### TH2 - Kiểm tra khả năng phát hiện của Windows Defender



\- Kiểm tra trạng thái Windows Defender.

\- Real-time Protection được duy trì ở trạng thái bật.

\- Tamper Protection được duy trì ở trạng thái bật.

\- Sử dụng file kiểm thử EICAR trong môi trường Lab.

\- Windows Defender phát hiện file kiểm thử với tên mối đe dọa Virus:DOS/EICAR\_Test\_File và thực hiện cách ly.

\- Không tắt Defender và không tạo exclusion cho file kiểm thử.



Kết quả: PASS



File bằng chứng:

\- Các kết quả liên quan được lưu trong thư mục TH2 trên máy ảo.

\- Không upload file EICAR/quarantine lên repository.



\### TH3 - Kiểm tra xác thực và ghi log đăng nhập



\- Bật Audit Logon cho cả Success và Failure.

\- Tạo tài khoản kiểm thử `lab3user`.

\- Thực hiện đăng nhập thành công.

\- Thực hiện các lần đăng nhập sai có kiểm soát.

\- Kiểm tra các sự kiện Security Event Log.

\- Ghi nhận các Event ID 4624, 4625 và 4648.

\- Thay đổi mật khẩu tài khoản kiểm thử và kiểm tra lại.

\- Xóa tài khoản kiểm thử sau khi hoàn thành.



Kết quả: PASS



File bằng chứng:

\- auth\_events\_before\_rotation.txt

\- TH3\_auth\_events.txt



\### TH4 - Phát hiện persistence và lưu lượng localhost



\- Cài đặt và kiểm tra Sysmon.

\- Kiểm tra Sysmon Event ID 1 thông qua Event Viewer.

\- Tạo một Run key benign có tên `LAB3\_Run\_Demo`.

\- Tạo Scheduled Task benign có tên `LAB3\_Persistence\_Demo`.

\- Kiểm tra các persistence artifact bằng Autoruns.

\- Kiểm tra tiến trình bằng Process Explorer.

\- Tạo HTTP listener chỉ trên `127.0.0.1:8080`.

\- Kiểm tra port 8080 đang LISTEN.

\- Kiểm tra kết nối localhost.

\- Sử dụng Wireshark để quan sát lưu lượng localhost trên port 8080.

\- Sau khi hoàn thành, dừng HTTP listener và xóa các persistence artifact.

\- Kiểm tra lại để bảo đảm các artifact LAB3 đã được dọn dẹp.



Kết quả: PASS



File bằng chứng:

\- autoruns\_before.csv

\- autoruns\_th4.csv

\- autoruns\_after.csv

\- task\_ran.txt

\- H10\_Localhost\_8080.pcapng



\## 5. Evidence và log



Các file bằng chứng và log được lưu trong thư mục LAB3 của repository.



Các nhóm file gồm:



\- Baseline hệ thống:

&#x20; - baseline\_defender.txt

&#x20; - baseline\_firewall.txt

&#x20; - baseline\_network.txt

&#x20; - baseline\_os.txt

&#x20; - baseline\_processes.txt

&#x20; - start\_time.txt



\- Authentication:

&#x20; - auth\_events\_before\_rotation.txt

&#x20; - TH3\_auth\_events.txt



\- Sysmon/Autoruns:

&#x20; - autoruns\_before.csv

&#x20; - autoruns\_th4.csv

&#x20; - autoruns\_after.csv



\- TH1:

&#x20; - TH1\_Answers.txt

&#x20; - TH1\_Risk\_Register.txt

&#x20; - TH1\_Threat\_Classification.txt



\- TH4:

&#x20; - task\_ran.txt

&#x20; - H10\_Localhost\_8080.pcapng



\- Kiểm tra toàn vẹn:

&#x20; - evidence\_sha256.csv



\## 6. Kết quả tổng hợp



| Nội dung | Kết quả |

|---|---|

| TH1 - Nhận diện và phân loại mối đe dọa | PASS |

| TH2 - Kiểm tra Windows Defender | PASS |

| TH3 - Authentication và Event Log | PASS |

| TH4 - Sysmon, Autoruns, Process Explorer và localhost | PASS |

| Cleanup sau thực hành | PASS |



\## 7. Lỗi và cách khắc phục



\- Trong quá trình thực hành có kiểm tra và xử lý các vấn đề liên quan đến công cụ, log và môi trường thực hành.

\- Các thao tác được thực hiện trong máy ảo Windows 11.

\- Sau khi hoàn thành TH4, các persistence artifact và HTTP listener localhost được dừng/xóa.

\- Windows Defender được duy trì ở trạng thái bật trong quá trình thực hành.



\## 8. Lưu ý kiểm tra lại



\- Repository chỉ chứa các file kết quả và bằng chứng cần thiết của LAB3.

\- Không upload bộ cài Sysmon, Autoruns, Process Explorer, Wireshark hoặc Python.

\- Không upload file Defender quarantine hoặc file EICAR kiểm thử.

\- Không chứa mật khẩu, token, cookie hoặc thông tin xác thực.

\- File `evidence\_sha256.csv` chứa SHA-256 của các file bằng chứng được nộp.

