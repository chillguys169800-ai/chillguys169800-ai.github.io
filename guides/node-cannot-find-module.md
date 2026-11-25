# Error: Node.js 'Cannot find module' after npm install

## TL;DR — fix nhanh

```bash
rm -rf node_modules package-lock.json
npm install
```

Nếu vẫn lỗi, kiểm tra bạn có đang chạy script từ đúng thư mục chứa `node_modules` không, và đảm bảo `require('...')` trỏ đúng tên package đã khai báo trong `package.json`.

---

## Nguyên nhân thật sự

`Cannot find module 'x'` nghĩa là Node đi qua toàn bộ thuật toán resolution (`require.resolve`) và không tìm thấy file nào khớp. Điểm quan trọng: **lỗi này không nằm ở npm install, mà ở bước resolve của Node.** `npm install` có thể đã chạy thành công, exit code 0, nhưng module vẫn "tàng hình" với process đang chạy vì một trong các lý do bên dưới.

Node resolve theo thứ tự: (1) core module, (2) `node_modules` leo ngược từ thư mục file hiện tại lên root, (3) `NODE_PATH`, (4) global folders. Chỉ cần lệch một mắt xích trong chuỗi đó là lỗi. Và vì resolution phụ thuộc vào **vị trí file gọi `require`**, chứ không phải cwd, nên hai script trong cùng project vẫn có thể hành xử khác nhau.

## 6 nguyên nhân phổ biến (xếp theo xác suất gặp)

1. **Chạy script sai thư mục / sai entry** — `node dist/index.js` trong khi `node_modules` nằm ở project root, hoặc monorepo cài ở root nhưng chạy ở package con.
2. **`node_modules` hỏng hoặc cài thiếu** — install bị ngắt, disk full, hoặc `package-lock.json` lệch với `package.json`.
3. **Package là devDependency nhưng chạy với `NODE_ENV=production`** — `npm install --production` bỏ qua dev deps.
4. **Sai tên hoặc sai subpath export** — `require('lodash/fp')` khi package dùng `exports` field chặn subpath, hoặc typo tên.
5. **Nhiều `node_modules` / nhiều phiên bản Node** — nvm đổi version, global module không thấy được từ local.
6. **Symlink / workspace** — `npm link`, yarn workspaces, pnpm: module nằm ở `.pnpm` store, resolve qua symlink bị gãy.

## Fix từng nguyên nhân

### 1. Chạy sai thư mục

Luôn chạy script bằng relative path từ root, hoặc dùng npm script để Node tự set cwd:

```json
// package.json
{
  "scripts": {
    "start": "node src/index.js"
  }
}
```

```bash
npm start   # thay vì node src/index.js từ thư mục khác
```

Kiểm tra Node đang resolve từ đâu:

```bash
node -e "console.log(require.resolve.paths('express'))"
```

### 2. node_modules hỏng — cài lại sạch

```bash
rm -rf node_modules package-lock.json
npm cache verify
npm install
```

Nếu dùng npm >= 7 và vẫn lỗi, thử `npm ci` để cài đúng theo lock:

```bash
npm ci
```

### 3. NODE_ENV=production nuốt devDependencies

```bash
# xem env hiện tại
echo $NODE_ENV

# cài đủ cả dev deps
NODE_ENV=development npm install
```

Hoặc cài trực tiếp package đang thiếu:

```bash
npm install --save-dev typescript ts-node
```

### 4. Sai subpath / package dùng exports field

Kiểm tra package có export đúng đường dẫn bạn require không:

```bash
node -e "console.log(require.resolve('lodash/fp'))"
cat node_modules/lodash/package.json | grep -A5 '"exports"'
```

Nếu package chặn subpath, dùng entry chính:

```js
// sai nếu exports chặn subpath
const fp = require('lodash/fp');

// đúng — dùng import chính
const _ = require('lodash');
```

### 5. Nhiều node_modules / đổi Node version

```bash
node -v                 # version đang chạy
which node              # binary nào
npm root -g             # global modules ở đâu
nvm current             # nếu dùng nvm
```

Cài lại native/hoặc cài package vào đúng version:

```bash
nvm use 20
npm rebuild
```

### 6. Symlink / workspace gãy

```bash
# với npm workspaces
npm install --workspaces --include-workspace-root

# kiểm tra symlink
ls -la node_modules/your-package
```

Với pnpm, đừng trộn `npm install` — dùng đúng package manager của repo:

```bash
pnpm install
```

## Edge cases cần soi sâu hơn

Ba tình huống dưới đây thường không giải quyết được bằng fix nhanh, vì chúng phụ thuộc môi trường chứ không phải code:

- **Proxy / registry env**: `HTTP_PROXY`, `HTTPS_PROXY`, `npm config get registry` trỏ sai khiến install "thành công giả" (tải từ cache rỗng). Cần soi `~/.npmrc`, `.npmrc` trong project, và log `npm install --verbose`.
- **Multiple venvs / multiple Node versions song song**: nvm + volta + system node chồng nhau, mỗi shell một version, module cài ở version này không thấy ở version kia. Cần soi `PATH` order và shell rc.
- **DNS propagation / registry 私 mirror**: custom registry chưa sync package mới, hoặc DNS trỏ tới mirror cũ. Cần so `npm view <pkg> versions` với registry công khai.

Nếu bạn kẹt ở một trong ba edge case này, một AI agent có thể tự chạy từng bước debug giúp bạn: đọc `NODE_PATH`, so sánh `require.resolve.paths`, kiểm tra symlink, và thử install trong sandbox sạch để cô lập nguyên nhân — thay vì bạn phải đoán.

## FAQ

**Tại sao `npm install` báo thành công mà Node vẫn báo Cannot find module?**
Vì `npm install` chỉ đảm bảo file nằm trong `node_modules`; còn Node resolve dựa trên vị trí file gọi `require` và `NODE_PATH`. Hai thứ này lệch nhau thì vẫn lỗi dù install exit code 0.

**Xóa `node_modules` rồi cài lại có mất gì không?**
Không, `node_modules` tái tạo được từ `package.json` + lock file. Đây là cách reset an toàn và nhanh nhất cho 80% trường hợp.

**Sửa `NODE_PATH` có phải cách đúng không?**
Không nên xem là fix chính. `NODE_PATH` chỉ che giấu vấn đề resolve sai thư mục. Fix đúng là chạy script từ đúng entry hoặc dùng npm script.

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
