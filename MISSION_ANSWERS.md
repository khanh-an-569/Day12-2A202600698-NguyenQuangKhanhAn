# Day 12 Lab - Mission Answers

> **Student Name:** Nguyễn Quang Khánh An  
> **Student ID:** 2A202600698  
> **Date:** 12/06/2026

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in basic app.py
1. **API key hardcoded trong code (`OPENAI_API_KEY = "sk-..."`)**: Dễ bị rò rỉ thông tin nhạy cảm khi đẩy mã nguồn lên GitHub.
2. **Database URL hardcoded (`DATABASE_URL = "postgresql://..."`)**: Thông tin đăng nhập database bị lộ và không linh hoạt giữa các môi trường.
3. **Cấu hình debug bật cứng (`DEBUG = True`)**: Làm lộ chi tiết lỗi hệ thống cho client và giảm hiệu năng trong production.
4. **Sử dụng lệnh `print()` để log**: Không có cấu trúc, khó phân tích trên log aggregators, và log cả thông tin nhạy cảm (API key).
5. **Không có Health Check endpoint**: Nền tảng cloud không thể tự phát hiện nếu app bị crash hoặc treo để tự động khởi động lại.
6. **Cố định host là `localhost` và port `8000`**: Không thể nhận kết nối bên ngoài khi chạy trong Docker container (`localhost` chỉ nghe nội bộ container), và bị crash trên cloud do các nền tảng tự động gán cổng động qua biến môi trường `PORT`.
7. **Không có Graceful Shutdown**: Khi có tín hiệu tắt, app dừng đột ngột làm đứt gãy các request đang xử lý dở dang của người dùng.

---

### Exercise 1.3: Comparison table

| Feature | Develop (Basic) | Production (Advanced) | Tại sao quan trọng? |
| :--- | :--- | :--- | :--- |
| **Config** | Hardcode trực tiếp trong code | Load động từ environment variables (`.env` thông qua Pydantic Settings) | Tuân thủ 12-Factor App. Bảo mật secrets tuyệt đối và dễ dàng chuyển đổi cấu hình giữa Dev/Staging/Prod mà không cần sửa code. |
| **Health check** | Không có | Cung cấp endpoints `/health` (Liveness) và `/ready` (Readiness) | Giúp Cloud Platform tự động giám sát. Nếu app lỗi/treo, platform sẽ tự restart (Liveness). Nếu app chưa tải xong DB/Model, platform sẽ không điều hướng traffic vào (Readiness). |
| **Logging** | Dùng `print()`, log cả API key | Định dạng JSON có cấu trúc (`Structured JSON logging`), che giấu secrets | Giúp các công cụ thu thập log tập trung (Datadog, Loki, ELK) dễ dàng parse, tìm kiếm và cảnh báo lỗi tự động. Tránh lộ thông tin nhạy cảm. |
| **Shutdown** | Đột ngột (abrupt) | Graceful shutdown (bắt tín hiệu `SIGTERM`, xử lý nốt request hiện tại) | Đảm bảo trải nghiệm người dùng không bị lỗi ngắt quãng khi deploy phiên bản mới. Cho phép đóng các kết nối (DB, Redis) một cách an toàn. |
| **Binding Host/Port**| `localhost` / `8000` | `0.0.0.0` / đọc cổng từ biến môi trường `PORT` | Đảm bảo app có thể chạy được bên trong Docker (0.0.0.0 lắng nghe mọi card mạng) và tương thích với cơ chế gán cổng động của các PaaS (Railway, Render). |

---

### Discussion Questions (Part 1)

#### 1. Điều gì xảy ra nếu bạn push code với API key hardcode lên GitHub public?
- **Bị quét và mất Key ngay lập tức**: Các tin tặc sử dụng bot quét tự động (automated key scrapers) liên tục rà soát GitHub 24/7. Nếu phát hiện key (như `sk-...` của OpenAI hay AWS credentials), bot sẽ chiếm quyền sử dụng trong vòng vài giây.
- **Hậu quả tài chính**: Kẻ xấu dùng key để spam requests, chạy các mô hình lớn hoặc đào tiền ảo, khiến bạn phải chịu hóa đơn khổng lồ từ nhà cung cấp dịch vụ.
- **Bị khóa tài khoản**: Các nhà cung cấp dịch vụ lớn (như OpenAI, GitHub) có cơ chế tự động phát hiện rò rỉ và sẽ tự động vô hiệu hóa (revoke) key đó để bảo vệ bạn, nhưng tài khoản của bạn vẫn có nguy cơ bị khóa tạm thời.
- **Lưu vết vĩnh viễn trong Git History**: Cho dù bạn sửa code và push commit mới lên, key vẫn tồn tại trong lịch sử commit cũ. Bạn phải dùng các công cụ đặc biệt như `git-filter-repo` hoặc BFG Repo-Cleaner để dọn dẹp lịch sử Git triệt để.

#### 2. Tại sao stateless quan trọng khi scale?
- **Khả năng cân bằng tải (Load Balancing)**: Khi ứng dụng scale horizontal (tăng số lượng instances chạy song song), bộ cân bằng tải (Load Balancer) sẽ phân phối requests của người dùng tới bất kỳ instance ngẫu nhiên nào. Nếu ứng dụng là stateless (không lưu trạng thái phiên làm việc trong RAM/ổ đĩa cục bộ), bất kỳ instance nào cũng có thể xử lý request đó mà không bị mất lịch sử hay lỗi phiên.
- **Auto-scaling dễ dàng**: Bạn có thể thêm hoặc bớt instances bất kỳ lúc nào dựa theo lưu lượng tải thực tế mà không cần lo lắng về việc chuyển dời hoặc đồng bộ hóa bộ nhớ giữa các máy chủ.
- **Khả năng chống chịu lỗi (Fault Tolerance)**: Nếu một instance bị crash đột ngột, instance khác sẽ ngay lập tức thay thế và tiếp tục xử lý công việc từ database trung tâm (như Redis) mà không làm gián đoạn trải nghiệm của người dùng.

#### 3. 12-factor nói "dev/prod parity" — nghĩa là gì trong thực tế?
- **Khái niệm**: Giữ cho môi trường phát triển (Development) và môi trường vận hành (Production) giống nhau tối đa có thể về: công cụ, mã nguồn cấu hình, thư viện và quy trình vận hành.
- **Ý nghĩa thực tế**:
  - **Đồng bộ hóa Backing Services**: Không dùng SQLite ở local nhưng deploy PostgreSQL ở production. Thay vào đó, phải dùng PostgreSQL ở cả hai nơi (chạy local qua Docker).
  - **Sử dụng Docker**: Đóng gói ứng dụng vào Docker container để chạy ở local hay cloud đều sử dụng chung một phiên bản Python, hệ điều hành nền và cấu hình thư viện y hệt nhau.
  - **Rút ngắn khoảng cách triển khai (Continuous Deployment)**: Code viết xong ở local được tự động kiểm thử và deploy lên production càng sớm càng tốt để phát hiện lỗi sớm thay vì tích tụ hàng tháng mới deploy.

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. **Base image là gì?**
   - Base image là `python:3.11` (Bản phân phối Python đầy đủ, dung lượng file nén lớn khoảng ~1 GB).
2. **Working directory là gì?**
   - Working directory trong container là `/app` (tất cả các lệnh chạy và copy sau đó sẽ mặc định thao tác tại đây).
3. **Tại sao COPY requirements.txt trước?**
   - Để tận dụng cơ chế lưu bộ nhớ đệm theo lớp của Docker (**Docker Layer Cache**). 
   - Vì danh sách các thư viện phụ thuộc (`requirements.txt`) rất ít khi thay đổi so với mã nguồn (`app.py`), nên việc copy và chạy `pip install` trước sẽ giúp Docker cache lại lớp này. Trong các lần build tiếp theo, Docker không cần cài đặt lại thư viện mà chỉ build từ lớp copy mã nguồn, giúp giảm thời gian build từ vài phút xuống còn vài giây.
4. **CMD vs ENTRYPOINT khác nhau thế nào?**
   - **ENTRYPOINT**: Định nghĩa lệnh cốt lõi không đổi khi container chạy (ví dụ: `python`). Nó rất khó bị ghi đè khi chạy container bằng lệnh `docker run` (phải dùng cờ `--entrypoint`). Mọi đối số bổ sung truyền vào từ command line sẽ được gắn tiếp vào sau lệnh này.
   - **CMD**: Định nghĩa lệnh hoặc tham số mặc định và có thể bị ghi đè hoàn toàn một cách dễ dàng khi người dùng truyền một lệnh khác ở cuối lệnh `docker run`. Khi dùng chung với `ENTRYPOINT`, `CMD` đóng vai trò là tham số mặc định truyền vào lệnh của `ENTRYPOINT`.

### Exercise 2.3: Image size comparison & Multi-stage build questions
- **Develop Image (Basic)**: ~1.66 GB
- **Production Image (Advanced)**: ~200-300 MB
- **Difference**: Giảm khoảng ~80% dung lượng.

#### Questions:
1. **Stage 1 (Builder) làm gì?**
   - Làm nhiệm vụ **chuẩn bị và cài đặt**: Tải base image `python:3.11-slim` và cài đặt các công cụ biên dịch (như `gcc`, `libpq-dev`) cần thiết để build và cài đặt các thư viện Python (dependencies) vào thư mục `/root/.local` bằng cờ `--user`.
2. **Stage 2 (Runtime) làm gì?**
   - Làm nhiệm vụ **chạy ứng dụng (production environment)**: Khởi tạo một môi trường Python slim sạch sẽ mới, tạo non-root user (`appuser` hoặc `agent`) để bảo mật, sau đó chỉ sao chép các file thư viện Python đã cài đặt thành công từ Stage 1 sang (`COPY --from=builder /root/.local ...`) và mã nguồn của app để chạy uvicorn.
3. **Tại sao image nhỏ hơn?**
   - Vì toàn bộ các công cụ biên dịch nặng nề (như `gcc`, `apt-get` packages) và cache của pip chỉ tồn tại ở Stage 1 và hoàn toàn bị loại bỏ, không copy sang Stage 2.
   - Hơn nữa, Stage 2 sử dụng base image `python:3.11-slim` (chỉ chứa các thành phần tối thiểu để chạy Python) thay vì image `python:3.11` đầy đủ.

### Exercise 2.4: Docker Compose stack
- **Các Services được chạy**: `nginx`, `agent`, `redis`, `qdrant`.
- **Cơ chế giao tiếp (Communication)**:
  - Tất cả các service này cùng kết nối chung vào một Docker bridge network biệt lập tên là `internal`.
  - **Nginx** là cổng duy nhất mở ra ngoài qua cổng `80` (HTTP) để nhận request từ Client, sau đó đóng vai trò làm Load Balancer điều hướng requests sang các instance `agent` ở cổng `8000`.
  - **Agent** nhận request, thực hiện kết nối tới `redis` để đọc/ghi session và lịch sử hội thoại, và kết nối tới `qdrant` để truy vấn dữ liệu vector tri thức hỗ trợ RAG.
  - Các container giao tiếp nội bộ thông qua tên service làm hostname (ví dụ: `redis:6379`, `qdrant:6333`).

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **Public URL**: https://day12-part6-production.up.railway.app
- **Screenshots**:
  - Deployment Dashboard: [dashboard.png](screenshots/dashboard.png)
  - Service Running: [running.png](screenshots/running.png)
  - Test Results: [test.png](screenshots/test.png)

---

## Part 4: API Security

### Exercise 4.1: API Key authentication
1. **API key được check ở đâu?**
   - API Key được check bởi hàm dependency **`verify_api_key`** (FastAPI Dependency Injection). Hàm này trích xuất API Key từ header `X-API-Key` của request và đối chiếu với biến môi trường `AGENT_API_KEY` của ứng dụng.
2. **Điều gì xảy ra nếu sai key?**
   - Nếu thiếu API Key (không truyền header `X-API-Key`): Trả về lỗi **`401 Unauthorized`** kèm thông báo `"Invalid or missing API key. Include header: X-API-Key: <key>"`.
   - Nếu truyền sai API Key: Trả về lỗi **`401 Unauthorized`** kèm thông báo `"Invalid or missing API key. Include header: X-API-Key: <key>"`.
3. **Làm sao rotate key?**
   - Bạn chỉ cần thay đổi giá trị của biến môi trường `AGENT_API_KEY` (trong file `.env` hoặc trên Cloud Dashboard như Railway/Render) và khởi động lại ứng dụng. Do ứng dụng đọc key động từ môi trường qua `os.getenv` thay vì viết cứng (hardcode) trong code, nên không cần sửa đổi hay build lại mã nguồn.

### Exercise 4.2: JWT authentication (Advanced)
1. **JWT Flow trong môi trường Production**:
   - **Bước 1 (Đăng nhập)**: Client gửi một request `POST /token` chứa thông tin đăng nhập (như `username` và `password`) dạng JSON tới máy chủ.
   - **Bước 2 (Xác thực & Tạo Token)**: Server kiểm tra thông tin đăng nhập từ cơ sở dữ liệu giả lập. Nếu chính xác, server tạo một mã Access Token JWT có chứa payload (thông tin user, vai trò `role: admin/user`, thời gian hết hạn `exp`, hạn mức chi tiêu `daily_limit`). Token này được ký số bằng thuật toán HMAC-SHA256 kết hợp khóa bí mật `JWT_SECRET` của server.
   - **Bước 3 (Gửi & Lưu trữ Token)**: Server trả về JWT token cho Client dưới dạng JSON. Client lưu trữ token này trong bộ nhớ local hoặc cookie để sử dụng sau.
   - **Bước 4 (Truy cập được bảo mật)**: Ở các request tiếp theo tới các endpoint cần bảo mật (như `/ask`), Client gửi kèm token này trong header `Authorization: Bearer <JWT_TOKEN>`.
   - **Bước 5 (Giải mã & Phê duyệt)**: FastAPI sử dụng dependency injection (`verify_token`) để chặn request, kiểm tra chữ ký và tính hợp lệ của token. Nếu token hợp lệ và chưa hết hạn, request được chuyển tiếp đến controller xử lý; nếu không hợp lệ, hệ thống trả về mã lỗi `401 Unauthorized`.

### Exercise 4.3: Rate limiting
1. **Thuật toán sử dụng**:
   - Trong ứng dụng ở Part 4 (Production), thuật toán **Sliding Window Counter** được sử dụng ở local (bộ nhớ in-memory) bằng cách duy trì một `deque` các mốc thời gian (timestamps) của các request của từng user. Các request có timestamp ngoài cửa sổ giám sát (60 giây trước) sẽ bị loại bỏ (`popleft()`).
   - Trong ứng dụng lab hoàn chỉnh (`06-lab-complete`), hệ thống sử dụng **Redis-based Fixed Window Counter** làm cơ chế chính. Mỗi phút, một key mới có dạng `rate_limit:{user_id}:{current_minute}` sẽ được tăng giá trị (`incr`) trong Redis và cấu hình thời gian sống `TTL = 60s`. Nếu Redis lỗi, hệ thống tự động fallback sang cơ chế sliding window in-memory bằng `deque`.
2. **Hạn mức (Limit)**:
   - Được cấu hình động thông qua biến môi trường `RATE_LIMIT_PER_MINUTE` (mặc định là **20 requests/minute**).
3. **Bypass limit cho admin**:
   - Hệ thống khởi tạo hai instance của RateLimiter: `rate_limiter_user` (10 req/min) và `rate_limiter_admin` (100 req/min). Khi giải mã token thành công và phát hiện payload chứa vai trò `"admin"`, hệ thống sẽ áp dụng bộ lọc `rate_limiter_admin` thay vì `rate_limiter_user` để nâng hạn mức cho tài khoản admin lên gấp 10 lần.

### Exercise 4.4: Cost guard implementation
1. **Giải pháp triển khai (Approach)**:
   - **Tính toán chi phí (Token Pricing)**: Mỗi request được ước tính chi phí trước khi gọi LLM: số lượng token input được ước lượng bằng cách đếm số từ của câu hỏi nhân với 2, nhân với đơn giá đầu vào của model GPT-4o-mini ($0.00015 / 1K tokens). Sau khi có câu trả lời, chi phí đầu ra tiếp tục được cộng thêm dựa trên số từ của câu trả lời ($0.0006 / 1K tokens).
   - **Lưu vết và cộng dồn bằng Redis**: Tổng chi phí tích lũy trong ngày của mỗi user được lưu trữ trong Redis bằng lệnh `incrbyfloat` với key dạng `cost:{user_id}:{today}` (TTL là 2 ngày để đảm bảo cache dọn dẹp tự động). Trước mỗi request, hệ thống gọi `check_budget` để kiểm tra chi phí tích lũy hiện tại có vượt quá giới hạn hàng ngày (`daily_budget_usd`, mặc định là $5.0) hay không. Nếu vượt quá, trả về mã lỗi `402 Payment Required` hoặc `503 Service Unavailable`.
   - **In-memory Fallback**: Trường hợp không có kết nối Redis, hệ thống lưu trữ tổng chi phí trong ngày vào biến RAM cục bộ (`_daily_cost` và `_cost_reset_day`), tự động reset khi chuyển sang ngày mới.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks implementation
- **GET /health (Liveness Probe)**: Trả về trạng thái hoạt động hiện tại của container (`"status": "ok"`), phiên bản ứng dụng, môi trường (development/production), thời gian hoạt động liên tục (uptime) và số lượng request đã xử lý. Nếu container bị treo hoặc crash, endpoint không phản hồi và platform (như Railway) sẽ tự động khởi động lại container mới.
- **GET /ready (Readiness Probe)**: Đảm bảo container đã sẵn sàng nhận traffic. Endpoint kiểm tra biến nội bộ `_is_ready` (được set thành `True` sau khi hoàn thành startup trong lifespan) và kiểm tra kết nối ping tới Redis. Nếu kết nối database/Redis lỗi, trả về mã HTTP `503 Service Unavailable` để load balancer tạm thời ngừng điều hướng request tới container này.

### Exercise 5.2: Graceful shutdown
- Ứng dụng đăng ký signal handler để bắt tín hiệu kết thúc chương trình (`SIGTERM`) từ container orchestrator khi thực hiện deploy bản mới hoặc scale-down.
- Khi nhận tín hiệu, server FastAPI ngừng nhận thêm các kết nối/request mới, nhưng Uvicorn được cấu hình `timeout_graceful_shutdown=30` sẽ đợi tối đa 30 giây để hoàn thành nốt các request hiện tại đang xử lý dở dang, sau đó đóng các kết nối tới Redis/database một cách an toàn trước khi thoát tiến trình, ngăn chặn việc mất mát dữ liệu hoặc làm lỗi request của người dùng.

### Exercise 5.3: Stateless design
- **Đặc điểm thiết kế**: Toàn bộ dữ liệu hội thoại (conversation history), bộ đếm rate limit và thông tin ngân sách cost guard được lưu trữ tập trung tại Redis thay vì ghi nhận vào RAM cục bộ của container.
- **Tại sao quan trọng**: Khi ứng dụng được scale horizontal (chạy nhiều bản sao song song đằng sau Load Balancer), client có thể được điều hướng tới bất kỳ container ngẫu nhiên nào. Nhờ thiết kế stateless, bất kỳ instance nào cũng có thể xử lý request vì chúng cùng đọc và ghi chung trạng thái lên Redis tập trung.

### Exercise 5.4: Load balancing
- Triển khai **Nginx** làm Load Balancer đặt phía trước cụm dịch vụ `agent`. Client chỉ giao tiếp qua cổng `80` của Nginx. Nginx sẽ phân tán các request đến các container `agent` ở cổng `8000` theo thuật toán quay vòng (Round Robin). Khi tăng số lượng instance (`--scale agent=3`), tải hệ thống được chia đều giúp giảm tải cho từng máy chủ riêng lẻ.

### Exercise 5.5: Test stateless design
- Thực hiện kịch bản kiểm thử:
  1. Gửi request đầu tiên đến endpoint `/ask` (thông qua load balancer Nginx) để lưu thông tin vào lịch sử trò chuyện.
  2. Dùng lệnh `docker compose kill` để tắt ngẫu nhiên một instance của `agent`.
  3. Gửi tiếp request thứ hai với nội dung hỏi về thông tin đã cung cấp ở request đầu tiên.
  4. **Kết quả**: Hệ thống vẫn phản hồi đúng ngữ cảnh của cuộc trò chuyện do instance còn lại nhận request đã tải thành công lịch sử từ Redis tập trung, chứng minh thiết kế stateless hoạt động hoàn toàn chính xác.
