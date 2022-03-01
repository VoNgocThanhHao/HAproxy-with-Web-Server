# Cài đặt cân bằng tải cho hệ thống máy chủ WEB

## Mô hình

![](https://i.imgur.com/y9MIfhc.png)

## Yêu cầu

Chuẩn bị 2 server có cài sẵn **nginx** và **wordpress**, 1 máy để cài HAProxy:

    Server 1: 172.20.232.101
    Server 2: 172.20.232.102
    HAProxy: 172.20.232.9

Tham khảo nếu server chưa có sẵn **nginx** và **wordpress**: https://github.com/VoNgocThanhHao/wordpress.git

## Máy HAProxy

Tạo host cho các máy server:

    sudo nano /etc/hosts

Thêm vào các dòng bên dưới:

    172.20.232.101 web-server1
    172.20.232.102 web-server2

Cài đặt bộ cân bằng tải HAProxy:

    sudo apt-get update
    sudo apt-get upgrade
    sudo sudo apt install haproxy

Kiểm tra phiên bản của HAProxy:

    haproxy -v

Cấu hình HAproxy làm bộ cân bằng tải:

    sudo nano /etc/haproxy/haproxy.cfg

    frontend web-frontend
        bind 172.20.232.9:80
        mode http
        default_backend web-backend

    backend web-backend
        balance roundrobin
        server web-server1 172.20.232.101:80 check
        server web-server2 172.20.232.102:80 check

Cấu hình giám sát HAproxy:

    listen stats
    bind 172.20.232.9:8080
    mode http
    option forwardfor
    option httpclose
    stats enable
    stats show-legends
    stats refresh 5s
    stats uri /stats
    stats realm Haproxy\ Statistics
    stats auth fit:Knn2022@            
    stats admin if TRUE
    default_backend web-backend

Thoát và lưu lại. Sau đó kiểm tra file cấu hình đã hợp lệ chưa bằng lệnh:

    haproxy -c -f /etc/haproxy/haproxy.cfg

Nếu kết quả như hình dưới thì cấu hình đã hợp lệ:

![](https://i.imgur.com/meqEsoT.png)

Khởi động lại dịch vụ HAProxy:

    sudo systemctl restart haproxy.service

Kiểm tra hoạt động của HAProxy:

    sudo systemctl status haproxy.service

Trạng thái đang bật:

![](https://i.imgur.com/DqaCwTn.png)

Kiểm tra xem HAproxy đã hoạt động chưa bằng cách truy cập vào `http://172.20.232.9`:

![](https://i.imgur.com/D2ZEwip.png)

Khi ta reload lại trang, HAproxy sẽ chuyển request đến server 2:

![](https://i.imgur.com/TEZmsX2.png)

## Cấu hình cho 2 server dùng chung 1 cơ sở dữ liệu

**Cơ sở dữ liệu được dùng chung sẽ nằm ở 172.20.232.101**

### Máy 172.20.232.101

Chỉnh sửa file cấu hình của **mysql**:

    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

Tìm đến hàng `bind-address = 127.0.0.1` và thêm dấu `#` vào trước câu để ghi chú nó lại:

![](https://i.imgur.com/gAuBeQ1.png)

Khởi động lại **mysql**:

    sudo systemctl restart mysql

Khởi động **mysql** để tạo `user` và cấp quyền cho server 2 truy cập:

    mysql -u root -p

Nhập mật khẩu của `root`. Sau khi khởi động xong chạy câu lệnh:

    CREATE USER 'wordpress'@'172.20.232.102' IDENTIFIED BY '123456';

Tiếp theo là cấp quyền cho server 2:

    GRANT CREATE, ALTER, DROP, INSERT, UPDATE, DELETE, SELECT, REFERENCES, RELOAD on *.* TO 'wordpress'@'172.20.232.102' WITH GRANT OPTION;

Sau đây, bạn nên chạy lệnh FLUSH PRIVILEGES. Điều này sẽ giải phóng bất kỳ bộ nhớ nào mà máy chủ đã lưu vào bộ nhớ cache do kết quả của các câu lệnh CREATE USER và GRANT trước đó:

    FLUSH PRIVILEGES;

Và thoát khỏi **mysql**:

    exit

Cuối cùng là chỉnh sửa hostname:

    sudo nano /etc/hosts

Thêm dòng `172.20.232.102 sqlclient` vào như hình:

![](https://i.imgur.com/9ma0rlV.png)

Thoát và lưu lại.

### Máy 172.20.232.102

Chỉnh sửa file cấu hình của **mysql**:

    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

Tìm đến hàng `bind-address = 127.0.0.1` và thêm dấu `#` vào trước câu để ghi chú nó lại:

![](https://i.imgur.com/gAuBeQ1.png)

Khởi động lại **mysql**:

    sudo systemctl restart mysql

Chỉnh sửa hostname tương tự như trên:

    sudo nano /etc/hosts

Thêm `172.20.232.101 mysqlserver` vào:

![](https://i.imgur.com/AEw9bmh.png)
    
Thoát và lưu lại.

Kiểm tra kết quả bằng lệnh:

    mysql -h mysqlserver -u wordpress -p

Nhập mật khẩu, nếu kết quả như hình là đã thành công:

![](https://i.imgur.com/NVwEzn5.png)

Cuối cùng truy cập vào file cấu hình của **wordpress** để thay đổi cấu hình cơ sở dữ liệu:

    nano /var/www/wordpress/wp-config.php

Chỉnh sửa **db_name** như ở máy `172.20.232.101` và **db_user**, **db_password** thành tên và mật khẩu vừa tạo ở máy `172.20.232.101`.

Ở dòng `define( 'DB_HOST', 'localhost' );` thay đổi thành:

    define( 'DB_HOST', 'mysqlserver' );

![](https://i.imgur.com/vFCWwTl.png)

Kiểm tra kết quả, bây giờ reload lại nhiều lần thì cũng sẽ chỉ nhận được dữ liệu ở server 1:

![](https://i.imgur.com/a1ph5B5.png)



# So sánh các thuật toán cân bằng tải HAproxy cung cấp:

- `roundrobin`: các request sẽ được chuyển đến server theo lượt. Đây là thuật toán mặc định được sử dụng cho HAProxy
- `leastconn`: các request sẽ được chuyển đến server nào có ít kết nối đến nó nhất
- `source`: các request được chuyển đến server bằng các hash của IP người dùng. Phương pháp này giúp người dùng đảm bảo luôn kết nối tới một server
