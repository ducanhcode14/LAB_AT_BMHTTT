# LAB9 — Local DNS Attack: DNS Spoofing và Cache Poisoning trong mạng ảo cô lập

- **Sinh viên thực hiện:** Bùi Đức Anh
- **Mã số sinh viên:** 1150080126
- **Lớp:** 11_ĐH_THMT
- **Môn học:** An toàn và Bảo mật Hệ thống Thông tin
- **Video Demo quá trình thực nghiệm:** [YouTube - Video Demo LAB 9](https://youtu.be/w9AlWv8m_p4)
- **Môi trường triển khai:** VMware Workstation Pro, pfSense Firewall CE 2.7.2, Ubuntu Server 24.04 LTS, Kali Linux 2024/2026.

---

## 1. Giới thiệu tổng quan

Bài thực hành này nghiên cứu chuyên sâu về các kỹ thuật tấn công và can thiệp dịch vụ phân giải tên miền (DNS) trong môi trường mạng cục bộ được bảo vệ bởi tường lửa **pfSense**, bao gồm:

1. **Client-side Host File Tampering:** Can thiệp file cấu hình tĩnh `/etc/hosts` và phân tích thứ tự ưu tiên của trình phân giải hệ thống (Name Service Switch - NSSwitch).
2. **Direct DNS Response Spoofing (On-path / Local Spoofing):** Giả mạo gói tin DNS Response trả lời trực tiếp cho máy trạm (Victim) nhanh hơn máy chủ DNS thật trên cùng broadcast domain.
3. **DNS Resolver Cache Poisoning (Đầu độc bộ nhớ đệm BIND9):** Tấn công chu trình phân giải đệ quy (Recursive Query) giữa Resolver và Authoritative Server, cấy bản ghi giả mạo vào bộ nhớ đệm của DNS Server nội bộ.
4. **Restoration & Hardening:** Các phương thức dọn dẹp cache, khôi phục dịch vụ an toàn và biện pháp phòng vệ (DNSSEC, DNS Cookie, Egress Filtering, Layer 2 Security).

Toàn bộ quá trình thực nghiệm được triển khai trong phân vùng mạng cô lập an toàn (**VMnet4 - DNSLAB - 10.9.0.0/24**), không định tuyến ra Internet công cộng, tuân thủ nghiêm ngặt các nguyên tắc đạo đức và an toàn thông tin.

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
          |  DNS-SERVER-LAB9  |         |   ATTACKER-LAB9   |       |    VICTIM-LAB9    |
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
| **DNS-SERVER-LAB9** | Ubuntu Server 24.04 | VMnet4 | `10.9.0.10/24` | Máy chủ DNS đệ quy nội bộ (BIND 9.18), Forwarder |
| **ATTACKER-LAB9** | Kali Linux | VMnet4 | `10.9.0.30/24`<br>`10.9.0.200/24` (alias) | Kẻ tấn công; Web server giả mạo (port 80); chạy công cụ Sniff & Spoof Scapy |
| **VICTIM-LAB9** | Ubuntu Desktop 24.04 | VMnet4 | `10.9.0.100/24` | Máy trạm nạn nhân, cấu hình DNS trỏ về `10.9.0.10` |
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
| **5** | `BLOCK`| Any | `10.9.0.0/24`| `10.6.0.0/24` | Any | Cách ly hoàn toàn DNSLAB với mạng LAN Domain Controller |
| **6** | `BLOCK`| Any | `10.9.0.0/24`| `192.168.56.0/24` | Any | Ngăn chặn truy cập vùng Quản trị (MGMT) |
| **7** | `BLOCK`| Any | `10.9.0.0/24`| Any | Any | **Default Block:** Cách ly tuyệt đối, không cho ra Internet/WAN |

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
  sudo python3 dns_spoof_lab9.py --mode victim --target www.lab9.test --fake-ip 10.9.0.200
  ```
- **Thực thi trên VICTIM (`10.9.0.100`):**
  ```bash
  resolvectl flush-caches
  curl http://www.lab9.test
  ```
- **Kết quả & Bằng chứng Wireshark:**
  - Gói số 1 (Query): Victim gửi `A www.lab9.test` (TID: `0x2b4c`, Src Port: `48215`).
  - Gói số 2 (Spoofed Response, $\Delta t = 2\text{ms}$): Attacker gửi phản hồi có TID `0x2b4c`, Dst Port `48215`, trả về IP `10.9.0.200`. Nạn nhân chấp nhận gói này ngay lập tức.
  - Gói số 3 (Delayed Response từ DNS Server thật, $\Delta t = 1.5\text{s}$): Victim nhận được nhưng bỏ qua (gắn cờ duplicate/ignored) do cổng UDP đã đóng và câu hỏi đã được giải quyết.

---

### Task 3: Tấn công đầu độc bộ đệm DNS Resolver (Cache Poisoning)
- **Nguyên lý:**
  - Khi Victim truy vấn tên miền ủy quyền `www.external.lab9.test`, máy chủ `DNS-SERVER-LAB9` (`10.9.0.10`) không có dữ liệu sẵn trong cache nên phải gửi truy vấn đệ quy tới Authoritative DNS Server tại DMZ (`172.16.6.20`).
  - Dịch vụ Authoritative DNS giả lập (`authdns.service`) được cố tình cấu hình trễ $0.8\text{s} - 1.5\text{s}$.
  - Attacker trên mạng LAN bắt gói đệ quy này, làm giả gói DNS Response từ IP nguồn `172.16.6.20` gửi về cho Resolver `10.9.0.10` với nội dung `www.external.lab9.test -> 10.9.0.200`.
  - Resolver chấp nhận gói tin đến sớm, lưu bản ghi độc hại vào cache bộ nhớ của BIND9 với TTL xác định.
- **Thực thi trên Kali:**
  ```bash
  sudo python3 dns_spoof_lab9.py --mode resolver --target www.external.lab9.test --fake-ip 10.9.0.200
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

## 5. Trả lời câu hỏi phân tích chuyên sâu (Q&A)

### Câu 1: Tại sao `/etc/hosts` có thể ảnh hưởng trình duyệt nhưng `dig @DNS-SERVER` vẫn trả kết quả DNS gốc?
**Trả lời:**
Trình duyệt web và các ứng dụng thông thường sử dụng cơ chế Name Service Switch (NSS) thông qua thư viện resolver chuẩn của hệ điều hành (hàm `getaddrinfo` / `gethostbyname`). Cấu hình trong `/etc/nsswitch.conf` mặc định ưu tiên nguồn `files` (`/etc/hosts`) trước `dns`. Khi phân giải tên miền, trình duyệt tìm thấy bản ghi tĩnh trong `/etc/hosts` và sử dụng ngay mà không gửi truy vấn ra mạng. Ngược lại, công cụ chẩn đoán `dig` bỏ qua toàn bộ cơ chế NSS của hệ điều hành và gửi trực tiếp gói tin UDP/TCP port 53 tới địa chỉ IP của DNS Server được chỉ định, do đó `dig` luôn nhận được câu trả lời gốc từ DNS Server.

### Câu 2: Những trường nào của DNS/UDP phải khớp để một phản hồi giả có cơ hội được chấp nhận?
**Trả lời:**
Để gói phản hồi DNS giả mạo được chấp nhận, phải khớp chính xác 5 thành phần:
1. **IP Source và Destination:** IP nguồn phải là IP của DNS Server/Authoritative Server, IP đích là IP máy gửi query.
2. **UDP Port Source và Destination:** UDP Source Port phải là 53, UDP Destination Port phải khớp chính xác cổng nguồn tạm thời (ephemeral port) ngẫu nhiên của client.
3. **Transaction ID (TXID):** Mã định danh 16-bit trong DNS Header phải trùng khớp hoàn toàn giữa query và response.
4. **Question Section:** Tên miền (QNAME), loại bản ghi (QTYPE) và lớp (QCLASS) phải trùng khớp.
5. **Cờ DNS và DNS Cookie:** Cờ QR=1 (Response), và nếu Resolver bật xác thực DNS Cookie (RFC 7873), gói phản hồi bắt buộc phải có mã cookie hợp lệ.

### Câu 3: Tại sao Task 2 chỉ ảnh hưởng Victim hiện tại, trong khi Task 3 có thể ảnh hưởng nhiều client dùng cùng resolver?
**Trả lời:**
- **Ở Task 2 (Direct Spoofing):** Phản hồi giả được gửi trực tiếp đến socket đang chờ của Victim. Bản ghi giả chỉ được lưu trong bộ nhớ tạm của tiến trình/stub resolver trên máy Victim đó, máy chủ DNS trung gian không hề biết hay lưu giữ thông tin này. Vì vậy chỉ một client duy nhất bị ảnh hưởng.
- **Ở Task 3 (DNS Cache Poisoning):** Mục tiêu tấn công là chính máy chủ trung gian DNS-SERVER (Recursive Resolver). Khi BIND9 chấp nhận bản ghi giả, nó lưu vào vùng nhớ đệm (Cache) dùng chung. Tất cả các máy trạm khác trong mạng khi gửi yêu cầu phân giải tên miền đó tới Resolver đều nhận được dữ liệu nhiễm độc từ cache, gây ảnh hưởng trên diện rộng mà không cần tấn công từng client riêng lẻ.

### Câu 4: Vai trò của TTL (Time to Live) trong cache poisoning là gì?
**Trả lời:**
TTL là trường thời gian tồn tại (tính bằng giây) của bản ghi trong bộ nhớ đệm:
- **Quyết định thời gian hiệu lực của cuộc tấn công:** Trong suốt thời gian TTL chưa hết hạn (ví dụ TTL=120s), Resolver sẽ trực tiếp trả về địa chỉ IP giả mạo cho mọi truy vấn của client mà không gửi bất kỳ truy vấn nào ra Authoritative Server thật, giúp duy trì quyền kiểm soát nạn nhân ổn định mà kẻ tấn công không cần tiếp tục chạy script bắt gói.
- **Đóng vai trò là cửa sổ tấn công:** Resolver chỉ gửi truy vấn ra ngoài khi bản ghi không có trong cache hoặc TTL đã hết hạn. Kẻ tấn công chỉ có thể thực hiện poisoning khi cache rỗng.

### Câu 5: Tại sao VirtualBox Promiscuous Mode = Allow VMs (hoặc VMware noPromisc = FALSE) cần thiết trong bài này?
**Trả lời:**
Các card mạng ảo kết nối qua Virtual Switch (Layer 2). Theo mặc định để đảm bảo an toàn, switch ảo chỉ chuyển tiếp khung tin unicast có địa chỉ MAC đích trùng với card mạng của chính VM đó. Trong bài lab, Attacker thực hiện tấn công On-Path Sniffing trên cùng mạng LAN, bắt buộc phải nhìn thấy truy vấn unicast giữa Victim (`10.9.0.100`) và DNS-SERVER (`10.9.0.10`) để lấy tức thì Transaction ID và UDP Port. Chế độ Promiscuous Mode cho phép card mạng của Attacker bắt được toàn bộ các gói tin unicast của các máy khác trong cùng phân đoạn mạng ảo.

### Câu 6: Vì sao LAB dùng mạng DNSLAB riêng thay vì tấn công trực tiếp DC01 DNS của LAN-DOMAIN?
**Trả lời:**
1. **Bảo vệ hạ tầng xác thực:** DC01 là Domain Controller quản lý Active Directory, Kerberos và DNS nội bộ của domain kế thừa từ các lab trước. Tấn công DNS trực tiếp lên DC01 có thể làm lỗi đồng bộ dịch vụ AD, cô lập các server thành viên và làm mất tính toàn vẹn của cả chuỗi bài thực hành.
2. **Kiểm soát môi trường thử nghiệm:** DNSLAB (`10.9.0.0/24`) là một subnet cô lập phía sau pfSense, cho phép bật/tắt các thiết lập đặc thù (như vô hiệu hóa send-cookie, thêm delay) nhằm tái lập cuộc tấn công DNS một cách tất định (deterministic), an toàn tuyệt đối và dễ dàng rollback trạng thái.

### Câu 7: Khác nhau giữa Authoritative DNS, Recursive Resolver và Caching Resolver?
**Trả lời:**
- **Authoritative DNS:** Là máy chủ nắm giữ cơ sở dữ liệu gốc (zone file) cho một vùng tên miền cụ thể. Nó có thẩm quyền đưa ra câu trả lời chính thức cho các bản ghi trong zone của mình, không tìm kiếm giúp client các domain bên ngoài.
- **Recursive Resolver:** Máy chủ trung gian nhận truy vấn từ client, nếu chưa có câu trả lời, nó tự động thay mặt client thực hiện toàn bộ chuỗi tìm kiếm đệ quy (hỏi từ Root Server, TLD Server đến Authoritative Server) cho đến khi lấy được kết quả cuối cùng trả về cho client.
- **Caching Resolver:** Là cơ chế lưu trữ tạm thời các kết quả phân giải DNS vào bộ nhớ đệm (RAM) trong khoảng thời gian TTL. Khi có yêu cầu truy vấn lặp lại, nó trả lời ngay lập tức từ bộ đệm, giúp giảm thiểu độ trễ mạng và giảm tải cho các máy chủ tên miền gốc trên Internet.

### Câu 8: DNSSEC giải quyết vấn đề gì và vì sao zone `.test` unsigned trong LAB không tự được bảo vệ?
**Trả lời:**
- **DNSSEC (Domain Name System Security Extensions)** giải quyết triệt để vấn đề giả mạo dữ liệu bằng cách sử dụng chữ ký số mật mã công khai (khóa RRSIG, DNSKEY, DS). Resolver xác thực chuỗi tin cậy (Chain of Trust) từ Root DNS xuống đến zone lá. Nếu phản hồi bị sửa đổi IP hoặc sai chữ ký, Resolver sẽ bác bỏ và trả về SERVFAIL.
- Trong bài lab, zone `lab9.test` và `external.lab9.test` là các zone cục bộ tự tạo và chưa được ký số (Unsigned zone). Khi DNS-SERVER kiểm tra, nó nhận thấy zone không có bản ghi DS/RRSIG hợp lệ nên coi đây là dữ liệu không an toàn (Insecure) và chấp nhận mọi phản hồi khớp TXID/cổng UDP mà không kiểm tra chữ ký, do đó cuộc tấn công giả mạo vẫn thành công.

### Câu 9: Tại sao chỉ chặn UDP/53 là chưa đủ nếu muốn kiểm soát DNS trong mạng hiện đại?
**Trả lời:**
Trong mạng hiện đại, lưu lượng DNS không chỉ chạy trên UDP/53 mà đã mở rộng sang nhiều giao thức mã hóa và kênh truyền khác:
1. **DNS over TCP (TCP port 53):** Sử dụng khi gói tin vượt kích thước cho phép hoặc truyền vùng zone transfer.
2. **DoT (DNS over TLS - RFC 7858):** Sử dụng cổng TCP 853, mã hóa bảo mật toàn bộ dữ liệu DNS.
3. **DoH (DNS over HTTPS - RFC 8484):** Sử dụng cổng TCP 443 (trộn lẫn hoàn toàn với lưu lượng web HTTPS thông thường), ngụy trang truy vấn DNS dưới dạng luồng dữ liệu web được mã hóa.
4. **DoQ (DNS over QUIC - RFC 9250):** Sử dụng nền tảng UDP port 853/784 với giao thức QUIC.

Nếu chỉ chặn UDP/53, người dùng hoặc phần mềm độc hại có thể dễ dàng chuyển hướng sang DoH (TCP 443) hoặc DoT để vượt qua tường lửa.

### Câu 10: Nêu tối thiểu bốn biện pháp Hardening sau khi hoàn thành bài.
**Trả lời:**
1. **Thiết lập DNS Egress Filtering trên Firewall:** Chặn toàn bộ lưu lượng DNS (cổng 53, 853) từ mạng máy trạm ra ngoài; chỉ cho phép duy nhất máy chủ DNS Recursive được chỉ định gửi truy vấn ra Internet/DMZ.
2. **Kích hoạt xác thực DNSSEC:** Cấu hình `dnssec-validation auto;` trên BIND9 và ký số DNSSEC cho các zone nội bộ để ngăn chặn dữ liệu giả mạo.
3. **Bật DNS COOKIE (RFC 7873) và ngẫu nhiên hóa cổng nguồn:** Kích hoạt cookie giữa Resolver và Authoritative Server kết hợp ngẫu nhiên hóa 16-bit Transaction ID và 16-bit UDP Source Port để vô hiệu hóa các cuộc tấn công đoán mò off-path (Kaminsky).
4. **Phân đoạn mạng và phòng vệ Layer 2:** Tách biệt DNS Server, Client và các vùng mạng máy chủ vào các VLAN riêng; tắt chế độ Promiscuous Mode trên Switch; triển khai Dynamic ARP Inspection (DAI), DHCP Snooping và Port Security để ngăn chặn bắt lén và giả mạo IP/MAC trong nội bộ mạng.

