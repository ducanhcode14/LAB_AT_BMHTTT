LAB 10: REMOTE DNS CACHE POISONING TRONG MÔI TRƯỜNG pfSense CÔ LẬP

Sinh viên thực hiện: Bùi Đức Anh  
MSSV: 1150080126 | Lớp: 11THMT  
---

I. THÔNG TIN HỆ THỐNG MÁY ẢO & TÀI KHOẢN TRUY CẬP

| Máy ảo | Tài khoản / Mật khẩu | IP phân vùng mạng | Vai trò trong bài Lab |
| :--- | :--- | :--- | :--- |
| **pfSense-LAB6** | `admin` / `BuiDucAnh2k4` | `192.168.56.254` (WebGUI)<br>`10.9.0.1` (DNSLAB)<br>`10.10.0.1` (ATTACKNET)<br>`172.16.6.1` (DMZ) | Firewall & Router định tuyến, cô lập phân vùng mạng. |
| **DMZ-WEB-LAB6** | `lab6admin` / `AnhDucBui2k4` | `172.16.6.10` (Web chính)<br>`172.16.6.20` (AUTH-DNS) | Máy chủ Web gốc Nginx 1.28 + Dịch vụ AUTH-DNS / Race Relay. |
| **DNS-SERVER-LAB8** | `lab8admin` / `AnhDucBui2k4` | `10.9.0.10 /24` (VMnet4 - DNSLAB) | Recursive Resolver (BIND 9.18 & Resolver mô phỏng cổng 33333). |
| **VICTIM-LAB8** | `lab8admin` / `AnhDucBui2k4` | `10.9.0.100 /24` (VMnet4 - DNSLAB) | Client người dùng (DNS trỏ về 10.9.0.10). |
| **ATTACKER-LAB8** (Kali) | `buiducanh` / `buiducanh2k4` | `10.10.0.200 /24` (VMnet5 - ATTACKNET) | Máy tấn công từ xa, chạy Evil DNS và Fake Web Server. |

---

II. THỨ TỰ KHỞI ĐỘNG CÁC MÁY ẢO
Khởi động các máy ảo theo thứ tự từ hạ tầng đến dịch vụ:
1. pfSense-LAB6
2. DMZ-WEB-LAB6
3. DNS-SERVER-LAB8
4. VICTIM-LAB8
5. ATTACKER-LAB8 (Kali Linux)

---

III. KỊCH BẢN THỰC THI LỆNH TỪNG BƯỚC (QUAY CLIP & CHỤP ẢNH)

BƯỚC 1: CHỤP ẢNH CẤU HÌNH MẠNG & CÔ LẬP
- Ảnh 1 (pfSense) Mở trình duyệt máy thật truy cập `https://192.168.56.254`.
  - Vào Interfaces > ATTACKNET: Thấy IP `10.10.0.1/24`, Gateway: `None`.
  - Vào Firewall > Rules > ATTACKNET: Thấy các luật Pass/Block đã tạo.
- Ảnh 2 (Kali): Chuột phải máy `ATTACKER-LAB8` trong VMware > Settings > Network Adapter (ở VMnet5, Promiscuous = Deny). Trên Terminal Kali gõ:
  ```bash
  ip a show eth0
  ip route show
  ```
  Chụp màn hình Terminal Kali và cửa sổ Settings VMware

---

BƯỚC 2: KIỂM TRA BASELINE (TRƯỚC TẤN CÔNG)
1. Trên VICTIM, mở Terminal gõ:
   ```bash
   resolvectl query www.example.lab10.test
   curl -i http://www.example.lab10.test
   ```
   Kết quả chuẩn: DNS trả về `172.16.6.10`; `curl` trả về `HTTP/1.1 200 OK` (Welcome to nginx!).
2. Chụp màn hình Terminal VICTIM .
3. Kiểm tra tính Remote (Mục E): Trên Kali, chạy lệnh bắt gói:
   ```bash
   sudo tcpdump -i any -n -v 'host 172.16.6.20 and udp port 53'
   ```
   Từ VICTIM gửi vài query `dig @10.9.0.10 www.example.lab10.test`. Dừng tcpdump trên Kali thấy `0 packets captured` (chứng minh Kali bị cô lập, không sniff được).

---

BƯỚC 3: TASK 1 - REMOTE KAMINSKY CACHE POISONING

1. Mở Cửa sổ 1 trên DMZ-WEB (AUTH-DNS Realtime Log):
```bash
journalctl -u lab10-authdns.service -f
```

2. Mở Cửa sổ 2 trên DNS-SERVER (Bật Resolver mô phỏng & Bắt gói pcap):
```bash
# Dừng BIND9 và bật bắt gói tin
sudo systemctl stop bind9
sudo tcpdump -i any -n -w /tmp/lab10_attack.pcap "udp port 53 or udp port 33333" &

# Chạy resolver mô phỏng lỗ hổng cổng 33333
sudo python3 -u /home/lab8admin/vuln_resolver_lab10.py
```
(Giữ nguyên cửa sổ này để xem log).

3. Mở Cửa sổ 3 trên KALI (Khởi động Evil DNS, Fake Web và Kích hoạt tấn công):
```bash
# Chuyển sang quyền root để không bị hỏi mật khẩu chạy ngầm
sudo su
# (Nhập mật khẩu: buiducanh2k4)

# Bật Evil DNS và Fake Web
python3 -u /home/buiducanh/evil_dns_lab10.py &
python3 -m http.server 80 --directory /home/buiducanh/fake_site &

# Kích hoạt tấn công Kaminsky Remote
python3 /home/buiducanh/kaminsky_remote_lab10.py
```

Màn hình Kali sẽ xuất hiện:
```text
[LAB10] remote mode: no sniffing. Trigger: r...example.lab10.test.
[LAB10] trigger reply received, bytes= 86
[VERIFY] www.example.lab10.test A = ['10.10.0.200']
SUCCESS: resolver now follows malicious delegation to attacker DNS
```

4. Chụp ảnh minh chứng:
- Chụp màn hình Terminal DMZ-WEB hiển thị dòng `[PENDING]` và `[HIT]`.
- Chụp màn hình Terminal DNS-SERVER hiển thị dòng `[CACHE] delegation poisoned: example.lab10.test. NS ns.evil... -> 10.10.0.200`.
- Trên DNS-SERVER, mở Terminal khác chạy:
  ```bash
  sudo pkill -f tcpdump
  tcpdump -r /tmp/lab10_attack.pcap -n -v "udp port 33333" | head -n 30
  ```
  Chụp màn hình hiển thị gói query và gói forged response trúng TXID.

---

BƯỚC 4: KIỂM CHỨNG HẬU QUẢ TRÊN VICTIM
Trên máy VICTIM, mở Terminal chạy lần lượt các lệnh:
```bash
# 1. Xóa cache
sudo resolvectl flush-caches

# 2. Phân giải tên miền (kết quả ra 10.10.0.200)
resolvectl query www.example.lab10.test

# 3. Kiểm tra bằng dig
dig @10.9.0.10 www.example.lab10.test

# 4. Truy cập Web (mở ra trang Fake Site của Kali)
curl -i http://www.example.lab10.test
```
Kết quả: Trả về `10.10.0.200` và trang `<h1>LAB 10 - DNS Cache Poisoning Demonstration</h1>`.  
Chụp màn hình Terminal VICTIM.

---

BƯỚC 5: TASK 2 & KHÔI PHỤC, HARDENING (ẢNH 8)

1. Khôi phục BIND9 trên DNS-SERVER:
   - Trên Terminal DNS-SERVER, bấm `Ctrl + C` để tắt `vuln_resolver_lab10.py`.
   - Khởi động lại BIND9:
     ```bash
     sudo systemctl start bind9
     sudo rndc flush
     sudo systemctl status bind9
     ```
2. Kiểm tra Hardening 1 (Recursion ACL):
   - Trên Kali, gửi query đệ quy:
     ```bash
     dig @10.9.0.10 test-kali.example.lab10.test
     ```
     Kết quả: BIND9 từ chối với mã lỗi `status: REFUSED (EDE: 18 Prohibited)`.
3. Kiểm tra Hardening 2 (Source-Port Randomization):**
   - Trên DNS-SERVER, bật bắt gói:
     ```bash
     sudo tcpdump -i any -n -c 8 "dst host 172.16.6.20 and udp dst port 53"
     ```
   - Trên VICTIM, gửi các query ngẫu nhiên:
     ```bash
     dig @10.9.0.10 rand1.example.lab10.test
     dig @10.9.0.10 rand2.example.lab10.test
     dig @10.9.0.10 rand3.example.lab10.test
     ```
   - Nhìn kết quả tcpdump trên DNS-SERVER: Cổng nguồn thay đổi liên tục (`59244`, `34072`, `59229`...) thay vì cố định `33333`.
4. Kiểm tra VICTIM hồi phục:
   - Trên VICTIM: `resolvectl query www.example.lab10.test` và `curl -i http://www.example.lab10.test` trả về lại trang web Nginx gốc `172.16.6.10`.
5.Chụp màn hình hiển thị BIND9 phục hồi, Kali bị `REFUSED` và bảng ngẫu nhiên hóa cổng nguồn