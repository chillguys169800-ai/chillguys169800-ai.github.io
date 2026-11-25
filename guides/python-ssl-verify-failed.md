# Error: SSL certificate verify failed — Python requests

## TL;DR — fix nhanh

```python
import requests
r = requests.get("https://example.com", verify="/path/to/cacert.pem")  # chỉ định CA bundle
r.raise_for_status()
```

Nếu chỉ cần chạy tạm trong môi trường dev (KHÔNG dùng production):

```python
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
r = requests.get("https://example.com", verify=False)
```

Cách đúng lâu dài: cập nhật `certifi` (`pip install -U certifi`) rồi trỏ `REQUESTS_CA_BUNDLE` tới bundle của bạn.

## Nguyên nhân thật sự

`requests` không tự tin vào certificate của server. Khi bắt tay TLS, client phải xác thực chuỗi cert về tới một Root CA mà nó tin. `requests` dùng bundle CA đi kèm thư viện `certifi`. Lỗi `SSLError: certificate verify failed` nghĩa là quá trình xác thực chuỗi thất bại — không tìm được issuer tin cậy, cert đã hết hạn, hostname không khớp, hoặc máy bạn đang trỏ nhầm một bundle rỗng/cũ.

Đây là lỗi xác thực, không phải lỗi mạng. Kết nối TCP đã thiết lập, TLS đã bắt tay, nhưng bước verify chủ động từ chối. Điểm mấu chốt: fix đúng nghĩa là cho client một chuỗi tin cậy hợp lệ, chứ không phải tắt verify.

## 4 nguyên nhân phổ biến (xếp theo xác suất)

1. **`certifi` cũ / thiếu Root CA mới** — máy cài từ lâu, bundle không có CA mới ký cert server.
2. **Corporate proxy / TLS interception** — công ty chèn cert của proxy vào luồng, cert đó không có trong bundle mặc định.
3. **Self-signed hoặc internal CA** — server nội bộ dùng cert tự ký hoặc CA riêng của tổ chức.
4. **Hostname mismatch / cert hết hạn** — CN/SAN không khớp domain, hoặc `notAfter` đã qua.
5. **Env var trỏ sai bundle** — `REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE` trỏ tới file rỗng hoặc sai định dạng.
6. **Hệ điều hành thiếu CA ở tầng OpenSSL** — container slim (alpine, distroless) không có `ca-certificates`.

## Fix từng nguyên nhân

### 1. `certifi` cũ

```bash
python -m pip install --upgrade certifi requests
python -c "import certifi; print(certifi.where())"
```

```python
import requests
# requests tự dùng certifi mới nhất sau khi upgrade
r = requests.get("https://example.com", timeout=10)
r.raise_for_status()
print(r.status_code)
```

### 2. Corporate proxy / TLS interception

Xuất cert của proxy từ Keychain (macOS) hoặc `/etc/ssl/certs` (Linux), gộp với bundle certifi:

```bash
# Linux: gộp CA hệ thống + certifi thành một bundle
cat /etc/ssl/certs/ca-certificates.crt "$(python -c 'import certifi;print(certifi.where())')" > /tmp/combined.pem
export REQUESTS_CA_BUNDLE=/tmp/combined.pem
```

```python
import os, requests
session = requests.Session()
session.verify = os.environ["REQUESTS_CA_BUNDLE"]  # dùng bundle đã gộp
r = session.get("https://internal-api.corp.local", timeout=10)
r.raise_for_status()
```

### 3. Self-signed / internal CA

```python
import requests
r = requests.get(
    "https://internal.example.local/health",
    verify="/etc/ssl/certs/internal-ca.pem",  # trỏ thẳng CA nội bộ
    timeout=10,
)
r.raise_for_status()
print(r.json())
```

### 4. Hostname mismatch / cert hết hạn

Kiểm tra trước khi kết luận:

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Nếu SAN không chứa domain bạn gọi → đây là lỗi cấu hình server, sửa cert phía server chứ không phải client. Không có workaround an toàn ở client ngoài việc gọi đúng hostname khớp SAN.

### 5. Env var trỏ sai bundle

```python
import os, certifi, requests
os.environ.pop("REQUESTS_CA_BUNDLE", None)   # xoá biến hỏng
os.environ.pop("SSL_CERT_FILE", None)
os.environ["REQUESTS_CA_BUNDLE"] = certifi.where()  # set lại đúng chỗ
r = requests.get("https://example.com", timeout=10)
r.raise_for_status()
```

### 6. OS thiếu `ca-certificates` (container)

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python", "app.py"]
```

## Nếu vẫn không hết: Edge cases cần soi sâu hơn

- **Proxy env + NO_PROXY đánh nhau**: biến `HTTP_PROXY`/`HTTPS_PROXY` khiến `requests` đi qua proxy nhưng cert proxy không nằm trong bundle; hoặc `NO_PROXY` loại nhầm domain nội bộ. Cần trace luồng thực tế bằng `requests` hooks và log `session.verify` tại thời điểm gọi.
- **Multiple venvs / nhiều Python**: `pip` bạn upgrade nằm ở venv này, nhưng app chạy bằng venv khác hoặc system Python, nên `certifi.where()` vẫn trỏ bundle cũ. Phải xác định đúng interpreter (`sys.executable`) và bundle tương ứng.
- **DNS propagation / SNI sai sau khi đổi hạ tầng**: domain mới trỏ tới LB nhưng cert trên LB chưa khớp SNI, hoặc DNS còn cache trỏ host cũ với cert khác. Cần soi từng bước: resolve IP → bắt tay TLS với đúng SNI → so cert trả về.

Ba tình huống này đều debug được theo từng bước có kiểm chứng (in cert thực tế nhận về, so issuer, kiểm tra interpreter, trace proxy). Nếu bạn muốn, một AI agent có thể tự chạy và phân tích từng bước này giúp bạn thay vì mò tay.

## FAQ

**Có nên dùng `verify=False` không?**
Không, trừ khi debug cục bộ. Nó tắt hoàn toàn xác thực, mở cửa cho MITM. Production luôn phải verify bằng một bundle hợp lệ.

**Làm sao biết cert server hết hạn hay thiếu CA?**
Chạy `openssl s_client ... | openssl x509 -noout -dates -issuer`. Nếu `-dates` đã qua → hết hạn. Nếu issuer là CA lạ/không có trong bundle → thiếu CA.

**`certifi` và CA của hệ điều khác nhau chỗ nào?**
`certifi` là bundle đóng gói trong package Python, cập nhật theo pip. CA hệ điều hành do OS quản lý. `requests` mặc định ưu tiên `certifi`, nên upgrade OS không tự động cập nhật bundle mà `requests` dùng.

---

Nếu edge case của bạn vẫn kẹt sau khi thử hết các fix trên, bạn có thể dùng agent debug tự động để nó soi cert, interpreter và luồng proxy từng bước — [dùng agent tại đây](https://example.com/agent-debug).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
