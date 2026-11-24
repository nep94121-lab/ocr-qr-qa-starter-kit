# Cấu Hình Dự Án: OCR / QR Code Starter Kit & QA Testing

## 1. Quy Tắc Giao Tiếp & Ngôn Ngữ
- Luôn giao tiếp bằng **tiếng Việt**, ngắn gọn, rõ ràng và đi thẳng vào giải pháp kỹ thuật.
- Mọi tệp mã nguồn, cấu hình và tài liệu phát sinh lưu trực tiếp trong thư mục dự án hiện tại.

## 2. Môi Trường & Phần Cứng Chuẩn
- **Hệ điều hành:** Windows 11.
- **CPU:** 4 nhân vật lý / 8 luồng logic.
- **Python:** 3.11 chạy trong môi trường ảo cục bộ `.venv` (không cài thư viện tràn lan vào Python hệ thống).
- **Giới hạn luồng CPU:** Luôn thiết lập biến môi trường `OMP_NUM_THREADS=4`, `MKL_NUM_THREADS=4` và `cv2.setNumThreads(4)` (hoặc `cpu_threads=4` trong PaddleOCR) để tránh nghẽn CPU.

## 3. Tài Nguyên Có Sẵn Trong Dự Án
- Bộ 4 tệp model OpenCV `WeChatQRCode` nằm sẵn tại `./models/wechat_qrcode/`:
  - `detect.prototxt`
  - `detect.caffemodel`
  - `sr.prototxt`
  - `sr.caffemodel`
- Tài liệu hướng dẫn cài đặt chi tiết: `./huong_dan_cai_dat_ocr_qr_va_qa_testing.md` và `./README.md`.

## 4. Phạm Vi Sử Dụng Hợp Lệ & Bảo Mật
- Dự án phục vụ đọc tài liệu, hóa đơn, ảnh chụp, giải mã QR nội bộ và chạy kiểm thử tự động QA/Staging cho website tự quản lý bằng khóa test chính thức (Cloudflare Turnstile dummy keys, Google reCAPTCHA v2 test keys).
- Không triển khai chức năng vượt qua (bypass) CAPTCHA, chống bot hoặc ghi cứng bí mật thật (`REAL_SECRET_KEY`) vào mã nguồn / Git.
- Tách biệt `.env.staging` và `.env.production`; luôn bật kiểm tra chặn nạp nhầm khóa test trên môi trường production.
