# Error: CORS policy: No 'Access-Control-Allow-Origin' header

## TL;DR — fix nhanh

Giả sử lỗi đến từ API Express chưa khai báo origin được phép (nguyên nhân gặp nhiều nhất):

```js
const cors = require('cors');
app.use(cors({ origin: 'https://app.example.com', methods: ['GET','POST','PUT','DELETE'], allowedHeaders: ['Content-Type','Authorization'] }));
app.options('*', cors());
```

Nếu dùng cookie/token trong header, đọc thêm mục 3 ở dưới — wildcard `*` sẽ không đủ.

## Nguyên nhân thật: browser chặn response, không phải request bị chặn

CORS là cơ chế của **browser**, không phải của server và cũng không phải lỗi mạng. Mọi request vẫn đi tới API, server vẫn xử lý, thậm chí vẫn ghi DB — chỉ là browser từ chối đọc response vì thiếu `Access-Control-Allow-Origin` (ACAO) khớp với `Origin` của trang.

`Origin` = **scheme + host + port**. `https://app.example.com` và `https://www.app.example.com` là hai origin khác nhau; `http://localhost:5173` và `http://localhost:3000` cũng khác nhau. Browser gửi header `Origin: https://app.example.com`, server phải trả về đúng giá trị đó (hoặc `*` cho request không có credential). Nếu không khớp, console báo đúng câu ở tiêu đề.

Điểm hay gây hiểu lầm: với request "không đơn giản" (`PUT`, `DELETE`, `Content-Type: application/json`, header `Authorization`...), browser gửi thêm request `OPTIONS` (preflight) **trước** request thật. Nếu `OPTIONS` trả 405/404 hoặc thiếu `Access-Control-Allow-Methods/Headers`, request thật không bao giờ được gửi, và lỗi vẫn hiện là "No 'Access-Control-Allow-Origin' header" — nên bạn dễ tưởng lỗi nằm ở endpoint `GET`/`POST`.

`curl` chạy được là chuyện bình thường: curl không tự gửi `Origin` và không áp same-origin policy. Muốn reproduce đúng, phải thêm `Origin` thủ công (xem mục 4).

## 4–6 nguyên nhân phổ biến, xếp theo xác suất gặp

1. **Backend chưa set ACAO, hoặc set sai origin** (thừa `/` cuối, nhầm `http` ↔ `https`, nhầm `www`, quên port dev). Cao nhất,尤其是 lúc mới tách frontend/back-end.
2. **Preflight `OPTIONS` không được xử lý** — route chỉ đăng ký `GET/POST`, middleware auth trả 401 cho `OPTIONS`, hoặc gateway trả 405.
3. **Dùng `*` cùng `credentials: 'include'` / cookie / `Authorization`** — browser từ chối wildcard khi có credential; cần origin cụ thể + `Access-Control-Allow-Credentials: true`.
4. **Header bị strip hoặc bị nhân đôi ở lớp trung gian** — nginx/CDN thêm ACAO trong khi app cũng thêm, hoặc proxy xóa header. Browser coi hai giá trị ACAO là invalid.
5. **Redirect trước khi tới API** — `api.example.com` → `www.api.example.com` hoặc `http` → `https`. Browser **không follow redirect cho preflight**, nên `OPTIONS` bị 301/302 là fail ngay.
6. **Origin động / cache sai** — whitelist hardcode một origin, nhiều subdomain/dev port; response được cache mà thiếu `Vary: Origin` nên origin này nhận header của origin kia.

## Fix từng nguyên nhân

### 1. Set đúng origin, có allowlist

```bash
npm init -y && npm i express
```

```js
// server.js — chạy: node server.js
const express = require('express');
const app = express();
const ALLOW = new Set(['https://app.example.com', 'http://localhost:5173']);

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && ALLOW.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Vary', 'Origin');
  }
  next();
});

app.get('/api/me', (req, res) => res.json({ ok: true }));
app.listen(3000, () => console.log('API on :3000'));
```

Giữ nguyên chuỗi: không có trailing slash, đúng scheme, đúng port. Log `req.headers.origin` một lần để thấy browser đang gửi giá trị gì.

### 2. Xử lý preflight `OPTIONS`

```js
// Đặt sau middleware CORS ở mục 1
app.options('/api/*', (req, res) => {
  const origin = req.headers.origin;
  if (origin && ALLOW.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,PATCH,DELETE,OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type,Authorization,X-Request-Id');
    res.setHeader('Access-Control-Max-Age', '600');
    res.setHeader('Vary', 'Origin');
  }
  res.sendStatus(204); // trả 2xx, tuyệt đối không redirect
});
```

Nếu dùng nginx làm lớp trước:

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;
    ssl_certificate     /etc/ssl/certs/api.example.com.pem;
    ssl_certificate_key /etc/ssl/private/api.example.com.key;

    location /api/ {
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin "$http_origin" always;
            add_header Access-Control-Allow-Methods "GET,POST,PUT,PATCH,DELETE,OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type,Authorization,X-Request-Id" always;
            add_header Access-Control-Max-Age 600 always;
            return 204;
        }
        add_header Access-Control-Allow-Origin "$http_origin" always;
        add_header Vary "Origin" always;
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
    }
}
```

### 3. Credential: bỏ wildcard, thêm flag

```js
// Frontend
fetch('https://api.example.com/api/me', {
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
});
```

```js
// Backend — origin phải cụ thể, KHÔNG được là '*'
app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && ALLOW.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Vary', 'Origin');
  }
  next();
});
```

### 4. Soi header thật trước khi sửa code

```bash
# Response thường
curl -i -H 'Origin: https://app.example.com' https://api.example.com/api/me | head -n 30

# Preflight
curl -i -X OPTIONS \
  -H 'Origin: https://app.example.com' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type,authorization' \
  https://api.example.com/api/me
```

Nếu thấy hai dòng `access-control-allow-origin`, chọn **một** nơi set: bỏ `cors()` trong app nếu nginx đã add, hoặc bỏ `add_header` trong nginx nếu app đã add.

### 5. Dẹt redirect và lệch origin

```nginx
# Chỉ redirect ở HTTP; endpoint HTTPS không redirect sang www
server {
    listen 80;
    server_name api.example.com;
    return 301 https://api.example.com$request_uri;
}
```

```js
// Frontend: dùng đúng base URL như whitelist, không ghép Acid từ location.host
const API_BASE = 'https://api.example.com';
```

### 6. Whitelist động + cache đúng

```js
const isAllowed = (o = '') =>
  /^https:\/\/[a-z0-9-]+\.example\.com$/.test(o) || o === 'http://localhost:5173';

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && isAllowed(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Vary', 'Origin');
  }
  next();
});
```

`Vary: Origin` bắt buộc nếu có CDN/cache,否则 response cho origin A sẽ bị trả cho origin B.

## Edge cases cần soi sâu hơn

- **Proxy env (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`)**: browser không đọc biến môi trường, nhưng server-side fetch và CLI thì có. Corporate proxy có thể thêm/sửa header hoặc đẩy request qua egress khác, khiến `Origin` bạn log ở app không phải `Origin` bạn thấy trên DevTools. So sánh `env | grep -i proxy`, `curl --noproxy '*' ...`, và log `req.headers.origin` tại chính process nhận request. AI agent có thể tự lần từ env → proxy config → log server → response header, từng bước một, để chỉ ra lớp nào đổi header.
- **Multiple venvs / nhiều runtime cùng máy**: bạn sửa middleware trong venv A nhưng `systemd`/PM2 đang chạy interpreter ở venv B, nên file cấu hình mới không bao giờ được load. Kiểm tra `which python`, `python -c "import starlette; print(starlette.__file__)"`, `pip show fastapi`, rồi đối chiếu với `ps -ef | grep uvicorn`. Agent có thể tự trace process → interpreter → package thực tế đang import và đoạn config nào đang có hiệu lực.
- **DNS propagation + CDN cache**: đổi origin trong whitelist rồi vẫn lỗi ở một số region vì POP còn giữ response cũ, hoặc bản ghi mới chưa lan tới resolver. Soi `dig api.example.com`, `curl -I` lấy `Cache-Control`/`Age`, thử `?cb=<value>` để bypass cache, rồi purge CDN. Agent có thể tự so sánh response theo region/resolver và结论 lại là DNS, cache, hay code.

## FAQ schema

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Tại sao curl gọi API được nhưng browser báo No 'Access-Control-Allow-Origin' header?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "curl không gửi header Origin và không áp same-origin policy, còn browser thì có. Để reproduce đúng, thêm -H 'Origin: https://app.example.com' vào lệnh curl và kiểm tra response header."
      }
    },
    {
      "@type": "Question",
      "name": "Có dùng Access-Control-Allow-Origin: * khi gửi cookie được không?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Không. Với credentials: 'include', cookie hoặc Authorization, browser yêu cầu origin cụ thể và Access-Control-Allow-Credentials: true. Wildcard bị từ chối."
      }
    },
    {
      "@type": "Question",
      "name": "Có fix lỗi CORS thuần ở frontend fetch được không?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Không thể bỏ qua từ phía browser. Frontend chỉ giảm preflight (dùng request đơn giản, khớp origin) hoặc gọi cùng origin qua reverse proxy; header ACAO vẫn phải do server hoặc proxy trả về."
      }
    }
  ]
}
```

Nếu edge case của bạn vẫn kẹt sau các bước trên, có thể dùng agent debug tự động để lần từ request, proxy, config tới response header: [Claude Code](https://docs.claude.com/en/docs/claude-code).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
