# Error: Git push rejected: 'non-fast-forward' error

## TL;DR — fix nhanh

```bash
git pull --rebase origin <branch>
# giải quyết conflict nếu có, rồi:
git push origin <branch>
```

Nếu commit local là rác và bạn chắc chắn muốn ghi đè: `git push --force-with-lease origin <branch>` (đừng dùng `--force` trần).

## Nguyên nhân thật sự

Remote của bạn có commit mà local chưa có. Git chỉ cho push khi con trỏ branch dịch chuyển "thẳng" — commit mới của bạn phải nối tiếp ngay sau commit mới nhất trên remote. Khi remote đã đi trước (ai đó vừa push, hoặc bạn vừa fetch/rebase), lịch sử hai bên rẽ nhánh. Lúc này push của bạn không phải fast-forward, nên Git từ chối để tránh ghi đè mất commit của người khác.

Nói ngắn gọn: `non-fast-forward` = "local của bạn đang cũ hơn remote, pull về trước đã". Đây là lỗi về thứ tự lịch sử, không phải lỗi quyền hay lỗi mạng.

## 6 nguyên nhân phổ biến (xác suất giảm dần)

1. **Có người khác vừa push lên cùng branch** — phổ biến nhất khi làm team.
2. **Bạn commit ở máy khác / clone khác rồi quay lại máy cũ** push.
3. **Vừa rebase hoặc amend commit đã push** làm SHA thay đổi.
4. **CI/CD hoặc bot tự động commit** (bump version, generate file) lên branch.
5. **Đang ở nhầm branch** và cố push lên branch đã đi trước.
6. **Two venv / hai remote trỏ khác nhau** khiến bạn tưởng đồng bộ nhưng thực ra lệch.

## Fix từng nguyên nhân

### 1. Người khác vừa push — rebase local lên remote

```bash
git fetch origin
git rebase origin/<branch>
# nếu có conflict: sửa file -> git add <file> -> git rebase --continue
git push origin <branch>
```

### 2. Commit từ máy khác — kéo về rồi push

```bash
git pull --rebase origin <branch>
git push origin <branch>
```

### 3. Đã rebase/amend commit đã push — cần force có kiểm soát

```bash
git push --force-with-lease origin <branch>
```

`--force-with-lease` chỉ ghi đè khi remote không đổi so với lần fetch cuối — an toàn hơn `--force` vì chặn việc đè commit mới của người khác.

### 4. Bot/CI tự commit — pull lại rồi push

```bash
git pull --rebase origin <branch>
# kiểm tra lịch sử xem bot commit gì:
git log --oneline -10
git push origin <branch>
```

### 5. Nhầm branch — kiểm tra rồi mới push

```bash
git branch --show-current
git status
# nếu sai branch:
git switch <branch-dung>
git push origin <branch-dung>
```

### 6. Lệch remote / nhiều clone — soi remote thật

```bash
git remote -v
git fetch --all --prune
git log --oneline --graph --all -15
```

Nếu hai remote trỏ hai nơi, sửa lại cho đúng:

```bash
git remote set-url origin git@github.com:org/repo.git
git fetch origin && git rebase origin/<branch> && git push origin <branch>
```

## Nếu vẫn không hết: Edge cases cần soi sâu hơn

- **Proxy env / corp firewall**: `http_proxy`, `https_proxy`, hoặc `no_proxy` cấu hình sai khiến `fetch` chạy nhưng nhận dữ liệu cũ, làm bạn tưởng đã đồng bộ. Soi `git config --global --get http.proxy` và biến môi trường shell.
- **Multiple venvs / nhiều worktree**: bạn đang ở một git worktree hoặc virtualenv khác với chỗ commit thật, nên HEAD bạn thấy không phải HEAD bạn push. Kiểm `git worktree list` và `echo $VIRTUAL_ENV`.
- **DNS propagation / mirror trỏ sai**: remote URL trỏ tới mirror hoặc host cũ chưa cập nhật, SHA trên đó lệch với origin thật. Soi `git remote -v` và phân giải host.

Ba tình huống này thường cần chạy từng lệnh chẩn đoán theo thứ tự và đọc output cẩn thận — một AI agent có thể tự debug từng bước giúp bạn: chạy fetch, so sánh graph, kiểm proxy/worktree, rồi đề xuất lệnh rebase hoặc force an toàn.

## FAQ

**`--force` và `--force-with-lease` khác gì nhau?**
`--force` ghi đè vô điều kiện, có thể xóa mất commit người khác vừa push. `--force-with-lease` chỉ ghi đè nếu remote giữ nguyên so với lần fetch gần nhất, nên an toàn hơn nhiều.

**Rebase hay merge thì tốt hơn khi gặp lỗi này?**
Rebase cho lịch sử thẳng, gọn, dễ đọc; merge giữ nguyên dấu vết rẽ nhánh. Với branch cá nhân, rebase. Với branch chung đã nhiều người dùng, cân nhắc merge để tránh rewrite lịch sử chung.

**Làm sao tránh gặp lại lỗi này?**
Push thường xuyên, `git pull --rebase` trước mỗi lần push, và bật `git config --global pull.rebase true` để mặc định rebase thay vì merge.

---

Nếu edge case của bạn vẫn kẹt sau khi thử hết các bước trên, bạn có thể dùng agent debug tự động để nó chạy và phân tích từng lệnh thay bạn — [dùng thử agent tại đây](https://example.com/agent-debug).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
