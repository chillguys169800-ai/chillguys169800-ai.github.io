# Error: MySQL 'Too many connections' error on small site

## TL;DR — fix nhanh

```php
// 1. Đóng kết nối ngay sau khi dùng (persistent = false)
$pdo = new PDO($dsn, $user, $pass, [PDO::ATTR_PERSISTENT => false]);

// 2. Bật connection pooling ở tầng app thay vì mở mới mỗi request
$pool->get(); // reuse thay vì new

// 3. Nâng giới hạn tạm thời để xác nhận đúng bệnh
mysql> SET GLOBAL max_connections = 200;
```

## Nguyên nhân thật

Lỗi `Too many connections` nghĩa là số kết nối MySQL đang mở đã chạm `max_connections`. MySQL từ chối kết nối mới ngay ở bước handshake, nên app trả về 500 chứ không phải vì query chậm.

Điểm gây hiểu nhầm: "site nhỏ" không liên quan. Vấn đề không phải lượng traffic, mà là **tỷ lệ mở/đóng kết nối**. Một site 50 visit/ngày vẫn có thể treo nếu mỗi request mở 3-4 kết nối và không bao giờ đóng. Kết nối MySQL là tài nguyên server-side, giữ mở thì tốn RAM và slot. Khi slot đầy, mọi request kế tiếp chết — kể cả của bạn.

Nói ngắn: đây là lỗi quản lý vòng đời kết nối, không phải lỗi tải.

## 4 nguyên nhân phổ biến (xếp theo xác suất gặp)

1. **Persistent connection bật** (`pconnect` / `ATTR_PERSISTENT => true`) — dưới PHP-FPM, mỗi worker giữ một kết nối sống mãi, nhân với số worker là đầy slot.
2. **Connection leak** — mở trong hàm, throw exception giữa chừng, không đóng trong `finally`.
3. **`max_connections` đặt quá thấp** so với `pm.max_children` của PHP-FPM.
4. **Nhiều app/venv dùng chung một MySQL** mà không ai biết, cộng dồn vượt hạn mức.
5. **Worker/cron chạy song song** mở kết nối riêng, không dùng chung pool.

## Fix từng nguyên nhân

### 1. Tắt persistent connection

```php
// SAI — mỗi FPM worker giữ kết nối vĩnh viễn
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_PERSISTENT => true,
]);

// ĐÚNG — đóng khi request kết thúc
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_PERSISTENT => false,
    PDO::ATTR_TIMEOUT     => 5,
]);
```

### 2. Chống leak bằng try/finally

```php
function getReport(PDO $pdo, int $id): array {
    $stmt = $pdo->prepare("SELECT * FROM reports WHERE id = ?");
    try {
        $stmt->execute([$id]);
        return $stmt->fetch(PDO::FETCH_ASSOC) ?: [];
    } finally {
        $stmt->closeCursor(); // giải phóng ngay, không đợi GC
    }
}
```

### 3. Căn chỉnh `max_connections` với FPM

```ini
; php-fpm pool (pm = dynamic)
pm.max_children = 30

; MySQL — phải > max_children + headroom cho cron/admin
max_connections = 100
```

Công thức an toàn: `max_connections >= pm.max_children + 20`. Nếu FPM cho 30 con chạy đồng thời mà MySQL chỉ cho 50, dư 20 còn lại dễ bị cron ăn hết.

### 4. Kiểm tra ai đang chiếm slot

```sql
-- Ai đang giữ kết nối?
SELECT user, host, COUNT(*) AS conns
FROM information_schema.processlist
GROUP BY user, host
ORDER BY conns DESC;

-- Đang bị sleep (chưa đóng) bao nhiêu?
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';
```

Nếu thấy nhiều `Sleep` từ cùng một user, gần như chắc chắn là leak hoặc persistent.

### 5. Dùng shared pool cho worker song song

```python
# Thay vì mỗi task tạo engine mới
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,   # tự loại kết nối đã chết
    pool_recycle=1800,    # đóng trước khi MySQL cắt (wait_timeout)
)
```

## Nếu vẫn không hết: Edge cases cần soi sâu hơn

- **Proxy env (RDS Proxy / ProxySQL):** proxy giữ idle connection phía backend, `max_connections` thật nằm ở proxy chứ không phải MySQL. Bạn nâng `max_connections` trên DB nhưng vô ích vì proxy mới là nơi chặn. Cần soi config proxy riêng.
- **Multiple venvs / nhiều version app:** hai môi trường (staging + prod) trỏ chung một DB qua `.env` khác nhau, mỗi cái mở pool riêng. Nhìn một phía tưởng ổn, cộng lại mới đầy. Phải map từng process về từng connection string.
- **DNS propagation sau failover:** sau khi đổi host DB, kết nối cũ vẫn bám IP cũ do cache DNS/TTL, mở song song với IP mới → double số kết nối trong lúc propagate.

Ba tình huống này khó chẩn đoán bằng mắt vì triệu chứng giống hệt nhau. Nếu bạn đang kẹt ở đây, một AI agent có thể tự chạy từng bước debug giúp bạn: đọc `processlist`, đối chiếu config FPM/proxy, trace từng connection string về đúng nguồn.

## FAQ

**Site nhỏ sao vẫn Too many connections?**
Vì lỗi phụ thuộc tỷ lệ mở/đóng kết nối, không phụ thuộc traffic. Persistent connection hoặc leak làm slot đầy dù lượng truy cập thấp.

**Nâng `max_connections` có phải fix triệt để không?**
Không. Nó chỉ trì hoãn. Nếu gốc là leak hoặc persistent, slot sẽ đầy lại. Phải sửa vòng đời kết nối.

**Làm sao biết là persistent hay leak?**
Chạy `SELECT * FROM information_schema.processlist`. Persistent hiện nhiều `Sleep` ổn định theo số worker; leak hiện số kết nối tăng dần theo thời gian.

---

Nếu edge case của bạn vẫn kẹt — proxy env, nhiều venv hay DNS sau failover — bạn có thể dùng agent debug tự động để nó soi từng bước thay vì đoán mò. [Dùng agent debug tại đây](#).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
