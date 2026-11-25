# Error: Website broken on mobile but fine on desktop

## TL;DR — fix nhanh

3 dòng này giải quyết đa số trường hợp. Dán lần lượt vào `<head>` và file CSS đầu tiên được load:

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```
```css
*,*::before,*::after{box-sizing:border-box}
img,video,iframe,canvas{max-width:100%;height:auto}
```

Nếu vẫn vỡ, nhảy tới mục **Nguyên nhân thật** — gần như chắc chắn bạn đang fix nhầm lớp.

---

## Nguyên nhân thật

Desktop và mobile không render cùng một tài liệu theo cùng một cách. Trên desktop, **layout viewport** chính là cửa sổ trình duyệt. Trên mobile, nếu thiếu `<meta name="viewport">`, browser giả vờ một layout viewport rộng hơn màn hình rồi **co toàn bộ trang lại** để vừa màn hình. Kết quả: media query vẫn thấy "màn hình to", chữ thì li ti, nút bấm thì lệch. Desktop "fine" thực ra là "được tha thứ", không phải đúng.

Khi bạn thêm viewport meta, mọi thứ đổi chiều: layout viewport giờ bằng chiều rộng thiết bị. Cái `width: 1200px`, `min-width: auto` của flex item, `100vw`, hay `position: fixed` trước đây bị co lại thì giờ **trở thành overflow thật**. Chữ to lên, grid không còn chỗ, thanh ngang xuất hiện.

Lớp thứ hai hay bị bỏ qua: **layout viewport ≠ visual viewport**. Khi bàn phím mở, khi pinch-zoom, khi Safari dùng `vh` khác `dvh`, phần tử cố định chiều cao sẽ đẩy nội dung ra ngoài màn hình. Thêm `env(safe-area-inset-*)` trên iPhone là chuyện thường.

Lớp thứ ba: build pipeline. CSS phục vụ cho desktop có thể khác CSS mobile do CDN cache, service worker cũ, hoặc class bị purge sai. Nên debug theo thứ tự: viewport → overflow → CSS actual → JS → cache.

Trước khi sửa, mở responsive mode và đi lần lượt các chiều rộng **389px, 335px, 426px, 400px, 354px**. Nếu lỗi chỉ xuất hiện ở một vài giá trị, bạn đang dính overflow do nội dung, không dính breakpoint.

---

## 4–6 nguyên nhân phổ biến, xếp theo xác suất gặp

1. Thiếu/sai viewport meta — cao nhất.
2. Overflow ngang do `min-width: auto` trong flex/grid và nội dung không co được.
3. `width` cố định, `100vw`, `vh`, quên safe-area.
4. Media query/cascade/build output không như bạn nghĩ.
5. JS chỉ tính toán lúc `resize`, không lắng nghe `visualViewport`/xoay màn hình.
6. Service worker / cache / mixed-content chặn bản CSS mới trên mobile.

---

## Fix từng nguyên nhân

### 1. Viewport meta

```html
<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>App</title>
  <link rel="stylesheet" href="/assets/app.css">
</head>
<body>
  <!-- ... -->
</body>
</html>
```

Đặt nó **trước** mọi CSS/JS. Đừng dùng `user-scalable=no`: vừa hại accessibility, vừa không sửa được overflow.

### 2. Overflow do flex/grid

```css
*,*::before,*::after{box-sizing:border-box}

.row{display:flex;gap:1rem;min-width:0}
.row > *{min-width:0}

.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(min(100%, 16rem),1fr))}

.prose{overflow-wrap:anywhere}
pre{white-space:pre-wrap;overflow-wrap:anywhere}

.table-wrap{overflow-x:auto;-webkit-overflow-scrolling:touch}
```

`min-width: 0` là chìa khóa: flex item mặc định không chịu co dưới kích thước nội dung. Tìm thủ phạm bằng console:

```js
[...document.querySelectorAll('*')]
  .filter(el => el.getBoundingClientRect().right > document.documentElement.clientWidth + 1)
  .slice(0, 12)
  .map(el => `${el.tagName.toLowerCase()}.${el.className} → ${Math.round(el.getBoundingClientRect().right)}px`);
```

### 3. Fixed width / `100vw` / `vh` / safe area

```css
html{max-width:100%;overflow-x:clip}

.hero{
  width:min(100%, 72rem);
  margin-inline:auto;
  padding-inline:max(1rem, env(safe-area-inset-left), env(safe-area-inset-right));
}

.modal{
  width:min(100vw, 32rem);
  height:100dvh;
  padding-bottom:calc(1rem + env(safe-area-inset-bottom));
}
```

Đừng vội `overflow-x:hidden` trên `body`: nó giấu triệu chứng, có thể phá `position:sticky`. Dùng `clip` tạm thời để cô lập, rồi vẫn sửa phần tử gây tràn.

### 4. Media query và thứ tự CSS

```css
/* mobile-first: base là mobile */
.card{padding:1rem;grid-template-columns:1fr}

@media (min-width: 48rem){
  .card{padding:2rem;grid-template-columns:repeat(2,1fr)}
}
```

Kiểm tra output thật, không phải source:

```bash
grep -R "min-width\|@media\|width:" -n dist/assets/*.css | head -40
```

Nếu dùng Tailwind, soi lại `content` glob; nếu dùng CSS Modules, soi class bị purge. Nhớ kiểm tra thứ tự `<link>`: file override load trước file base sẽ bị ghi đè ngược.

### 5. JS chỉ nghe `resize`

```js
const setAppHeight = () => {
  const h = window.visualViewport?.height ?? window.innerHeight;
  document.documentElement.style.setProperty('--app-h', `${h}px`);
};

setAppHeight();
window.visualViewport?.addEventListener('resize', setAppHeight);
window.addEventListener('orientationchange', () => requestAnimationFrame(setAppHeight));
```

```css
.app{min-height:var(--app-h, 100dvh)}
```

`visualViewport` là phần nhìn thấy sau khi bàn phím/zoom; `innerHeight` là layout viewport. Dùng nhầm là modal bị cụt chân trên iOS.

### 6. Service worker / cache

```js
// sw.js
self.addEventListener('install', e => self.skipWaiting());
self.addEventListener('activate', e => e.waitUntil(self.clients.claim()));

self.addEventListener('fetch', e => {
  if (e.request.mode === 'navigate') {
    e.respondWith(fetch(e.request).catch(() => caches.match(e.request)));
  }
});
```

Trong DevTools: **Network → Disable cache**, rồi **Application → Service Workers → Unregister**. Test lại bằng private window để loại cookie/Storage.

---

## Edge cases cần soi sâu hơn

- **Proxy env**: biến `HTTP_PROXY`/`HTTPS_PROXY` hoặc proxy reverse sửa header `User-Agent`, làm CDN trả bản CSS khác cho mobile. Kèm theo là mixed-content/HSTS khiến asset bị chặn chỉ trên 4G. AI agent có thể tự debug từng bước: in env, so sánh `curl -I` qua và không qua proxy, đối chiếu response `Content-Encoding` và `Cache-Control`.
- **Multiple venvs**: nhiều môi trường (Node version, Python venv, corepack/pnpm) khiến build mobile chạy PostCSS/Tailwind khác với build desktop — class sinh ra khác nhau dù source như nhau. Agent có thể tự kiểm tra `which node`, `node -v`, `npm ls`, `pnpm why`, rồi build lại trong cùng shell với `rm -rf dist` để so sánh hash CSS.
- **DNS propagation**: `www` và apex trỏ khác bản, TTL cũ còn sống ở resolver di động, CDN edge theo khu vực phục vụ bundle cũ. Agent có thể tự lần theo `dig +short`, so sánh từng resolver, kiểm tra `CNAME` và trạng thái purge CDN trước khi kết luận lỗi là CSS.

---

## FAQ schema

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Vì sao desktop hiển thị đúng nhưng mobile bị vỡ dù cùng một URL?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Vì mobile dùng layout viewport khác desktop. Nếu thiếu meta viewport, browser co cả trang lại nên lỗi bị che; khi có meta viewport, mọi width cố định và min-width auto trở thành overflow thật."
      }
    },
    {
      "@type": "Question",
      "name": "Có nên dùng overflow-x: hidden trên body để hết thanh cuộn ngang không?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Không nên dùng như fix cuối cùng. Nó chỉ giấu triệu chứng và có thể làm hỏng sticky positioning. Hãy tìm phần tử vượt quá clientWidth bằng script đo getBoundingClientRect, sửa min-width: 0 hoặc width: min(100%, ...)."
      }
    },
    {
      "@type": "Question",
      "name": "Nên dùng 100vh hay 100dvh cho modal trên mobile?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Dùng 100dvh cho phần tử cần theo chiều cao nhìn thấy hiện tại, kết hợp visualViewport trong JS và env(safe-area-inset-bottom) cho iPhone. 100vh thường giữ chiều cao cũ khi bàn phím mở."
      }
    }
  ]
}
</script>
```

Nếu edge case của bạn vẫn kẹt sau các bước trên, có thể để agent debug tự động từng bước tại [bật agent debug](/agent-debug).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
