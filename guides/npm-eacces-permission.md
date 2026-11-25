# Error: npm ERR! EACCES permission denied (global install)

**TL;DR — fix nhanh** (chạy bằng user thường, **không** dùng `sudo npm`):

```bash
mkdir -p "$HOME/.npm-global/bin"
npm config set prefix "$HOME/.npm-global"
grep -q '.npm-global/bin' ~/.bashrc || echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc; source ~/.bashrc
```

Dùng zsh thì thay `~/.bashrc` bằng `~/.zshrc`. Kiểm tra lại bằng `npm config get prefix` — phải trỏ về `$HOME/.npm-global`.

## Nguyên nhân thật: npm không được phép ghi vào prefix global

`npm install -g` không cài vào `node_modules` của project. Nó ghi vào **prefix global**: thường là `/usr/local` trên Linux/macOS khi Node được cài bằng apt/installer, hoặc một đường dẫn hệ thống do `.npmrc` chỉ định. User thường có quyền đọc thư mục đó nhưng **không có quyền ghi** vào `lib/node_modules`, `bin`, hoặc `~/.npm` (cache). Kernel trả `EACCES`, npm in ra dòng bạn thấy.

Hai điểm hay bị nhầm:

- `EACCES` là lỗi **ghi file**, không phải lỗi mạng hay registry.
- `command not found` sau khi cài là lỗi **PATH**, không phải EACCES. Sửa prefix xong mà vẫn không chạy được lệnh thì bạn đang dính thêm lỗi PATH.

`sudo npm install -g` “chạy được” nhưng để lại file thuộc `root` trong `~/.npm`. Lần sau bạn chạy npm bằng user thường lại tiếp tục EACCES — vì vậy đây là cách chữa triệu chứng, không chữa nguyên nhân.

## 6 nguyên nhân phổ biến, xếp theo xác suất gặp

1. **Prefix global trỏ tới thư mục hệ thống** (`/usr/local`, `/usr`) do cài Node qua apt/official installer.
2. **`~/.npm` bị root chiếm quyền** sau một lần từng chạy `sudo npm ...`.
3. **`.npmrc` ghi đè prefix** ở user/project/global, trỏ sang thư mục read-only hoặc thuộc user khác.
4. **Chạy trong Docker/CI với user không khớp owner của volume mount**.
5. **Filesystem đặc biệt**: mount read-only, `chattr +i`, SELinux/AppArmor chặn ghi dù `ls -l` nhìn có vẻ ổn.
6. **Nhiều runtime Node cùng lúc** (nvm + apt + Corepack/Volta): shell này thấy prefix này, shell kia thấy prefix khác.

## Fix từng nguyên nhân

### 1. Đổi prefix về thư mục của bạn

```bash
PROFILE="$HOME/.bashrc"; [ -n "$ZSH_VERSION" ] && PROFILE="$HOME/.zshrc"

npm config get prefix
mkdir -p "$HOME/.npm-global/bin" "$HOME/.npm-global/lib/node_modules"
npm config set prefix "$HOME/.npm-global"
grep -q '.npm-global/bin' "$PROFILE" || echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> "$PROFILE"
source "$PROFILE"

npm config get prefix
npm install -g typescript
```

### 2. Lấy lại quyền cache `~/.npm`

```bash
mkdir -p "$HOME/.npm" "$HOME/.npm-global"
sudo chown -R "$(id -u):$(id -g)" "$HOME/.npm" "$HOME/.npm-global"
npm cache verify
npm install -g typescript
```

Không cần `chmod 777`; chỉ cần owner đúng.

### 3. Dò và xóa override trong `.npmrc`

```bash
npm config list -l | grep -i prefix || true

for f in "$HOME/.npmrc" "$PWD/.npmrc" "$(npm root -g)/.npmrc"; do
  [ -f "$f" ] && echo "--- $f" && sed -n '1,80p' "$f"
done

npm config delete prefix --location=project 2>/dev/null || true
npm config set prefix "$HOME/.npm-global" --location=user
npm config get prefix
npm install -g typescript
```

### 4. Docker/CI: map đúng uid:gid

```bash
sudo chown -R "$(id -u):$(id -g)" . 2>/dev/null || true

docker run --rm -it \
  -v "$PWD:/workspace" \
  -u "$(id -u):$(id -g)" \
  -w /workspace \
  node:lts-alpine sh
```

Trong container:

```bash
mkdir -p /workspace/.npm-global/bin
npm config set prefix /workspace/.npm-global
export PATH="/workspace/.npm-global/bin:$PATH"
npm install -g typescript
```

### 5. Soi filesystem/SELinux/AppArmor

```bash
df -T "$HOME" "$HOME/.npm-global" 2>/dev/null
lsattr -d "$HOME/.npm-global" 2>/dev/null || true
sudo chattr -i "$HOME/.npm-global" 2>/dev/null || true

getenforce 2>/dev/null || true
sudo restorecon -Rv "$HOME/.npm" "$HOME/.npm-global" 2>/dev/null || true
sudo aa-status 2>/dev/null | head -20 || true

npm install -g typescript
```

Nếu mount đang read-only, dời prefix sang path ghi được:

```bash
P="${TMPDIR:-/tmp}/npm-global"
mkdir -p "$P/bin" "$P/lib/node_modules"
npm config set prefix "$P"
export PATH="$P/bin:$PATH"
npm install -g typescript
```

### 6. Đồng nhất runtime khi có nvm/fnm/Volta

```bash
which -a node npm
echo "$PATH" | tr ':' '\n' | nl -ba

npm config delete prefix --location=user 2>/dev/null || true
npm config delete prefix --location=project 2>/dev/null || true

if [ -s "$HOME/.nvm/nvm.sh" ]; then
  . "$HOME/.nvm/nvm.sh"
  nvm use --lts
else
  echo "Không dùng nvm; quay lại fix 1 hoặc 4."
fi

command -v fnm >/dev/null && eval "$(fnm env --use-on-cd)" || true
command -v volta >/dev/null && volta which node || true

npm config get prefix
npm install -g typescript
```

## Edge cases cần soi sâu hơn

1. **Proxy env không khớp npm config.** `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` trong shell có thể khác `npm config get proxy`. Log thường trộn lỗi registry với lỗi cache write, dễ chẩn đoán nhầm thành EACCES. AI agent có thể tự so sánh `env | grep -i proxy` với `npm config list -l`, rồi chạy thử `npm view typescript version --loglevel=http` để tách riêng từng lớp.

2. **Multiple venvs / multiple Node runtimes.** Project dùng nvm, CI dùng apt Node, Corepack lại thêm một shim nữa — cùng một lệnh `npm i -g` chạy ở hai shell cho hai prefix khác nhau. Agent có thể tự đọc `~/.bashrc`, `~/.zshrc`, `.npmrc`, `which -a npm`, `ls -ld "$(npm config get prefix)"` và sửa đúng file cấu hình đang được shell thực sự load.

3. **DNS propagation / mirror nội bộ vừa đổi domain.** DNS không trực tiếp gây EACCES, nhưng khi registry chuyển sang Artifactory/Nexus mới, cache metadata cũ còn trỏ tới path mirror read-only nên log hiện cả EACCES lẫn `EAI_AGAIN`. Agent có thể tự kiểm tra `npm config get registry`, `curl -I`, `dig`, làm sạch cache có chọn lọc rồi cài lại để xác nhận nguyên nhân.

Nếu bạn muốn agent tự lần ra prefix/cache/PATH thực tế trên máy của mình thay vì đoán thủ công, nói rõ shell + hệ điều hành + output `npm config list -l` là đủ để nó debug từng bước.

## FAQ schema

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Vì sao không nên dùng sudo npm install -g để hết EACCES?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "sudo cho phép ghi lần đó nhưng tạo file và cache thuộc root. Lần chạy npm tiếp theo bằng user thường sẽ lại bị EACCES. Cách đúng là đổi npm prefix sang thư mục thuộc về user hiện tại."
      }
    },
    {
      "@type": "Question",
      "name": "Đổi prefix xong vẫn báo EACCES thì kiểm tra gì tiếp?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Kiểm tra ~/.npm có file thuộc root không, .npmrc có dòng prefix nào ghi đè không, và volume/container có đang mount read-only hoặc sai uid:gid không."
      }
    },
    {
      "@type": "Question",
      "name": "EACCES khác gì với command not found sau khi cài global?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "EACCES là lỗi quyền ghi trong lúc cài. command not found là lỗi PATH: package đã cài nhưng thư mục bin của prefix chưa nằm trong PATH của shell hiện tại."
      }
    }
  ]
}
```

Nếu edge case của bạn vẫn kẹt sau các lệnh trên, có thể dùng agent debug tự động để lần ra prefix/cache/PATH thực tế từ log và shell của bạn: https://claude.com/claude-code

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
