# Error: 403 Forbidden after deploying to hosting

## TL;DR — fix nhanh

```bash
sudo chown -R www-data:www-data /var/www/app && sudo chmod -R u=rwX,go=rX /var/www/app
sudo find /var/www/app/storage /var/www/app/bootstrap/cache -type d -exec chmod 775 {} \;
sudo systemctl reload nginx && curl -I https://example.com
```

Chạy trên origin (không phải qua CDN), thay `example.com` bằng domain của bạn, rồi soi `/var/log/nginx/error.log` nếu vẫn 403.

## Nguyên nhân thật

403 khác với 404 và 500: server **hiểu** request nhưng **từ chối authorize**. Deployment vừa xong là thời điểm dễ rơi vào trạng thái này vì ba lớp thay đổi cùng lúc: file trên disk được thay (ownership, mode, symlink), process web server được reload (config mới, user chạy khác đi), và DNS/CDN bắt đầu trỏ tới bản mới.

Điểm hay bẫy nhất: nginx không cần đọc nội dung file để trả 403. Nó chỉ cần `stat()` thất bại ở **bất kỳ thành phần nào của đường dẫn**. Nếu `/var/www` thiếu bit `x` cho `www-data`, hoặc `index.php` thuộc `root:root` mode `600`, nginx trả 403 ngay cả khi file vẫn tồn tại. Apache thì hay trả 403 khi vào một thư mục không có `DirectoryIndex` và bị `Options -Indexes`. Nói cách khác: 403 sau deploy thường là **lỗi filesystem/config của web server**, không phải lỗi app logic — trừ khi app của bạn tự trả 403 từ middleware auth.

Muốn xác nhận nhanh lớp nào chặn, so sánh response từ origin và từ edge, rồi đọc error log教研 hard evidence:

```bash
tail -n 50 /var/log/nginx/error.log
curl -sS -o /dev/null -w '%{http_code} %{server} %{remote_ip}\n' https://example.com
ps -eo user,pid,cmd | grep -E 'nginx|php-fpm|apache2' | grep -v grep
namei -l /var/www/app/public/index.php
```

`namei -l` là lệnh đáng giá nhất ở đây: nó liệt kê quyền từ `/` tới file, nên bạn thấy ngay đoạn nào mất `x`.

## 4–6 nguyên nhân phổ biến (xếp theo xác suất gặp)

1. **Sai owner/permission sau khi upload artifact** — CI chạy bằng `root`, file nén giữ uid/gid cũ, hoặc `rsync` không có `--chown`. Phổ biến nhất.
2. **Document root hoặc `index` sai** — symlink `current` chưa đổi, `root` trỏ tới thư mục cha, `try_files` thiếu `index.php`; nginx trả 403 thay vì 404 vì truy cập thư mục bị cấm liệt kê.
3. **Rule deny trong config web server** — `deny all`, `Require all denied`, `.htaccess` cũ từ repo, geo/IP allowlist chưa thêm IP egress mới.
4. **Lớp CDN/WAF/cache chặn** — rule bot, reputation IP, stale 403 được cache lại sau lần deploy trước, proxy egress trả 403 thay cho origin.
5. **Container/volume + SELinux/AppArmor** — volume mount `:ro`, container chạy với `user:` không khớp uid host, hoặc label SELinux chưa sửa sau khi bind mount mới.
6. **DNS trỏ nhầm host/default_server** — trỏ tới IP cũ của service khác, SNI sai, virtual host mặc định trả 403 cho mọi domain chưa khai báo; DNS còn đang propagate làm tình huống khó tái hiện.

## Fix từng nguyên nhân

**1. Sửa ownership/permission cho đúng user web server**

```bash
sudo chown -R www-data:www-data /var/www/app
sudo find /var/www/app -type d -exec chmod 755 {} \;
sudo find /var/www/app -type f -exec chmod 644 {} \;
sudo chmod -R ug+rwX /var/www/app/storage /var/www/app/bootstrap/cache
sudo -u www-data test -r /var/www/app/public/index.php && echo OK
```

Nếu PHP-FPM chạy user khác nginx, đọc pool config và chỉnh cho khớp:

```bash
grep -E '^(user|group|listen)' /etc/php/*/fpm/pool.d/www.conf
sudo systemctl restart php-fpm nginx
```

Có ACL thì `chmod` chưa đủ, hãy soi `getfacl -p /var/www/app/public/index.php` và sửa bằng `setfacl -m u:www-data:r-x`.

**2. Kiểm tra document root, index và symlink release**

```bash
ls -ld /var/www/app/current /var/www/app/current/public
sudo nginx -T | grep -nE 'root|index|try_files|autoindex'
```

Config nginx chuẩn cho app PHP:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/app/current/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
    }
}
```

Apache thì kiểm tra `DirectoryIndex index.php` và `AllowOverride` trong `<Directory>`. Đừng bật `autoindex on` chỉ để "cho nó hiện danh sách" — đó là cách vá triệu chứng, để lộ cấu trúc project.

**3. Tìm rule deny đang áp dụng**

```bash
sudo nginx -T | grep -nEi 'deny|allow|satisfy|auth_basic'
grep -RniE 'deny from|require all denied|order allow,deny' /var/www/app/current/public/.htaccess
sudo apachectl -S
```

Sau khi sửa, reload và test từ chính IP bị chặn. Với allowlist, nhớ thêm cả IP của reverse proxy/load balancer, không chỉ IP người dùng cuối.

**4. Tách lớp CDN/WAF ra khỏi origin**

```bash
curl -sSI https://example.com | grep -iE 'server|cf-cache-status|x-cache|via'
curl -sSI --resolve example.com:443:ORIGIN_IP https://example.com
curl -sSI -H 'Cache-Control: no-cache' 'https://example.com/?cb=debug'
```

Nếu origin trả 200 mà edge trả 403, purging cache và review rule WAF là hướng đúng. Nếu `Via:` cho thấy đi qua proxy công ty, thử `curl --noproxy '*' -I ...` để loại trừ egress.

**5. Container, volume và SELinux**

```yaml
services:
  app:
    image: myapp:latest
    user: "www-data:www-data"
    volumes:
      - ./public:/var/www/app/public:z
```

```bash
docker inspect app --format '{{json .Mounts}}'
ls -Z /var/www/app/public/index.php
sudo ausearch -m avc -ts recent | tail -n 20
```

`:z`/`:Z` sửa label SELinux cho bind mount; nếu log AVC hiện `avc: denied { read }`, đó là nguyên nhân, không phải quyền Unix.

**6. DNS, SNI và default_server**

```bash
dig +short example.com A
dig +trace example.com A
curl -sSI --connect-to example.com:443:NEW_IP:443 https://example.com
sudo nginx -T | grep -n 'default_server'
```

Đảm bảo block `server_name example.com;` tồn tại trên host mới. Nếu chỉ block `default_server` trả 403, mọi domain trỏ nhầm vào IP đó sẽ cùng lỗi.

## Edge cases cần soi sâu hơn

- **Proxy env không như bạn nghĩ:** `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` trong shell/CI có thể khiến `curl` và cả health-check đi qua egress proxy, nên 403 phát ra từ proxy chứ không phải hosting. So sánh `env | grep -i proxy`, log `curl -v`, và header `Via`/`X-Forwarded-For`. Đừng kết luận origin hỏng khi chưa test bằng `--noproxy '*'` hoặc từ một mạng khác.
- **Nhiều venv/runtime cùng tồn tại:** deploy bằng `pip install` vào venv A nhưng systemd chạy `ExecStart` trỏ tới venv B; app vẫn load được nhưng middleware auth trả 403 vì config/env khác nhau. Kiểm tra `which gunicorn`, `python -c "import sys; print(sys.prefix)"`, và `systemctl show -p Environment app.service` trước khi đổ lỗi cho filesystem.
- **DNS propagation + cache chồng lên cache:** resolver local còn giữ IP cũ, CDN edge khu vực cũ đã cache 403, trong khi origin đã đúng. Lấy IP từ authoritative NS (`dig @NS...`), so sánh TTL, test qua `dig +trace` và `curl --resolve`, rồi mới purge từng lớp theo thứ tự origin → edge → resolver.

Ở ba tình huống này, AI agent có thể tự debug từng bước: đọc error log, đối chiếu config nginx/systemd, chạy `curl` có kiểm soát, so sánh response origin/edge, rồi đề xuất patch cụ thể thay vì đoán — bạn chỉ cần cung cấp quyền đọc log và môi trường deploy.

## FAQ schema — 3 câu hỏi

**403 sau deploy khác gì 404?**  
404 là không tìm thấy resource; 403 là resource có thể tồn tại nhưng server từ chối truy cập. Sau deploy, 403 thường do quyền, deny rule hoặc document root sai.

**Có phải lúc nào cũng do `chmod 777` chưa đúng không?**  
Không. `777` che giấu sai ownership và vẫn bị SELinux/ACL chặn. Hãy dùng `755` cho thư mục, `644` cho file và chỉ mở ghi cho thư mục cần thiết như `storage/`.

**Bao lâu thì hết 403 sau khi sửa DNS?**  
Tùy TTL. Origin có thể đúng ngay, nhưng resolver và CDN edge giữ bản cũ tới khi hết TTL hoặc bạn purge chủ động.

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "403 sau deploy khác gì 404?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "404 là không tìm thấy resource; 403 là server từ chối authorize. Sau deploy, 403 thường do quyền filesystem, deny rule hoặc document root sai."
      }
    },
    {
      "@type": "Question",
      "name": "Có phải lúc nào cũng do chmod chưa đúng không?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Không. Dùng 755 cho thư mục và 644 cho file là đủ trong đa số trường hợp; SELinux, ACL và ownership sai vẫn gây 403 dù đã chmod."
      }
    },
    {
      "@type": "Question",
      "name": "Bao lâu thì hết 403 sau khi sửa DNS?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Tùy TTL. Origin có thể đúng ngay nhưng resolver và CDN edge giữ bản cũ tới khi hết TTL hoặc được purge."
      }
    }
  ]
}
```

Nếu edge case của bạn vẫn kẹt sau các bước trên, có thể dùng agent debug tự động để lần theo log, cấu hình và response header từng lớp: https://docs.claude.com/en/docs/claude-code/overview

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
