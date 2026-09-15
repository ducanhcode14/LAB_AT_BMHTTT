# LAB 1 - Bắt gói tin Telnet - SSH trong Wireshark

## 1. Thông tin sinh viên

- Họ và tên: Bùi Đức Anh
- Mã số sinh viên: 1150080126
- Tên bài Lab: Lab 1 - Bắt gói tin Telnet - SSH / Examining SSH & Telnet in Wireshark

## 2. Nội dung đã thực hiện

- Cài đặt và cấu hình Ubuntu Server.
- Cài đặt và cấu hình dịch vụ Telnet.
- Kiểm tra và sử dụng dịch vụ SSH.
- Sử dụng máy Client để kết nối đến Server.
- Bắt gói tin Telnet bằng Wireshark với bộ lọc TCP port 23.
- Sử dụng Follow TCP Stream để quan sát dữ liệu Telnet.
- Bắt gói tin SSH bằng Wireshark với bộ lọc TCP port 22.
- So sánh khả năng bảo mật giữa Telnet và SSH.

## 3. Kết quả thực hiện

### Telnet

- Kết nối Telnet đến Server thành công.
- Wireshark bắt được các gói tin trên TCP port 23.
- Dữ liệu Telnet có thể quan sát dưới dạng plaintext bằng Follow TCP Stream.
- Thông tin đăng nhập có nguy cơ bị lộ.

### SSH

- Kết nối SSH đến Server thành công.
- Wireshark bắt được các gói tin trên TCP port 22.
- Có thể quan sát quá trình bắt tay và các thông tin kết nối.
- Nội dung phiên SSH được mã hóa nên không thể đọc trực tiếp username, password và lệnh như Telnet.

## 4. Môi trường thực hiện

- Server: Ubuntu Server
- Client: Windows
- Công cụ bắt gói tin: Wireshark 4.6.8
- Công cụ kết nối: PuTTY / Command Prompt
- Phần mềm máy ảo: Oracle VirtualBox
- IP Server: 192.168.56.101