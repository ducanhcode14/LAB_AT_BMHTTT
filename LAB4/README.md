LAB4 - Nmap



1\. Thông tin sinh viên



Họ và tên: Bùi Đức Anh



MSSV: 1150080126



Mã lớp: !!_THMT



Lab: LAB4 - Nmap



Môi trường: VMware Workstation



Kali Linux: 2026.2



Nmap: 7.99



Windows 11 VM: Windows 11



Mạng: Host-Only (VMnet1)



Video: https://youtu.be/ol9LqjSzrDU



2\. Mục tiêu



Thực hiện các kỹ thuật quét và nhận diện dịch vụ bằng Nmap trong môi trường lab cô lập Host-Only, gồm:



Host Discovery



TCP SYN Scan



Service/Version Detection



OS Detection



NSE vulnerability checking



Xuất kết quả Nmap ra TXT, XML, Grepable và HTML



Thực hiện tình huống trước và sau khi hardening trên Windows VM



3\. Mô hình môi trường



Các máy ảo được kết nối bằng mạng Host-Only VMnet1.



Máy



Địa chỉ IP



Vai trò



Kali Linux



192.168.199.129



Máy quét Nmap



Metasploitable 2



192.168.199.130



Máy mục tiêu



Windows 11 VM



192.168.199.128



Máy kiểm thử hardening



4\. Các kịch bản đã thực hiện



4.1. Kiểm tra địa chỉ IP



Kiểm tra địa chỉ IP của Kali Linux và Metasploitable 2 để xác nhận các máy nằm trong mạng Host-Only.



4.2. Host Discovery



Thực hiện phát hiện các máy đang hoạt động trong mạng:



nmap -sn 192.168.199.0/24



Kết quả ghi nhận 4 host đang hoạt động trong phạm vi quét.



4.3. TCP SYN Scan



Thực hiện:



nmap -sS 192.168.199.130



Kết quả ghi nhận 23 cổng TCP ở trạng thái open và 977 cổng ở trạng thái closed.



4.4. Service Detection



Thực hiện:



nmap -sV 192.168.199.130



Một số dịch vụ được nhận diện gồm FTP, SSH, Telnet, HTTP, SMB, MySQL, PostgreSQL, VNC và các dịch vụ khác.



4.5. OS Detection



Thực hiện:



nmap -O 192.168.199.130



Nmap nhận diện mục tiêu là hệ điều hành Linux 2.6.X.



4.6. NSE



Thực hiện kiểm tra bằng nhóm script:



nmap --script vuln 192.168.199.130



Kết quả cho thấy một số dịch vụ có dấu hiệu hoặc trạng thái lỗ hổng theo kết quả NSE. Các kết quả được sử dụng cho mục đích nhận diện và phân tích trong môi trường lab, không thực hiện khai thác tiếp.



4.7. Export kết quả



Các định dạng đã thực hiện:



TXT: lab4\_result.txt



XML: lab4\_result.xml



Grepable: smb.txt



HTML: lab4\_result.html



5\. Tình huống trước và sau Hardening



Windows 11 VM được sử dụng làm máy kiểm thử dịch vụ HTTP trên cổng 8080.



Dịch vụ test được chạy bằng Python HTTP Server. Sau đó thiết lập Windows Firewall để chặn kết nối TCP đến cổng 8080 và thực hiện lại phép quét Nmap từ Kali Linux.



Lệnh quét sử dụng cùng một mục tiêu và cùng một cổng ở hai thời điểm trước và sau hardening.



Kết quả Before: \[ĐIỀN KẾT QUẢ THỰC TẾ]



Kết quả After: \[ĐIỀN KẾT QUẢ THỰC TẾ]



Nhận xét: \[ĐIỀN NHẬN XÉT DỰA TRÊN OUTPUT THỰC TẾ]



6\. Kết quả PASS/FAIL



Nội dung



Kết quả



Kiểm tra IP Host-Only



PASS



Host Discovery



PASS



TCP SYN Scan



PASS



Service Detection



PASS



OS Detection



PASS



NSE



PASS



Export TXT/XML/Grepable/HTML



PASS



Before/After Hardening



\[ĐIỀN THEO OUTPUT THỰC TẾ]



7\. Lỗi gặp phải và cách xử lý



Lỗi 1: VMware không đủ bộ nhớ để khởi động Windows VM



VMware báo lỗi không thể cấp phát đủ bộ nhớ cho máy ảo.



Cách xử lý: Giảm RAM của Windows VM từ 6144 MB xuống 4096 MB và khởi động lại máy ảo.



Lỗi 2: Kiểm tra kết nối giữa các VM



Các máy ảo cần sử dụng cùng mạng Host-Only VMnet1.



Cách xử lý: Cấu hình card mạng của Kali, Metasploitable 2 và Windows VM về VMnet1 Host-Only và kiểm tra lại địa chỉ IP/kết nối.



8\. Cấu trúc file



LAB4/

├── README.md

├── \[MãLớp]-LAB3\_1150080126-BuiDucAnh.docx

├── lab4\_result.txt

├── lab4\_result.xml

├── lab4\_result.html

└── smb.txt

