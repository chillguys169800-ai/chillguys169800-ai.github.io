# Error: ModuleNotFoundError: No module named 'xyz' in Python

## TL;DR — fix nhanh

```bash
python -m pip install xyz
python -c "import sys; print(sys.executable)"
python -c "import xyz; print(xyz.__file__)"
```

Ba dòng trên giải quyết đa số trường hợp: cài bằng đúng interpreter đang chạy, xác nhận interpreter đó là interpreter bạn nghĩ, rồi xác nhận file module được nạp từ đâu. Nếu dòng cuối vẫn in ra đường dẫn khác với dự kiến, đọc tiếp phần nguyên nhân.

## Nguyên nhân thật: import không phải là "tìm file", mà là "hỏi interpreter đang chạy"

Khi Python gặp `import xyz`, nó không tìm kiếm toàn bộ ổ cứng. Trình import đi theo một chuỗi finder trong `sys.meta_path`, hỏi lần lượt: module built-in có không, module frozen có không, rồi có file/thư mục nào khớp trong từng entry của `sys.path` không. `ModuleNotFoundError` được raise khi **không có spec nào cho tên top-level `xyz`** trong suốt chuỗi đó. Nó là subclass của `ImportError`, nhưng khác biệt quan trọng: `ImportError` thường nghĩa là module đã tìm thấy nhưng bị lỗi bên trong (thuộc tính/submodule không tồn tại, circular import), còn `ModuleNotFoundError` nghĩa là **chưa tìm thấy module để mà import**.

Điểm hay bị hiểu nhầm nằm ở chỗ `sys.path` được xây dựng *cho interpreter cụ thể đang chạy*: nó chứa thư mục script, `PYTHONPATH`, stdlib, và `site-packages` của chính interpreter đó. `pip` cũng là một module của một interpreter: lệnh `pip install xyz` không cài "cho máy", nó cài cho `sys.prefix` của Python chạy lệnh đó. Vì vậy "máy tôi đã cài rồi mà vẫn báo lỗi" gần như luôn là hai interpreter khác nhau, chứ không phải Python lỗi.

Một tầng nữa: tên cài trên PyPI không đồng nhất với tên import. `xyz` trong thông báo lỗi có thể là placeholder trong tài liệu, nhưng nguyên tắc vẫn áp dụng cho tên thật — `beautifulsoup4` cho ra `bs4`, `Pillow` cho ra `PIL`, `PyYAML` cho ra `yaml`. Cài đúng package mà import sai tên vẫn ra đúng error này.

## 6 nguyên nhân phổ biến, xếp theo xác suất gặp

1. **pip và python không cùng một interpreter** — phổ biến nhất, đặc biệt khi máy có `python`, `python3`, conda, venv lồng nhau.
2. **Nhầm tên package với tên import** — cài `Pillow` rồi `import Pillow`, cài `python-dateutil` rồi `import python-dateutil`.
3. **Module local nằm ngoài `sys.path`** — chạy `python path/to/script.py` từ thư mục khác, project dùng layout `src/` mà chưa cài editable.
4. **Jupyter kernel hoặc IDE dùng interpreter khác terminal** — terminal cài xong nhưng kernel notebook vẫn trỏ sang base env.
5. **Dependency thiếu trong manifest** — chạy được ở local nhờ cài tay, nhưng CI/production chỉ đọc `requirements.txt` hoặc `pyproject.toml`.
6. **Editable install / namespace package / phân biệt hoa thường** — thư mục đổi tên, `.egg-info` cũ, thư mục rỗng cùng tên che namespace, hoặc `Xyz` trên macOS nhưng `xyz` trên Linux.

## Fix từng nguyên nhân

### 1.pip và python không cùng interpreter

```bash
which -a python python3 pip pip3
python -c "import sys; print(sys.executable); print(*sys.path, sep='\n')"
python -m pip --version
python -m pip install xyz
python -c "import xyz; print(xyz.__file__)"
```

Quy tắc cốt lõi: **luôn dùng `python -m pip`**, không gọi `pip` trực tiếp. Nếu chưa có venv, tạo lại từ đầu cho sạch:

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows PowerShell: .venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install xyz
python -c "import sys, xyz; print(sys.executable); print(xyz.__file__)"
```

### 2.Tên cài khác tên import

| Lệnh cài | Câu import đúng |
| --- | --- |
| `beautifulsoup4` | `import bs4` |
| `Pillow` | `import PIL` |
| `PyYAML` | `import yaml` |
| `python-dateutil` | `import dateutil` |
| `scikit-learn` | `import sklearn` |
| `opencv-python` | `import cv2` |

```bash
python -m pip show beautifulsoup4
python - <<'PY'
import importlib.util
for name in ("bs4", "PIL", "yaml", "dateutil", "sklearn", "cv2"):
    print(name, "->", bool(importlib.util.find_spec(name)))
PY
```

`find_spec()` trả về `None` chính là lỗi này dưới dạng kiểm tra được. Với `xyz` là nội bộ, thay tên thật vào và kiểm tra phần `Name:` trong `pip show`.

### 3.Module local không nằm trên `sys.path`

```bash
find . -type d -name xyz -o -type f -name 'xyz.py'
export PYTHONPATH="$(pwd)/src:${PYTHONPATH}"
python -m mypackage.main
python -c "import sys; print(sys.path[0])"
```

Chạy `python -m mypackage.main` sẽ đưa thư mục hiện tại vào `sys.path[0]`; chạy `python src/mypackage/main.py` thì `sys.path[0]` là thư mục chứa file script, không phải gốc repo — đó là lý do import tầng trên cùng hỏng. Cách bền hơn là khai báo project trong `pyproject.toml` rồi cài editable (xem mục 5). Trên Windows, thay `export` bằng `$env:PYTHONPATH = "$PWD\src;$env:PYTHONPATH"`.

### 4.Kernel Jupyter / IDE dùng interpreter khác

```bash
python -m pip install ipykernel
python -m ipykernel install --user --name project-xyz --display-name "Python (project-xyz)"
```

Trong notebook, chọn kernel `Python (project-xyz)` từ menu, rồi verify:

```python
import sys
print(sys.executable)
print(sys.path)
```

```python
%pip install xyz
```

Magic `%pip` cài vào đúng interpreter của kernel hiện tại, khác với `!pip install` vẫn đi theo `PATH` của shell. Trong VS Code/PyCharm, đối chiếu đường dẫn interpreter hiển thị ở thanh trạng thái với kết quả `python -c "import sys; print(sys.executable)"` trong terminal.

### 5.Thiếu dependency trong manifest

```toml
# pyproject.toml
[project]
dependencies = ["xyz>=1,<2"]
```

```txt
# requirements.txt
xyz>=1,<2
```

```bash
python -m pip install -e ".[dev]"
python -m pip install -r requirements.txt
python -m pip check
```

`pip check` bắt phần dependency của dependency bị thiếu — trường hợp bạn cài `xyz` nhưng một subpackage của nó không được kéo theo.

### 6.Editable install, namespace package, case-sensitivity

```bash
python -m pip show -f xyz
python -c "import importlib.util; print(importlib.util.find_spec('xyz'))"
find . -type d \( -name '*.egg-info' -o -name 'xyz' \) -print
python -m pip install --force-reinstall --no-cache-dir -e .
```

Một thư mục tên `xyz` nhưng không có `__init__.py` có thể trở thành namespace package rỗng và làm lu mờ package thật cài trong site-packages. Tương tự, `.pth`/`.egg-info` còn sót sau khi đổi tên thư mục project sẽ trỏ vào đường dẫn cũ. Trên Linux, `import Xyz` không bao giờ khớp file `xyz.py`; trên macOS thì có thể khớp — nên lỗi chỉ xuất hiện khi lên CI.

## Edge cases cần soi sâu hơn

- **Proxy và biến môi trường của pip**: `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `PIP_INDEX_URL`, `PIP_TRUSTED_HOST`, `REQUESTS_CA_BUNDLE`. Cài qua mirror nội bộ có thể "thành công" nhưng trả về artifact cũ hoặc sai tên, rồi mới báo `ModuleNotFoundError` lúc runtime. Soi bằng `python -m pip config list -v` và `python -m pip install -vvv xyz`.
- **Nhiều venv xếp chồng**: conda base + venv + direnv + Poetry `virtualenvs.in-project` + uv có thể cùng active. Đối chiếu `echo $VIRTUAL_ENV`, `echo $CONDA_PREFIX`, `which python` và `python -c "import sys; print(sys.prefix, sys.base_prefix)"`; đừng tin vào dấu nhắc shell.
- **DNS propagation / private index vừa đổi domain**: sau khi đổi mirror nội bộ, resolver hoặc cache DNS còn trỏ bản cũ. Kiểm tra `getent hosts <index-host>`, thử `python -m pip install --no-cache-dir xyz`, rồi so sánh kết quả từ mạng khác.

Với ba tình huống này, AI agent có thể tự debug từng bước nếu cần: lần lượt chạy lệnh chẩn đoán, đọc output, so sánh `sys.executable` với `pip --version`, thu hẹp dần từ môi trường xuống từng dòng cấu hình thay vì đoán.

## FAQ — schema JSON-LD

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "ModuleNotFoundError khác ImportError thế nào?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "ModuleNotFoundError nghĩa là trình import không tìm thấy spec cho tên module đó trong sys.meta_path và sys.path. ImportError thường nghĩa là module đã tìm thấy nhưng nội dung bên trong lỗi, ví dụ attribute không tồn tại hoặc import vòng."
      }
    },
    {
      "@type": "Question",
      "name": "Tại sao đã pip install xyz mà vẫn báo No module named 'xyz'?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Vì pip có thể thuộc interpreter khác với interpreter đang chạy script. Hãy dùng python -m pip install xyz rồi kiểm tra bằng python -c \"import sys; print(sys.executable)\" và python -c \"import xyz; print(xyz.__file__)\"."
      }
    },
    {
      "@type": "Question",
      "name": "Có nên thêm sys.path.insert vào code để sửa lỗi này không?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Chỉ nên dùng như biện pháp tạm thời. Cách bền là cài project ở chế độ editable với python -m pip install -e . hoặc khai báo dependency trong pyproject.toml, để môi trường nào cũng dựng lại được cùng một sys.path."
      }
    }
  ]
}
```

Nếu edge case của bạn vẫn kẹt sau khi đã đối chiếu interpreter, kernel và proxy, có thể để agent debug tự động từng bước thay vì lần mò thủ công: [Claude Code](https://claude.com/claude-code).

---

*Tự debug với AI agent: [Python Fix Guides — agent giải lỗi kèm checklist tự kiểm](https://chillguys169800-ai.github.io/)*
