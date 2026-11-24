# Hướng Dẫn Cài Đặt & Cấu Hình Thư Viện OCR / Đọc Mã QR (Mã Nguồn Mở) & Môi Trường QA/Staging Chính Chủ

> **Phạm vi tài liệu:** Hướng dẫn đầy đủ từng bước trên **Windows 11 (CPU 4 nhân / 8 luồng, Python 3.11)** cho hai mảng kỹ thuật hợp lệ:
> 1. Cài đặt và cấu hình các thư viện **OCR / đọc mã QR mã nguồn mở** chạy local (tài liệu, hóa đơn, ảnh chụp, quét mã QR nội bộ).
> 2. Cấu hình **khóa kiểm thử chính thức (QA/Staging Test Keys)** của **Cloudflare Turnstile** và **Google reCAPTCHA v2** trên website do chính Sếp quản lý để chạy bộ test tự động (Playwright / Selenium / Cypress) mà không cần vượt rào.

---

## Phần 1 — Khởi Tạo Môi Trường Ảo Python `.venv` Chuẩn Sạch

Mở **PowerShell** tại thư mục dự án (tránh cài thẳng vào Python toàn hệ thống để không xung đột phiên bản thư viện):

```powershell
# 1. Tạo thư mục làm việc và thư mục lưu model cục bộ
mkdir ocr_qr_workspace\models\wechat_qrcode -Force
cd ocr_qr_workspace

# 2. Tạo môi trường ảo Python 3.11 và kích hoạt
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 3. Cập nhật pip lên bản mới nhất
python -m pip install --upgrade pip setuptools wheel

# 4. Giới hạn số luồng CPU phù hợp máy 4 nhân / 8 luồng (tránh nghẽn luồng khi xử lý song song)
$env:OMP_NUM_THREADS = "4"
$env:MKL_NUM_THREADS = "4"
```

---

## Phần 2 — Cài Đặt & Cấu Hình Từng Thư Viện OCR / Đọc Mã QR

### 2.1. OpenCV `WeChatQRCode` + `pyzbar` (Đọc mã QR chính xác cao từ ảnh/tài liệu)

**Bước 1: Cài đặt gói Python & thư viện hệ thống Windows**
```powershell
pip install opencv-contrib-python pyzbar pillow
```
- **Lưu ý Windows:** Nếu khi `import pyzbar` gặp lỗi thiếu DLL (`libiconv.dll` hoặc `libzbar-64.dll`), tải và cài đặt **Microsoft Visual C++ Redistributable (x64, bản 2015–2022)** từ trang chính thức của Microsoft.

**Bước 2: Tải 4 file model AI chính thức cho `WeChatQRCode`**
Tải trực tiếp 4 tệp từ nhánh `wechat_qrcode` của kho chính thức `https://github.com/WeChatCV/opencv_3rdparty` vào thư mục `models\wechat_qrcode\`:

```powershell
$base = "https://raw.githubusercontent.com/WeChatCV/opencv_3rdparty/wechat_qrcode"
$files = @("detect.prototxt", "detect.caffemodel", "sr.prototxt", "sr.caffemodel")

foreach ($f in $files) {
    Invoke-WebRequest -Uri "$base/$f" -OutFile "models\wechat_qrcode\$f" -UseBasicParsing
}
```

**Bước 3: Cấu hình khởi tạo tối ưu cho CPU 4 nhân**
```python
import cv2
from pyzbar.pyzbar import decode as zbar_decode
from PIL import Image

# Giới hạn 4 luồng CPU để chạy mượt trên máy 4C/8T
cv2.setNumThreads(4)

detector = cv2.wechat_qrcode_WeChatQRCode(
    "models/wechat_qrcode/detect.prototxt",
    "models/wechat_qrcode/detect.caffemodel",
    "models/wechat_qrcode/sr.prototxt",
    "models/wechat_qrcode/sr.caffemodel",
)

def read_qr_local(image_path: str) -> list[str]:
    img = cv2.imread(image_path)
    if img is None:
        return []
    results, _ = detector.detectAndDecode(img)
    if results:
        return list(results)
    # Dự phòng bằng pyzbar nếu ảnh có kích thước lớn, sắc nét
    return [obj.data.decode("utf-8", errors="replace") for obj in zbar_decode(Image.open(image_path))]
```

---

### 2.2. PaddleOCR (`PP-OCR Mobile` — Đọc chữ tiếng Việt, hóa đơn, chứng từ trên CPU)

**Bước 1: Cài đặt PaddlePaddle bản CPU và PaddleOCR**
```powershell
pip install paddlepaddle paddleocr
```

**Bước 2: Cấu hình tối ưu CPU 4 nhân (bật `MKLDNN` tăng tốc suy luận CPU)**
```python
from paddleocr import PaddleOCR

ocr = PaddleOCR(
    use_angle_cls=True,     # Tự xoay nếu ảnh tài liệu bị nghiêng/ngược
    lang="vi",              # Hỗ trợ tiếng Việt có dấu
    use_gpu=False,          # Chạy thuần CPU
    enable_mkldnn=True,     # Tăng tốc Intel/AMD x86 trên Windows CPU
    cpu_threads=4,          # Khớp 4 nhân vật lý
)
```

**Bước 3: Lưu cấu hình model cục bộ để chạy offline hoàn toàn**
Sau lần chạy đầu tiên, PaddleOCR tự tải model về thư mục `%USERPROFILE%\.paddleocr\whl\`. Để cố định đường dẫn vào thư mục dự án (không phụ thuộc mạng về sau), copy thư mục model vào `models\paddleocr\` và truyền rõ tham số:
```python
ocr = PaddleOCR(
    use_angle_cls=True,
    lang="vi",
    use_gpu=False,
    enable_mkldnn=True,
    cpu_threads=4,
    det_model_dir="./models/paddleocr/det",
    rec_model_dir="./models/paddleocr/rec",
    cls_model_dir="./models/paddleocr/cls",
)
```

---

### 2.3. Tesseract OCR (`pytesseract` — OCR siêu nhẹ cho văn bản in nền sạch)

**Bước 1: Cài bộ cài nhị phân Windows (UB-Mannheim)**
1. Tải bộ cài Windows 64-bit tại: `https://github.com/UB-Mannheim/tesseract/wiki`
2. Trong trình cài đặt, mở mục **Additional language data (download)** $\rightarrow$ tick chọn **`Vietnamese (vie)`**.
3. Giữ nguyên đường dẫn mặc định `C:\Program Files\Tesseract-OCR\`.

**Bước 2: Cài thư viện kết nối trong `.venv`**
```powershell
pip install pytesseract pillow opencv-python
```

**Bước 3: Cấu hình đường dẫn file thực thi & tham số đọc chuẩn**
```python
import pytesseract
from PIL import Image

# Trỏ thẳng vào file cài đặt (không phụ thuộc biến PATH của Windows)
pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"

# --oem 3: LSTM engine mới nhất | --psm 6: Khối văn bản đồng nhất | -l vie+eng: Song ngữ Việt + Anh
text = pytesseract.image_to_string(
    Image.open("tai_lieu.png"),
    lang="vie+eng",
    config="--oem 3 --psm 6",
)
```

---

### 2.4. EasyOCR (Đa ngôn ngữ trên nền PyTorch)

**Bước 1: Cài PyTorch bản thuần CPU trước** *(giúp nhẹ hơn ~2 GB so với bản mặc định kèm CUDA)*:
```powershell
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install easyocr
```

**Bước 2: Cấu hình thư mục lưu model cục bộ & tắt GPU**
```python
import easyocr

reader = easyocr.Reader(
    ["vi", "en"],
    gpu=False,
    model_storage_directory="./models/easyocr",
    download_enabled=True,  # Sau lần tải đầu có thể đổi thành False để chạy offline
)
```

---

## Phần 3 — Cấu Hình Khóa Kiểm Thử Chính Chủ (QA / Staging) Cho Website Của Sếp

Khi chạy kiểm thử tự động (Playwright, Cypress, Selenium) trên website **do chính Sếp vận hành** (môi trường `localhost`, staging hoặc CI/CD), **KHÔNG** dùng công cụ vượt rào mà cấu hình **bộ khóa test chính thức** do nhà cung cấp cấp sẵn.

### 3.1. Cloudflare Turnstile — Bộ Khóa Kiểm Thử Chính Thức (Official Dummy Keys)

Các khóa test này hoạt động trên **mọi tên miền**, bao gồm `localhost`, `127.0.0.1` và pipeline CI/CD mà không cần khai báo hostname trên Cloudflare Dashboard.

**A. Test Sitekeys (Cấu hình ở Frontend HTML / JS)**

| Sitekey | Hành vi mô phỏng | Kiểu Widget | Mục đích kiểm thử |
|---|---|---|---|
| `1x00000000000000000000AA` | Luôn vượt qua (**Always passes**) | Hiển thị (Visible) | Test luồng submit form thành công |
| `2x00000000000000000000AB` | Luôn thất bại (**Always fails**) | Hiển thị (Visible) | Test giao diện báo lỗi và nút thử lại |
| `1x00000000000000000000BB` | Luôn vượt qua (**Always passes**) | Ẩn (Invisible) | Test widget ẩn chạy ngầm thành công |
| `2x00000000000000000000BB` | Luôn thất bại (**Always fails**) | Ẩn (Invisible) | Test xử lý lỗi với widget ẩn |
| `3x00000000000000000000FF` | Bắt buộc hiện thử thách tương tác | Hiển thị (Visible) | Test luồng người dùng tương tác thủ công |

**B. Test Secret Keys (Cấu hình ở Backend gọi API `siteverify`)**

| Secret Key | Phản hồi từ `siteverify` | Mục đích kiểm thử |
|---|---|---|
| `1x0000000000000000000000000000000AA` | Luôn xác thực thành công (`success: true`) | Test backend chấp nhận token hợp lệ |
| `2x0000000000000000000000000000000AA` | Luôn từ chối (`success: false`) | Test backend chặn token không hợp lệ |
| `3x0000000000000000000000000000000AA` | Trả về lỗi `"token already spent"` | Test cơ chế chống dùng lại token cũ (replay protection) |

**Lưu ý kỹ thuật quan trọng:**
- **Không trộn khóa:** Secret Key thật (Production) sẽ từ chối token sinh từ Dummy Sitekey và ngược lại.
- **Thời hạn token:** Token chỉ dùng được **1 lần** và hết hạn sau **300 giây (5 phút)**. Nếu form submit lỗi, cần gọi `turnstile.reset()` ở frontend để lấy token mới.

---

### 3.2. Google reCAPTCHA v2 — Bộ Khóa Kiểm Thử Chính Thức

- **Site key (Frontend):** `6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI`
- **Secret key (Backend `siteverify`):** `6LeIxAcTAAAAAGG-vFI1TnRWxMZNFuojJ4WifJWe`

**Lưu ý vận hành:**
- Toàn bộ request xác thực bằng cặp khóa này luôn trả về thành công trên môi trường dev/staging để không chặn test tự động.
- Widget sẽ hiển thị dòng chữ cảnh báo màu đỏ *"This reCAPTCHA is for testing purposes only"* để tránh đẩy nhầm lên môi trường thật (Production).

---

### 3.3. Mẫu Tách Cấu Hình `.env.staging` vs `.env.production` (Chống Đẩy Nhầm Lên Production)

Tách biến môi trường rõ ràng giữa Staging và Production:

**Tệp `.env.staging` (dùng khi chạy `localhost` / CI test):**
```ini
APP_ENV=staging
TURNSTILE_SITE_KEY=1x00000000000000000000AA
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA
RECAPTCHA_SITE_KEY=6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI
RECAPTCHA_SECRET_KEY=6LeIxAcTAAAAAGG-vFI1TnRWxMZNFuojJ4WifJWe
```

**Tệp `.env.production` (chỉ nạp trên máy chủ thật):**
```ini
APP_ENV=production
TURNSTILE_SITE_KEY=<REAL_SITE_KEY_TỪ_CLOUDFLARE_DASHBOARD>
TURNSTILE_SECRET_KEY=<REAL_SECRET_KEY_TỪ_CLOUDFLARE_DASHBOARD>
```

**Kiểm tra an toàn khi khởi động server (Safety Guard):**
```python
import os

APP_ENV = os.getenv("APP_ENV", "development")
TURNSTILE_SECRET = os.getenv("TURNSTILE_SECRET_KEY", "")
RECAPTCHA_SECRET = os.getenv("RECAPTCHA_SECRET_KEY", "")

DUMMY_SECRETS = {
    "1x0000000000000000000000000000000AA",
    "2x0000000000000000000000000000000AA",
    "3x0000000000000000000000000000000AA",
    "6LeIxAcTAAAAAGG-vFI1TnRWxMZNFuojJ4WifJWe",
}

if APP_ENV == "production" and (TURNSTILE_SECRET in DUMMY_SECRETS or RECAPTCHA_SECRET in DUMMY_SECRETS):
    raise RuntimeError("CẤM: Đang nạp nhầm Testing Secret Key vào môi trường Production!")
```

---

## Phần 4 — Lệnh Tự Kiểm Tra Nhanh (`Self-Test`) Trong PowerShell

Sau khi cài xong các gói cần dùng vào `.venv`, chạy 1 lệnh sau trong PowerShell để xác nhận môi trường nhận đủ thư viện và model:

```powershell
python -c "
import importlib
for pkg in ['cv2', 'pyzbar', 'PIL', 'paddleocr', 'pytesseract', 'easyocr']:
    try:
        m = importlib.import_module(pkg)
        print(f'[OK] {pkg}:', getattr(m, '__version__', 'installed'))
    except Exception as e:
        print(f'[MISS/ERR] {pkg}:', e)
"
```

Nếu dùng thêm Tesseract trên Windows, kiểm tra nhanh binary:
```powershell
& "C:\Program Files\Tesseract-OCR\tesseract.exe" --version
& "C:\Program Files\Tesseract-OCR\tesseract.exe" --list-langs
```
*(Kết quả `--list-langs` cần hiện cả `eng` và `vie`).*
