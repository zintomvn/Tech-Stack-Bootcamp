# Chạy Milvus Server Local bằng Docker + Attu UI

README này hướng dẫn chạy **Milvus Standalone** trên máy local bằng **Docker Compose**, mở giao diện **Attu**, và test kết nối bằng Python.

---

## 1. Milvus là gì trong setup này?

Khi chạy Milvus local bằng Docker Compose, bạn thường sẽ có 3 service chính:

```text
milvus-standalone  -> Milvus vector database server chính
milvus-etcd        -> lưu metadata, trạng thái collection/index/segment
milvus-minio       -> lưu object data, segment data
```

Backend hoặc Python client sẽ kết nối đến Milvus qua:

```text
http://localhost:19530
```

Flow tổng quát:

```text
Python / FastAPI / Backend
        ↓
Milvus Client
        ↓
localhost:19530
        ↓
Milvus Standalone
        ↓
etcd + MinIO
```

---

## 2. Yêu cầu trước khi chạy

Cần cài:

- Docker Desktop
- WSL2 nếu dùng Windows
- Docker Compose V2
- Python 3.10+ nếu muốn test bằng Python

Kiểm tra Docker:

```bash
docker --version
docker compose version
```

Nếu 2 lệnh trên chạy được thì có thể tiếp tục.

---

## 3. Tạo thư mục project

Mở **PowerShell**:

```powershell
mkdir milvus-demo
cd milvus-demo
```

---

## 4. Tải file Docker Compose của Milvus

### Cách 1: Dùng PowerShell trên Windows

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/milvus-io/milvus/releases/download/v3.0-beta/milvus-standalone-docker-compose.yml" `
  -OutFile "docker-compose.yml"
```

### Cách 2: Dùng Git Bash / WSL / Linux

```bash
wget https://github.com/milvus-io/milvus/releases/download/v3.0-beta/milvus-standalone-docker-compose.yml -O docker-compose.yml
```

Sau khi tải xong, kiểm tra:

```bash
ls
```

Bạn nên thấy file:

```text
docker-compose.yml
```

---

## 5. Chạy Milvus server

Trong thư mục có file `docker-compose.yml`, chạy:

```bash
docker compose up -d
```

Kiểm tra container:

```bash
docker compose ps
```

Bạn nên thấy các container tương tự:

```text
milvus-etcd
milvus-minio
milvus-standalone
```

Nếu trạng thái là `Up` hoặc `healthy` thì Milvus đã chạy.

---

## 6. Kiểm tra port Milvus

Milvus server mặc định dùng port:

```text
19530
```

Kiểm tra port:

```bash
docker port milvus-standalone 19530/tcp
```

Endpoint local thường là:

```text
http://localhost:19530
```

---

## 7. Test kết nối bằng Python

Cài thư viện Milvus Python client:

```bash
pip install -U pymilvus
```

Tạo file:

```bash
touch test_milvus.py
```

Nếu dùng PowerShell và không có `touch`, dùng:

```powershell
New-Item test_milvus.py
```

Mở file `test_milvus.py` và thêm code:

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")

print("Connected to Milvus")
print("Collections:", client.list_collections())
```

Chạy:

```bash
python test_milvus.py
```

Nếu thành công, output sẽ kiểu:

```text
Connected to Milvus
Collections: []
```

Dấu `[]` nghĩa là hiện tại chưa có collection nào.

---

## 8. Tạo collection test trong Milvus

Tạo file `create_collection.py`:

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")

collection_name = "demo_collection"

if client.has_collection(collection_name):
    client.drop_collection(collection_name)

client.create_collection(
    collection_name=collection_name,
    dimension=4,
    metric_type="COSINE",
)

print("Created collection:", collection_name)
print("Collections:", client.list_collections())
```

Chạy:

```bash
python create_collection.py
```

Nếu thành công:

```text
Created collection: demo_collection
Collections: ['demo_collection']
```

---

## 9. Insert vector test

Tạo file `insert_data.py`:

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")

collection_name = "demo_collection"

data = [
    {
        "id": 1,
        "vector": [0.1, 0.2, 0.3, 0.4],
        "text": "first vector"
    },
    {
        "id": 2,
        "vector": [0.2, 0.1, 0.4, 0.3],
        "text": "second vector"
    },
    {
        "id": 3,
        "vector": [0.9, 0.8, 0.7, 0.6],
        "text": "third vector"
    },
]

result = client.insert(
    collection_name=collection_name,
    data=data
)

print("Insert result:", result)
```

Chạy:

```bash
python insert_data.py
```

---

## 10. Search vector test

Tạo file `search_data.py`:

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")

collection_name = "demo_collection"

query_vector = [0.1, 0.2, 0.3, 0.4]

results = client.search(
    collection_name=collection_name,
    data=[query_vector],
    limit=3,
    output_fields=["text"]
)

print(results)
```

Chạy:

```bash
python search_data.py
```

Kết quả sẽ trả về các vector gần nhất với `query_vector`.

---

## 11. Chạy giao diện Attu cho Milvus

**Attu** là giao diện web để quản lý Milvus.

Chạy Attu bằng Docker:

```bash
docker run -p 8000:3000 zilliz/attu:latest
```

Sau đó mở trình duyệt:

```text
http://localhost:8000
```

Khi Attu hỏi địa chỉ Milvus, nhập:

```text
http://localhost:19530
```

Nếu không kết nối được, thử nhập:

```text
localhost:19530
```

hoặc:

```text
milvus-standalone:19530
```

> Lưu ý: `milvus-standalone:19530` thường dùng khi Attu và Milvus nằm cùng Docker network. Nếu chạy Attu bằng `docker run` riêng, `localhost:19530` thường dễ dùng hơn trên máy host.

---

## 12. Milvus Web UI built-in

Một số phiên bản Milvus có Web UI built-in để quan sát trạng thái hệ thống.

Thử mở:

```text
http://localhost:9091/webui
```

Giao diện này thiên về debug/monitoring hơn là thao tác database.

Dùng để xem:

```text
runtime info
component status
collection detail
segment
task
slow request
config
```

---

## 13. Dừng Milvus

Dừng container nhưng giữ data:

```bash
docker compose down
```

Chạy lại:

```bash
docker compose up -d
```

---

## 14. Xóa sạch Milvus và data local

Cẩn thận: bước này xóa dữ liệu local của Milvus.

```bash
docker compose down
```

Trên Windows PowerShell:

```powershell
Remove-Item -Recurse -Force .\volumes
```

Trên Linux / WSL:

```bash
sudo rm -rf volumes
```

---

## 15. Lỗi thường gặp

### Lỗi 1: Docker chưa chạy

Nếu chạy:

```bash
docker ps
```

mà lỗi, hãy mở Docker Desktop trước.

---

### Lỗi 2: Port 19530 bị chiếm

Kiểm tra trên Windows:

```powershell
netstat -ano | findstr 19530
```

Nếu có process đang dùng port này, bạn cần tắt process đó hoặc đổi port trong `docker-compose.yml`.

---

### Lỗi 3: Milvus container chưa healthy

Kiểm tra log:

```bash
docker compose logs milvus-standalone
```

Hoặc xem log toàn bộ:

```bash
docker compose logs
```

Sau đó restart:

```bash
docker compose down
docker compose up -d
```

---

### Lỗi 4: Python không kết nối được

Kiểm tra Milvus có chạy không:

```bash
docker compose ps
```

Kiểm tra endpoint:

```text
http://localhost:19530
```

Thử restart:

```bash
docker compose down
docker compose up -d
```

---

### Lỗi 5: Attu không kết nối được Milvus

Thử các địa chỉ:

```text
http://localhost:19530
localhost:19530
127.0.0.1:19530
```

Nếu Attu chạy trong Docker network riêng và không thấy Milvus, cách ổn hơn là đưa Attu vào cùng network với Docker Compose của Milvus.

Xem network:

```bash
docker network ls
```

Chạy Attu trong cùng network, ví dụ:

```bash
docker run -p 8000:3000 --network milvus-demo_default zilliz/attu:latest
```

Sau đó trong Attu nhập:

```text
milvus-standalone:19530
```

Tên network có thể khác tùy tên folder project.

---

## 16. Khi nào dùng Milvus?

Milvus phù hợp khi bạn cần:

```text
vector search lớn
image retrieval
video retrieval
text retrieval
RAG
multimodal search
AI Challenge / Lifelog / Video Browser Showdown style system
```

Ví dụ pipeline video retrieval:

```text
Video
  ↓
Cắt frame / shot / segment
  ↓
Encode bằng CLIP / SigLIP / InternVL / ImageBind
  ↓
Tạo vector embedding
  ↓
Insert vector vào Milvus
  ↓
User query text
  ↓
Encode query thành vector
  ↓
Milvus ANN search
  ↓
Trả về frame/video/timestamp gần nhất
```

---

## 17. Lệnh nhanh tổng hợp

```bash
# Tạo folder
mkdir milvus-demo
cd milvus-demo

# Tải docker-compose.yml bằng PowerShell
Invoke-WebRequest `
  -Uri "https://github.com/milvus-io/milvus/releases/download/v3.0-beta/milvus-standalone-docker-compose.yml" `
  -OutFile "docker-compose.yml"

# Chạy Milvus
docker compose up -d

# Kiểm tra
docker compose ps

# Cài Python client
pip install -U pymilvus

# Chạy Attu UI
docker run -p 8000:3000 zilliz/attu:latest
```

Mở Attu:

```text
http://localhost:8000
```

Milvus endpoint:

```text
http://localhost:19530
```
