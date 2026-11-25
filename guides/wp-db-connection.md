# Error establishing a database connection

## TL;DR — fix nhanh

```php
// wp-config.php — kiểm tra 4 dòng này trước tiên
define('DB_NAME', 'ten_database');
define('DB_USER', 'user_db');
define('DB_PASSWORD', 'mat_khau_dung');
define('DB_HOST', 'localhost');
```

```bash
# Test kết nối MySQL thủ công
mysql -u user_db -p -h localhost ten_database
```

```bash
# Restart service nếu DB die
sudo systemctl restart mysql && sudo systemctl restart php-fpm
```

## Nguyên nhân thật sự

Câu "Error establishing a database connection" nghĩa là PHP không mở được socket tới MySQL/MariaDB. WordPress gọi `wpdb` → `mysqli_connect()` fail → `dead_db()` chạy và in ra thông báo đó. Nó **không** nói cho bạn biết *tại sao* fail — chỉ nói握手không thành. Nên đừng fix mù, hãy khoanh vùng: sai credentials, DB down, hay socket/host sai. 90% là một trong ba cái đó.

## 4-6 nguyên nhân phổ biến (xếp theo xác suất)

1. **Sai `DB_PASSWORD` / `DB_USER`** — hay gặp nhất, đặc biệt sau khi migrate hoặc đổi mật khẩu hosting.
2. **MySQL/MariaDB service đã chết** — OOM kill, hết disk, hoặc crash.
3. **`DB_HOST` sai** — dùng `localhost` trong khi DB nằm ở host khác, hoặc cần socket path.
4. **Hết `max_connections`** — connection pool đầy, site nhận request mới đều fail.
5. **Hết dung lượng disk** — MySQL không ghi được, treo kết nối.
6. **Table `wp_options` bị corrupt / file socket mất** sau reboot đột ngột.

## Fix từng nguyên nhân

### 1. Sai credentials

Đọc lại `wp-config.php`, so khớp với thông tin hosting. Test trực tiếp:

```bash
mysql -u user_db -p -h localhost
# nhập mật khẩu, nếu vào được → credentials đúng
```

Nếu sai, reset quyền trong MySQL:

```sql
ALTER USER 'user_db'@'localhost' IDENTIFIED BY 'mat_khau_moi';
FLUSH PRIVILEGES;
```

Rồi cập nhật lại `wp-config.php` cho khớp.

### 2. MySQL service chết

```bash
sudo systemctl status mysql      # hoặc mariadb
sudo systemctl restart mysql
sudo journalctl -u mysql -n 50 --no-pager   # xem log vì sao chết
```

Nếu log báo OOM → thiếu RAM, cân nhắc tăng swap hoặc giảm `innodb_buffer_pool_size`.

### 3. `DB_HOST` sai

Nhiều môi trường managed (RDS, Cloudways...) không dùng `localhost`:

```php
// Ví dụ khi DB ở host riêng
define('DB_HOST', 'db.example.com:3306');
```

Nếu dùng socket:

```php
define('DB_HOST', 'localhost:/var/run/mysqld/mysqld.sock');
```

### 4. Hết `max_connections`

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"
```

Tăng tạm:

```sql
SET GLOBAL max_connections = 200;
```

Vĩnh viễn trong `my.cnf`:

```ini
[mysqld]
max_connections = 200
```

### 5. Hết disk

```bash
df -h
du -sh /var/lib/mysql
```

Dọn log cũ rồi restart:

```bash
sudo journalctl --vacuum-size=200M
sudo systemctl restart mysql
```

### 6. Repair table

```bash
mysqlcheck -u root -p --auto-repair --all-databases
```

Hoặc bật repair của WordPress tạm thời:

```php
// wp-config.php — thêm rồi XÓA SAU KHI XONG
define('WP_ALLOW_REPAIR', true);
// truy cập /wp-admin/maint/repair.php
```

## Nếu vẫn không hết: Edge cases cần soi sâu hơn

- **Proxy/cloud env vars**: `HTTP_PROXY`, `ALL_PROXY`, hoặc socket bị route qua proxy khiến `mysqli` connect nhầm chỗ. Cần `env | grep -i proxy` và test bypass.
- **Multiple vhosts / nhiều `wp-config.php`**: bạn sửa một file nhưng web server đang load file khác (path `DOCUMENT_ROOT` lệch). Soi `phpinfo()` để biết file nào thực sự được nạp.
- **DNS propagation sau khi đổi host DB**: record `db.example.com` chưa lan, `getent hosts db.example.com` trả IP cũ. Chờ TTL hoặc override `/etc/hosts` tạm để verify.
- **Chênh lệch phiên bản client/server** hoặc `caching_sha2_password` vs `mysql_native_password` trên MySQL 8.
- **Firewall/security group** chặn port 3306 từ app server.

Mấy tình huống này một AI agent có thể tự debug từng bước: chạy `getent`, `nc -zv host 3306`, so sánh `DOCUMENT_ROOT` với `wp-config.php` đang load, và lần theo log — thay vì bạn phải đoán.

## FAQ

**Sửa `wp-config.php` rồi mà vẫn lỗi, tại sao?**
Có thể web server đang nạp một file khác (nhiều vhost), hoặc service DB vẫn chưa lên. Kiểm tra `phpinfo()` để biết file config thực tế và `systemctl status mysql`.

**Có nên bật `WP_ALLOW_REPAIR` lâu dài không?**
Không. Đó là cửa hậu, để `true` vô thời hạn là rủi ro bảo mật. Bật → repair → tắt ngay.

**Làm sao phân biệt lỗi credentials với DB down?**
Chạy `mysql -u user -p -h host`. Nếu báo "Access denied" → sai credentials. Nếu "Can't connect" → DB/service/host có vấn đề.

---

Nếu edge case của bạn vẫn kẹt — proxy lạ, nhiều venv/vhost chồng chéo, hay DNS chưa lan — có thể dùng agent debug tự động để lần từng bước thay vì đoán mò. [Dùng thử agent tại đây](https://example.com/agent-debug).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
