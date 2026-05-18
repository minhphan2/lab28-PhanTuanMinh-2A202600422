# Hướng Dẫn Chi Tiết Lab 28: Full Platform Integration Sprint

Bài lab này nhằm mục tiêu ghép nối toàn bộ các thành phần công nghệ đã học (Kafka, Prefect, Delta Lake, Feast, Qdrant, Prometheus, Grafana, vLLM) thành một hệ thống AI Platform hoàn chỉnh. 
Hệ thống này sẽ hoạt động theo **kiến trúc Hybrid**: chạy các dịch vụ Backend/Monitoring trên Local (bằng Docker) và chạy mô hình LLM trên Kaggle (tận dụng GPU miễn phí).

Dưới đây là hướng dẫn làm bài từng bước chi tiết:

---

## Bước 1: Khởi động Hạ tầng trên Local (Docker Compose)
Mục tiêu là chạy Kafka, Prefect, Delta Lake, Feast (Redis), Qdrant, Prometheus, Grafana và API Gateway.

1. Mở Terminal / PowerShell và di chuyển vào thư mục Lab 28:
   ```bash
   cd c:\Users\ADMIN\Desktop\lab28-PhanTuanMinh-2A202600422
   ```
2. Khởi động toàn bộ các dịch vụ bằng lệnh:
   ```bash
   docker compose up -d
   ```
3. Chờ khoảng 1 phút cho các dịch vụ khởi động xong, sau đó kiểm tra trạng thái:
   ```bash
   docker compose ps
   ```
   *(Đảm bảo tất cả các container đều hiện trạng thái `Up`)*

> 💡 **Mẹo:** Bạn có thể kiểm tra xem các dịch vụ đã chạy thành công chưa bằng cách truy cập:
> - Prefect UI: [http://localhost:4200](http://localhost:4200)
> - Grafana: [http://localhost:3000](http://localhost:3000) (đăng nhập: `admin` / `admin`)
> - Qdrant Dashboard: [http://localhost:6333/dashboard](http://localhost:6333/dashboard)

---

## Bước 2: Thiết lập GPU Inference trên Kaggle & Tạo Tunnel
Mô hình ngôn ngữ lớn (LLM) và Embedding model cần chạy trên máy có GPU mạnh. Ta sẽ dùng Kaggle để xử lý việc này và mở một "đường hầm" (tunnel) kết nối về máy Local.

1. Đăng nhập vào Kaggle, tạo một Notebook mới.
2. Trong phần **Settings**, bật tùy chọn **Accelerator là GPU T4 x2**.
3. Sử dụng tính năng `cloudflared` để thiết lập dễ dàng nhất. Trong Kaggle Notebook, hãy chạy lần lượt các block code sau:

   **Cell 1: Cài đặt thư viện cần thiết và sửa lỗi hệ thống Kaggle**
   ```python
   !pip install -q vllm fastapi uvicorn mlflow sentence-transformers
   !wget -q -O cloudflared https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
   !chmod +x cloudflared

   # Tìm file gốc libcuda.so.1 và tạo symlink ảo để vLLM có thể biên dịch
   !mkdir -p /kaggle/working/cuda_lib
   !find / -name "libcuda.so.1" 2>/dev/null | head -n 1 | xargs -I {} ln -sf {} /kaggle/working/cuda_lib/libcuda.so
   ```

   **Cell 2: Khởi động vLLM Server**
   ```python
   import subprocess, threading, time, os

   def run_vllm():
       # Đưa đường dẫn chứa libcuda.so vào biến môi trường LIBRARY_PATH
       my_env = os.environ.copy()
       my_env["LIBRARY_PATH"] = f"/kaggle/working/cuda_lib:{my_env.get('LIBRARY_PATH', '')}"
       
       subprocess.run([
           "python", "-m", "vllm.entrypoints.openai.api_server",
           "--model", "Qwen/Qwen2.5-7B-Instruct-GPTQ-Int4",
           "--port", "8001",
           "--max-model-len", "4096",
           "--gpu-memory-utilization", "0.85"
       ], env=my_env)

   thread = threading.Thread(target=run_vllm, daemon=True)
   thread.start()
   time.sleep(60) # Chờ model load lên GPU (có thể mất 1-2 phút)
   print("vLLM server started")
   ```

   **Cell 3: Khởi chạy Cloudflare Tunnel và lấy đường link URL**
   ```python
   import subprocess
   tunnel = subprocess.run(["./cloudflared", "tunnel", "--url", "http://localhost:8001"], capture_output=True, text=True)
   print("Đường link của bạn:")
   print(tunnel.stderr)
   ```
   👉 **Quan trọng:** Ở đầu ra của Cell 3, bạn sẽ thấy một đường link có dạng `https://xxxx.trycloudflare.com`. Hãy **copy lại đường link này**.

4. Trở lại máy tính của bạn, sao chép file `.env.example` thành file `.env` mới.
5. Mở file `.env`, dán đường link bạn vừa copy vào phần `VLLM_NGROK_URL`. (Nếu có tài khoản LangSmith, bạn có thể điền thêm `LANGCHAIN_API_KEY` vào file này).

---

## Bước 3: Chạy Các Kịch Bản Tích Hợp Hệ Thống
Bây giờ hệ thống phần mềm và phần cứng đều đã sẵn sàng, chúng ta sẽ bắt đầu cho dòng dữ liệu chạy xuyên suốt hệ thống.

Mở một Terminal mới (vẫn trong thư mục lab28) và chạy lần lượt các lệnh sau:

1. **Deploy Prefect Flow:** (Chuẩn bị đường ống dẫn dữ liệu)
   ```bash
   cd prefect/flows
   pip install -r requirements.txt
   python kafka_to_delta.py
   cd ../..
   ```

2. **Ingest Data vào Kafka:** (Đưa dữ liệu đầu vào)
   ```bash
   python scripts/01_ingest_to_kafka.py
   ```

3. **Chuyển Feature từ Delta Lake vào Feast (Redis):**
   ```bash
   python scripts/03_delta_to_feast.py
   ```

4. **Tạo Embedding và lưu vào Vector Store (Qdrant):**
   ```bash
   python scripts/05_embed_to_qdrant.py
   ```

---

## Bước 4: Kiểm tra Sức khỏe Hệ Thống (Smoke Tests)
Để đảm bảo tất cả các thành phần giao tiếp được với nhau (API Gateway, Qdrant, Kaggle vLLM...), hãy chạy bộ Smoke Test.

1. Chạy lệnh kiểm thử tự động:
   ```bash
   pytest smoke-tests/ -v
   ```
2. Nếu màn hình trả về báo cáo cả 5/5 tests đều **PASSED**, thì xin chúc mừng, hệ thống của bạn đã hoàn thiện!

> Bạn có thể kiểm tra gửi câu hỏi thủ công trực tiếp bằng lệnh:
> ```bash
> curl -X POST http://localhost:8000/api/v1/chat \
>   -H "Content-Type: application/json" \
>   -d '{ "query": "What is platform engineering?", "embedding": [0.1] }'
> ```

---

## Bước 5: Đánh giá Chuẩn bị Lên Production (Production Readiness Check)
Bước cuối cùng là chạy kịch bản đo lường xem hệ thống có đủ tiêu chuẩn để đi vào hoạt động thực tế hay không:

1. Chạy kịch bản đánh giá:
   ```bash
   python scripts/production_readiness_check.py
   ```
2. Đọc kết quả in ra trên màn hình. Yêu cầu của bài lab là đạt được **điểm số (Score) > 80%**. 

---

## Bước 6: Hoàn tất và Nộp Bài
Khi đã hoàn thành chạy thành công (5/5 tests pass và readiness score > 80%), bạn hãy tiến hành Nộp Bài:
1. Đọc hướng dẫn trong file `SUBMISSION.md` của thư mục Lab 28 để biết chính xác cần nộp ảnh màn hình nào.
2. Thường sẽ cần chụp lại Terminal chứa log chứng minh bạn đã **pass 5/5 Smoke tests** và đạt điểm **Readiness Score trên 80%**.
3. Push toàn bộ code cùng folder ảnh màn hình lên Github Public và copy link dán vào hệ thống VinUni LMS.
