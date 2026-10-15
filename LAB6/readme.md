# README – LAB 6: Xác thực Active Directory và chính sách truy cập trên pfSense

## 1. Thông tin sinh viên

- **Họ và tên:** Anh Bùi Đức
- **MSSV:** `1150080126`
- **Tên bài lab:** LAB 6 – Xác thực Active Directory (LDAP), Captive Portal và cấu hình chính sách truy cập trên pfSense
- **Hình thức:** Thực hành trên máy ảo VMware Workstation
- **Phạm vi hoàn thành:** Kịch bản 1 đến kịch bản 5 (KB1–KB5)

## 2. Phiên bản và môi trường

| Thành phần | Phiên bản / vai trò | Ghi chú |
|---|---|---|
| Máy vật lý | Windows 11 | Máy chạy VMware và quản trị pfSense qua trình duyệt |
| Phần mềm ảo hóa | VMware Workstation | Chưa xác nhận số phiên bản cụ thể |
| Firewall | **pfSense CE 2.7.2** | 4 card mạng WAN, MGMT, LAN, DMZ |
| Domain Controller | **Windows Server 2025** | Active Directory Domain Services, DNS, LDAP |
| Máy kiểm thử | **Windows 11** | VM `LAB3-Windows11`, thay đổi IP để thử Client-A/B |
| Web Server DMZ | **Ubuntu Server** | Nginx; chưa xác nhận phiên bản Ubuntu cụ thể |
| Domain | `lab6.test` | DC01: `10.6.0.10` |
| LDAP bind | `svc_pfsense_ldap` | Tài khoản dịch vụ dùng cho xác thực LDAP |

### Sơ đồ địa chỉ mạng

| Interface / VMnet | Subnet | Địa chỉ chính |
|---|---|---|
| WAN / VMnet8 (NAT) | `192.168.250.0/24` | pfSense: `192.168.250.128` |
| MGMT / VMnet1 | `192.168.56.0/24` | pfSense: `192.168.56.254` |
| LAN / VMnet2 | `10.6.0.0/24` | pfSense: `10.6.0.1`; DC01: `10.6.0.10` |
| DMZ / VMnet3 | `172.16.6.0/24` | pfSense: `172.16.6.1`; Ubuntu: `172.16.6.10` |
| Client-A | LAN | `10.6.0.21` |
| Client-B | LAN | `10.6.0.22` |

## 3. Cách dựng môi trường

1. **Tạo mạng ảo trong VMware:** thiết lập VMnet8 ở chế độ NAT (WAN); VMnet1 cho MGMT; VMnet2 cho LAN; VMnet3 cho DMZ. Gán bốn card mạng tương ứng vào VM `pfSense-LAB6`.
2. **Cài pfSense CE 2.7.2:** cấu hình WAN `192.168.250.128`, MGMT `192.168.56.254`, LAN `10.6.0.1`, DMZ `172.16.6.1`. Quản trị WebGUI tại `https://192.168.56.254`.
3. **Cài DC01-LAB6:** Windows Server 2025 trên LAN, IP `10.6.0.10`, cài AD DS và DNS, tạo domain `lab6.test`, tài khoản `sv_a`, `sv_b` và `svc_pfsense_ldap`.
4. **Cài LAB3-Windows11:** nối VMnet2 (LAN), DNS trỏ về `10.6.0.10`, gateway `10.6.0.1`; lần lượt đổi IP thành `.21` và `.22` để thử các alias.
5. **Cài DMZ-WEB-LAB6:** Ubuntu Server nối VMnet3, IP `172.16.6.10/24`, gateway `172.16.6.1`; cài Nginx và bổ sung IP phụ `172.16.6.20`, `.30`, `.40` trên `ens33`.
6. **Tạo các website mô phỏng:** Nginx phục vụ `JOB_SITE`, `RESTRICTED_SITE`, `SOCIAL_SITE` bằng các IP riêng. Trong cấu hình đã kiểm thử, `JOB_SITE=.20`, `RESTRICTED_SITE=.30`, `SOCIAL_SITE=.40`.
7. **Kết nối pfSense với AD:** cấu hình LDAP Authentication Server trỏ về `10.6.0.10`; bật Captive Portal cho LAN, xác thực bằng tài khoản AD.
8. **Tạo Firewall Aliases:** `CLIENT_A=10.6.0.21`, `CLIENT_B=10.6.0.22`, `POLICY_CLIENTS={10.6.0.21,10.6.0.22}`, các alias website và `WEB_PORTS={80,443}`.
9. **Tạo Schedules:** `LAB6_BREAK_TIME` (thứ 2–6, 12:00–13:00); `LAB6_WORK_HOURS` (thứ 2–6, 08:00–12:00 và 13:00–17:00). Có thêm một khoảng thử nghiệm ngày 15/10/2026 cho KB4 rồi đã xóa sau kiểm thử.
10. **Tạo firewall rules:** cấu hình theo thứ tự từ trên xuống cho KB1–KB4 tại `Firewall > Rules > LAN`, KB5 tại `Firewall > Rules > DMZ`. Bật log trên các rule Block và nhấn **Apply Changes**.
11. **Kiểm thử:** sử dụng trình duyệt và `curl.exe`, xem `Status > System Logs > Firewall`; khi cần, kiểm tra/xóa state liên quan trong `Diagnostics > States` rồi tạo kết nối mới.

> **Chú ý:** Các IP phụ thêm bằng `sudo ip addr add ...` chỉ có hiệu lực tạm thời, sẽ mất sau khi Ubuntu khởi động lại nếu chưa cấu hình Netplan. Cần lưu cấu hình lâu dài trước khi bàn giao.

## 4. Các tình huống đã thực hiện và kết quả PASS/FAIL

**Quy ước:** `PASS` = có bằng chứng kết quả mong đợi; `PARTIAL` = đã cấu hình hoặc kiểm tra một phần nhưng thiếu bằng chứng cho toàn bộ yêu cầu; `FAIL` = kết quả trái mong đợi. Bảng dưới đây dựa trên ảnh và kết quả thực hành đã cung cấp, không phải phép thử mới.

| STT | Tình huống / yêu cầu | Kết quả | Bằng chứng / nhận xét |
|---|---|---|---|
| 0A | Xác thực AD/LDAP bằng tài khoản người dùng | **PARTIAL** | Đã cấu hình LDAP; có bằng chứng đăng nhập Captive Portal thành công với `sv_b`, chưa đối chiếu ảnh test riêng cho cả `sv_a` và `sv_b` trong Diagnostics > Authentication. |
| 0B | Captive Portal LAN | **PASS** | Giao diện hiển thị **“The portal session is connected.”** |
| 1 (KB1) | Chặn Client-A truy cập JOB_SITE (`172.16.6.20:80`) | **PASS** | Firewall Log hiển thị rule `LAB6_KB1_Block_ClientA_JobSite`, nguồn `10.6.0.21`, đích `172.16.6.20:80`, hành động Block. |
| 2 (KB2) | Chặn Client-A và Client-B truy cập RESTRICTED_SITE (`172.16.6.30:80`) | **PASS** | Firewall Log hiển thị rule `LAB6_KB2_Block_Clients_Restricted` cho cả `10.6.0.21` và `10.6.0.22`. |
| 3 (KB3) | Client-B được truy cập SOCIAL_SITE trong giờ nghỉ; ngoài giờ bị chặn | **PASS** | Đã thấy `HTTP/1.1 200 OK` khi được phép và log Block tới `172.16.6.40:80` ngoài giờ. Rule Pass theo `LAB6_BREAK_TIME` đặt trên rule Block. |
| 4 (KB4) | Chặn HTTP/HTTPS trong giờ làm việc theo rule thực hành | **PASS** | Log `LAB6_KB4_Block_Web_WorkHours` chặn TCP/80 và TCP/443 từ `10.6.0.21`; schedule `LAB6_WORK_HOURS` đã được cấu hình và thử. |
| 5 (KB5) | Ngăn máy DMZ chủ động truy cập mạng LAN | **PASS** | Firewall Log hiển thị `LAB6_KB5_Block_DMZ_to_LAN`, nguồn `172.16.6.10`, đích `10.6.0.x`, hành động Block. |

**Lưu ý phạm vi:** KB4 được triển khai theo biến thể **chặn HTTP/HTTPS trong giờ làm việc**. Nếu đề bài yêu cầu **chỉ cho phép HTTP/HTTPS trong giờ làm việc**, phải đổi thiết kế rule và kiểm thử lại; không được coi hai biến thể là tương đương. Các kịch bản KB6–KB7 không nằm trong phạm vi đã hoàn thành.

## 5. Lỗi gặp phải và cách khắc phục

| Lỗi / hiện tượng | Nguyên nhân hoặc cách xác định | Cách khắc phục / kết quả |
|---|---|---|
| Captive Portal báo không kết nối được Authentication Server | Máy DC01 bị tắt, LDAP không truy cập được | Bật lại `DC01-LAB6`, kiểm tra mạng tới `10.6.0.10`, sau đó xác thực lại. |
| Truy cập site trả `HTTP 302 Found` thay vì trang web | Captive Portal chuyển hướng client chưa có phiên đăng nhập | Mở trang đăng nhập Captive Portal, xác thực AD; xác nhận “The portal session is connected”, sau đó thử lại. |
| Trình duyệt báo timeout khi vào `10.6.0.1:8000` | Truy cập sai/không đúng cổng của trang Captive Portal trong lần thử đó | Sử dụng URL portal đã xác nhận hoạt động `http://10.6.0.1:8002/index.php?zone=lab6_users` hoặc kích hoạt chuyển hướng qua trang HTTP. |
| Ubuntu DMZ không phân giải được tên miền | DNS trỏ tới DC01 `10.6.0.10` nhưng truy vấn DNS không thành công | Tạm thời đặt DNS qua `sudo resolvectl dns ens33 1.1.1.1` và `sudo resolvectl domain ens33 '~.'`; xác minh kết nối Internet. |
| Website bị chặn dù đã sửa rule | Rule thứ tự sai, schedule chưa active, Captive Portal chưa đăng nhập hoặc state cũ | Kiểm tra thứ tự Pass/Block, giờ hệ thống, session portal; tạo kết nối mới hoặc xóa state liên quan rồi thử lại. |
| KB3 truy cập SOCIAL_SITE không được ngoài giờ | Đây là kết quả đúng của rule Block ngoài schedule | Xem Firewall Log, đối chiếu nguồn `10.6.0.21/.22`, đích `172.16.6.40:80`. |
| Khó nhập lệnh dài trong cửa sổ Ubuntu của VMware | Sao chép/dán vào console VMware bất tiện | Dùng SSH từ PowerShell máy thật: `ssh lab6admin@172.16.6.10` nếu máy thật kết nối được VMnet3. |
| Nhiều máy ảo chạy đồng thời làm máy thật chậm | Tài nguyên RAM hạn chế | Chỉ bật các VM cần cho bước kiểm thử; bật DC01 khi cần LDAP/Captive Portal. |
| Firewall Log có nhiều bản ghi, khó tìm kết quả | Chưa lọc theo IP nguồn, đích, cổng | Vào `Status > System Logs > Firewall`, dùng **Advanced Log Filter** và lọc theo từng kịch bản. |
| Sau khởi động lại, IP phụ của Ubuntu có thể mất | Các lệnh `ip addr add` không tạo cấu hình vĩnh viễn | Cần chuyển sang Netplan để lưu các IP phụ lâu dài. Đây là lưu ý vận hành, chưa xác nhận đã cấu hình Netplan. |

## 6. Một số lệnh kiểm thử quan trọng

### Trên Windows 11

```powershell
ipconfig
ipconfig /all
ping 10.6.0.1
curl.exe -v --noproxy "*" --connect-timeout 5 --max-time 10 http://172.16.6.10/
curl.exe -v --noproxy "*" --connect-timeout 5 --max-time 10 http://172.16.6.20/
curl.exe -v --noproxy "*" --connect-timeout 5 --max-time 10 http://172.16.6.30/
curl.exe -v --noproxy "*" --connect-timeout 5 --max-time 10 http://172.16.6.40/
```

### Trên Ubuntu DMZ

```bash
ip -4 addr show ens33
sudo ip addr add 172.16.6.20/24 dev ens33
sudo ip addr add 172.16.6.30/24 dev ens33
sudo ip addr add 172.16.6.40/24 dev ens33
sudo nginx -t
sudo systemctl reload nginx
curl http://172.16.6.20
curl http://172.16.6.30
curl http://172.16.6.40
```

## 7. Minh chứng cần nộp

- Ảnh cấu hình mạng VMware và các interface pfSense.
- Ảnh cấu hình LDAP và Captive Portal; ảnh phiên portal kết nối thành công.
- Ảnh Aliases (`CLIENT_A`, `CLIENT_B`, `POLICY_CLIENTS`, các site, `WEB_PORTS`).
- Ảnh Schedules `LAB6_BREAK_TIME`, `LAB6_WORK_HOURS`.
- Ảnh Rules LAN và DMZ cho KB1–KB5.
- Ảnh `curl.exe` thể hiện HTTP 200, HTTP 302 hoặc timeout đúng tình huống.
- Ảnh Firewall Log xác nhận Block KB1–KB5; ảnh State Table nếu có.

## 8. Kết luận

Môi trường LAB 6 đã được triển khai trên VMware với pfSense CE 2.7.2, Active Directory, LDAP, Captive Portal và Nginx DMZ. Năm kịch bản firewall KB1–KB5 đã được cấu hình và có bằng chứng kiểm thử tương ứng. Phần xác thực LDAP riêng cho từng tài khoản cần đối chiếu bổ sung nếu giáo viên yêu cầu minh chứng độc lập. Hệ thống mô phỏng lọc website bằng **địa chỉ IP và cổng**, không phải công cụ phân loại nội dung web thực tế.

---
