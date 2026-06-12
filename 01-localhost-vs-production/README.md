# Section 1 — Từ Localhost Đến Production

## Mục tiêu học
- Hiểu tại sao "it works on my machine" là vấn đề
- Nhận ra sự khác biệt giữa dev và production environment
- Áp dụng 4 nguyên tắc 12-factor cơ bản

---

## Ví dụ Basic — Agent "Kiểu Localhost"

```
develop/
├── app.py          # ❌ Anti-patterns: hardcode secrets, no config, no health check
└── requirements.txt
```

### Chạy thử
```bash
cd develop
pip install -r requirements.txt
python app.py
# Truy cập: http://localhost:8000
```

### Những vấn đề trong code này:
1. API key hardcode trong code
2. Không có health check endpoint
3. Debug mode bật cứng
4. Không xử lý SIGTERM gracefully
5. Config không đến từ environment

---

## Ví dụ Advanced — 12-Factor Compliant Agent

```
production/
├── app.py          # ✅ Clean: config from env, health check, graceful shutdown
├── config.py       # ✅ Centralized config management
├── .env.example    # ✅ Template — không commit .env thật
└── requirements.txt
```

### Chạy thử
```bash
cd production
pip install -r requirements.txt
cp .env.example .env
# Sửa .env nếu cần
python app.py
```

### So sánh với Basic:

| | Basic (❌) | Advanced (✅) |
|--|-----------|--------------|
| Config | Hardcode trong code | Đọc từ env vars |
| Secrets | `api_key = "sk-abc123"` | `os.getenv("OPENAI_API_KEY")` |
| Port | Cố định `8000` | Từ `PORT` env var |
| Health check | Không có | `GET /health` |
| Shutdown | Tắt đột ngột | Graceful — hoàn thành request hiện tại |
| Logging | `print()` | Structured JSON logging |

---

## Câu hỏi thảo luận

1. Điều gì xảy ra nếu bạn push code với API key hardcode lên GitHub public?
- **Bị quét và mất Key ngay lập tức**: Các tin tặc sử dụng bot quét tự động (automated key scrapers) liên tục rà soát GitHub 24/7. Nếu phát hiện key (như `sk-...` của OpenAI hay AWS credentials), bot sẽ chiếm quyền sử dụng trong vòng vài giây.
- **Hậu quả tài chính**: Kẻ xấu dùng key để spam requests, chạy các mô hình lớn hoặc đào tiền ảo, khiến bạn phải chịu hóa đơn khổng lồ từ nhà cung cấp dịch vụ.
- **Bị khóa tài khoản**: Các nhà cung cấp dịch vụ lớn (như OpenAI, GitHub) có cơ chế tự động phát hiện rò rỉ và sẽ tự động vô hiệu hóa (revoke) key đó để bảo vệ bạn, nhưng tài khoản của bạn vẫn có nguy cơ bị khóa tạm thời.
- **Lưu vết vĩnh viễn trong Git History**: Cho dù bạn sửa code và push commit mới lên, key vẫn tồn tại trong lịch sử commit cũ. Bạn phải dùng các công cụ đặc biệt như `git-filter-repo` hoặc BFG Repo-Cleaner để dọn dẹp lịch sử Git triệt để.

2. Tại sao stateless quan trọng khi scale?
- **Khả năng cân bằng tải (Load Balancing)**: Khi ứng dụng scale horizontal (tăng số lượng instances chạy song song), bộ cân bằng tải (Load Balancer) sẽ phân phối requests của người dùng tới bất kỳ instance ngẫu nhiên nào. Nếu ứng dụng là stateless (không lưu trạng thái phiên làm việc trong RAM/ổ đĩa cục bộ), bất kỳ instance nào cũng có thể xử lý request đó mà không bị mất lịch sử hay lỗi phiên.
- **Auto-scaling dễ dàng**: Bạn có thể thêm hoặc bớt instances bất kỳ lúc nào dựa theo lưu lượng tải thực tế mà không cần lo lắng về việc chuyển dời hoặc đồng bộ hóa bộ nhớ giữa các máy chủ.
- **Khả năng chống chịu lỗi (Fault Tolerance)**: Nếu một instance bị crash đột ngột, instance khác sẽ ngay lập tức thay thế và tiếp tục xử lý công việc từ database trung tâm (như Redis) mà không làm gián đoạn trải nghiệm của người dùng.

3. 12-factor nói "dev/prod parity" — nghĩa là gì trong thực tế?
- **Khái niệm**: Giữ cho môi trường phát triển (Development) và môi trường vận hành (Production) giống nhau tối đa có thể về: công cụ, mã nguồn cấu hình, thư viện và quy trình vận hành.
- **Thực tế**:
  - **Đồng bộ hóa Backing Services**: Không dùng SQLite ở local nhưng deploy PostgreSQL ở production. Thay vào đó, phải dùng PostgreSQL ở cả hai nơi (chạy local qua Docker).
  - **Sử dụng Docker**: Đóng gói ứng dụng vào Docker container để chạy ở local hay cloud đều sử dụng chung một phiên bản Python, hệ điều hành nền và cấu hình thư viện y hệt nhau.
  - **Rút ngắn khoảng cách triển khai (Continuous Deployment)**: Code viết xong ở local được tự động kiểm thử và deploy lên production càng sớm càng tốt để phát hiện lỗi sớm thay vì tích tụ hàng tháng mới deploy.