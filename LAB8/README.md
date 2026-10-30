LAB8 — Local DNS Attack: DNS Spoofing và Cache Poisoning trong mạng ảo cô lập
- **Sinh viên thực hiện:** Bùi Đức Anh
- **Môi trường triển khai:** VMware Workstation Pro, pfSense Firewall CE 2.7.2, Ubuntu Server 24.04, Kali Linux 2026.

---

## 1. Giới thiệu tổng quan

Bài thực hành này nghiên cứu chuyên sâu về các kỹ thuật tấn công dịch vụ phân giải tên miền (DNS) trong môi trường mạng cục bộ được bảo vệ bởi tường lửa **pfSense**, bao gồm:
1. **Client-side Host File Tampering:** Can thiệp file cấu hình tĩnh `/etc/hosts` và phân tích thứ tự ưu tiên của trình phân giải hệ thống (NSSwitch).
2. **Direct DNS Response Spoofing (On-path / Local Spoofing):** Giả mạo gói tin DNS Response trả lời trực tiếp cho máy trạm (Victim) nhanh hơn máy chủ DNS thật.
3. **DNS Resolver Cache Poisoning (Đầu độc bộ nhớ đệm BIND9):** Tấn công chu trình phân giải đệ quy (Recursive Query) giữa Resolver và Authoritative Server, cấy bản ghi giả mạo vào bộ đệm của DNS Server nội bộ.

Toàn bộ quá trình thực nghiệm được triển khai trong phân vùng mạng cô lập an toàn (**VMnet4 - DNSLAB**), không định tuyến ra Internet công cộng, tuân thủ nghiêm ngặt các nguyên tắc đạo đức và an toàn thông tin.

---

## 2. Sơ đồ mạng & Bảng địa chỉ IP (Network Architecture)

```
                              [ INTERNET / WAN ]
                                      |
                            +--------------------+
                            |    pfSense-LAB6    | (Firewall / Gateway)
                            |  192.168.56.254    | (MGMT: VMnet1)
                            +---------+----------+
                                      |
            +-------------------------+-------------------------+
            | (VMnet3: DMZ)                                     | (VMnet4: DNSLAB - Cô lập)
            | 172.16.6.1/24                                     | 10.9.0.1/24
            |                                                   |
  +---------+-----------+                         +-------------+-------------+
  |    DMZ-WEB-LAB6     |                         |                           |
  | IP: 172.16.6.10     | (Nginx Real Web)        |                           |
  | IP: 172.16.6.20:53  | (Auth-DNS Delayed)      |                           |
  +---------------------+                         |                           |
                                                  |                           |
                    +-----------------------------+---------------------------+
                    |                             |                           |
          +---------+---------+         +---------+---------+       +---------+---------+
          |  DNS-SERVER-LAB8  |         |   ATTACKER-LAB8   |       |    VICTIM-LAB8    |
          |  10.9.0.10/24     |         |   10.9.0.30/24    |       |   10.9.0.100/24   |
          |  BIND9 Resolver   |         | Alias: 10.9.0.200 |       |   Ubuntu 24.04    |
          |  Zone: lab9.test  |         | Kali / Spoofer    |       |   Client / Victim |
          +-------------------+         +-------------------+       +-------------------+
```

### Bảng phân hoạch địa chỉ IP chi tiết:

| Tên máy ảo | Hệ điều hành | Card mạng | IP Address / Subnet | Vai trò trong kịch bản |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense-LAB6** | FreeBSD 14 | VMnet4 (`opt3`) | `10.9.0.1/24` | Gateway quản lý DNSLAB, kiểm soát Firewall Rules |
| **pfSense-LAB6** | FreeBSD 14 | VMnet3 (`opt1`) | `172.16.6.1/24` | Gateway phân vùng DMZ |
| **DNS-SERVER-LAB8** | Ubuntu 24.04 | VMnet4 | `10.9.0.10/24` | Máy chủ DNS đệ quy nội bộ (BIND 9.18), Forwarder |
| **ATTACKER-LAB8** | Kali Linux | VMnet4 | `10.9.0.30/24`<br>`10.9.0.200/24` (alias) | Kẻ tấn công; Web server giả mạo (port 80); chạy công cụ Sniff & Spoof Scapy |
| **VICTIM-LAB8** | Ubuntu 24.04 | VMnet4 | `10.9.0.100/24` | Máy trạm nạn nhân, cấu hình DNS trỏ về `10.9.0.10` |
| **DMZ-WEB-LAB6** | Ubuntu Server | VMnet3 | `172.16.6.10/24`<br>`172.16.6.20/24` (alias) | `172.16.6.10`: Nginx Web Server thật.<br>`172.16.6.20`: Authoritative DNS cho `external.lab9.test` |

---

## 3. Chính sách Tường lửa pfSense (Firewall Rules Matrix)

Trên giao diện pfSense WebGUI (`https://192.168.56.254`) tại tab **Firewall > Rules > DNSLAB**, bộ luật cô lập được cấu hình như sau:

| STT | Action | Protocol | Source | Destination | Port | Mục đích / Giải thích |
| :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **1** | `PASS` | ICMP | `10.9.0.30` | DNSLAB address | Any | Kiểm tra kết nối ICMP cơ bản cho máy Kali |
| **2** | `PASS` | ICMP | `10.9.0.10` | `10.9.0.1` | Any | Kiểm tra kết nối gateway từ DNS Server |
| **3** | `PASS` | UDP/TCP | `10.9.0.10` | `172.16.6.20` | `53 (DNS)` | Cho phép DNS Server truy vấn đệ quy sang Auth-DNS ở DMZ |
| **4** | `PASS` | TCP | `10.9.0.0/24`| `172.16.6.10` | `80 (HTTP)`| Cho phép client trong DNSLAB truy cập Web thật Nginx |
| **5** | `BLOCK`| Any | `10.9.0.0/24`| `10.6.0.0/24` | Any | Cách ly hoàn toàn DNSLAB với mạng LAN |
| **6** | `BLOCK`| Any | `10.9.0.0/24`| `192.168.56.0/24` | Any | Ngăn chặn truy cập vùng Quản trị (MGMT) |
| **7** | `BLOCK`| Any | `10.9.0.0/24`| Any | Any | **Default Block:** Cách ly tuyệt đối, không cho ra WAN |

---

## 4. Các kịch bản thực nghiệm & Kết quả chi tiết

### Baseline: Trạng thái chuẩn trước tấn công
- **Mục tiêu:** Xác minh dịch vụ DNS và HTTP hoạt động đúng quy chuẩn khi chưa bị can thiệp.
- **Thực thi trên VICTIM (`10.9.0.100`):**
  ```bash
  # Phân giải tên miền qua DNS-SERVER nội bộ:
  dig @10.9.0.10 www.external.lab9.test +short
  # Kết quả trả về: 172.16.6.10 (IP thật)

  # Truy cập dịch vụ Web:
  curl http://www.lab9.test
  # Kết quả trả về: Trang chào mừng Nginx thật (Welcome to nginx!)
  ```

---

### Task 1: Tấn công sửa đổi tệp tin phân giải cục bộ (`/etc/hosts`)
- **Nguyên lý:** Trình phân giải hệ thống Linux kiểm tra thứ tự qua `/etc/nsswitch.conf` (`hosts: files dns`). Tệp tin tĩnh `/etc/hosts` luôn được ưu tiên xử lý trước khi gửi truy vấn DNS ra mạng.
- **Thực thi:**
  1. Thêm bản ghi giả mạo vào `/etc/hosts` của Victim:
     ```bash
     echo "10.9.0.200    www.lab9.test" | sudo tee -a /etc/hosts
     ```
  2. Kiểm tra truy cập web:
     ```bash
     curl http://www.lab9.test
     ```
     👉 **Kết quả:** Trả về trang Web giả mạo của Attacker (`LAB9 FAKE SITE - 10.9.0.200`).
  3. Kiểm tra bằng công cụ `dig`:
     ```bash
     dig @10.9.0.10 www.lab9.test +short
     ```
     👉 **Kết quả:** Vẫn trả về IP thật `172.16.6.10`.
- **Nhận xét cốt lõi:** Lệnh `curl` sử dụng hàm `getaddrinfo()` của hệ thống nên tuân theo `/etc/hosts`, trong khi `dig` tạo socket UDP truy vấn trực tiếp máy chủ DNS nên không bị ảnh hưởng bởi file hosts cục bộ.
- **Khôi phục:** Xóa dòng giả mạo trong `/etc/hosts` và chạy `resolvectl flush-caches`.

---

### Task 2: Tấn công Local DNS Response Spoofing (On-path Spoofing)
- **Nguyên lý:** Attacker nằm cùng broadcast domain (VMnet4) với Victim và DNS Server. Attacker kích hoạt cờ Promiscuous Mode trên vSwitch, lắng nghe DNS Query từ `10.9.0.100` gửi tới `10.9.0.10`, trích xuất tức thì:
  - **Transaction ID (TID)**
  - **Source UDP Port**
  - **Question Record (QNAME)**
  Ngay lập tức, Attacker dựng gói DNS Response giả mạo trả lời `QNAME -> 10.9.0.200`, gửi về cho Victim trước khi DNS Server thật kịp xử lý.
- **Thực thi trên Kali (`10.9.0.30`):**
  ```bash
  sudo python3 /home/buiducanh/LAB9/dns_spoof_lab9.py --mode victim --target www.lab9.test --fake-ip 10.9.0.200
  ```
- **Thực thi trên VICTIM (`10.9.0.100`):**
  ```bash
  resolvectl flush-caches
  curl http://www.lab9.test
  ```
- **Kết quả & Bằng chứng Wireshark (`lab9_evidence.pcap`):**
  - Gói số 1 (Query): Victim gửi `A www.lab9.test` (TID: `0x2b4c`, Src Port: `48215`).
  - Gói số 2 (Spoofed Response, $\Delta t = 2\text{ms}$): Attacker gửi phản hồi có TID `0x2b4c`, Dst Port `48215`, trả về IP `10.9.0.200`. Nạn nhân chấp nhận gói này ngay lập tức.
  - Gói số 3 (Delayed Response từ DNS Server thật, $\Delta t = 1.5\text{s}$): Victim nhận được nhưng bỏ qua (gắn cờ duplicate/ignored) do cổng UDP đã đóng và câu hỏi đã được giải quyết.

---

### Task 3: Tấn công đầu độc bộ đệm DNS Resolver (Cache Poisoning)
- **Nguyên lý:**
  - Khi Victim truy vấn tên miền ủy quyền `www.external.lab9.test`, máy chủ `DNS-SERVER-LAB8` (`10.9.0.10`) không có dữ liệu sẵn trong cache nên phải gửi truy vấn đệ quy tới Authoritative DNS Server tại DMZ (`172.16.6.20`).
  - Dịch vụ Authoritative DNS giả lập (`authdns.service`) được cố tình cấu hình trễ $0.8\text{s} - 1.5\text{s}$.
  - Attacker trên mạng LAN bắt gói đệ quy này, làm giả gói DNS Response từ IP nguồn `172.16.6.20` gửi về cho Resolver `10.9.0.10` với nội dung `www.external.lab9.test -> 10.9.0.200`.
  - Resolver chấp nhận gói tin đến sớm, lưu bản ghi độc hại vào cache bộ nhớ của BIND9 với TTL xác định.
- **Thực thi trên Kali:**
  ```bash
  sudo python3 /home/buiducanh/LAB9/dns_spoof_lab9.py --mode resolver --target www.external.lab9.test --fake-ip 10.9.0.200
  ```
- **Kiểm chứng trên DNS Server (`10.9.0.10`):**
  ```bash
  sudo rndc dumpdb -cache
  sudo grep -i "10.9.0.200" /var/cache/bind/named_dump.db
  ```
  👉 **Bằng chứng thành công:** File dump cache ghi nhận:
  ```text
  www.external.lab9.test.    116    A    10.9.0.200
  ```
- **Hệ quả sâu rộng:** Ngay cả khi Attacker tắt tool tấn công, tất cả các máy trạm trong hệ thống khi hỏi `www.external.lab9.test` đều bị chuyển hướng sang máy chủ độc hại cho đến khi hết thời gian TTL của cache.

---

### Task 4: Dọn dẹp & Khôi phục an toàn (Restoration)
Sau khi kết thúc quá trình thử nghiệm:
1. **Attacker:** Nhấn `Ctrl + C` dừng hoàn toàn script `dns_spoof_lab9.py`.
2. **DNS Server:** Xóa trắng bộ nhớ đệm BIND9:
   ```bash
   sudo rndc flush
   ```
3. **Victim:** Xóa cache bộ giải hệ thống:
   ```bash
   resolvectl flush-caches
   ```
4. **Kiểm tra phục hồi:**
   ```bash
   dig @10.9.0.10 www.external.lab9.test +short
   # Kết quả trả về IP chuẩn: 172.16.6.10
   ```
5. **vSwitch VMware:** Chuyển Promiscuous Mode trên `VMnet4` trở lại giá trị **Deny** để đảm bảo tính cô lập mạng.

---

