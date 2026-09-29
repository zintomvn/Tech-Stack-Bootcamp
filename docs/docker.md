# Docker: Cơ sở lý thuyết, kiến trúc và thực hành

## 1. Mục tiêu tài liệu

Tài liệu này trình bày Docker theo hướng lý thuyết kết hợp thực hành, giúp người học nắm được:

- Docker là gì và vì sao container quan trọng trong phát triển, kiểm thử và triển khai phần mềm hiện đại.
- Sự khác nhau giữa container, virtual machine, image, container runtime và Docker Engine.
- Các khái niệm cốt lõi như image, container, layer, registry, Dockerfile, volume, network, port mapping và Docker Compose.
- Cách Docker đóng gói ứng dụng cùng dependency để chạy nhất quán trên nhiều môi trường.
- Cách viết Dockerfile, build image, chạy container và quản lý dữ liệu bền vững.
- Cách dùng Docker Compose để chạy nhiều service như backend, database, cache và message queue.
- Các lỗi thiết kế thường gặp và cách tránh khi học hoặc triển khai Docker trong dự án thực tế.

Tài liệu này phù hợp để học nền tảng Docker. Một số cú pháp hoặc tính năng nâng cao có thể thay đổi theo phiên bản Docker Engine, Docker Desktop hoặc Docker Compose, vì vậy khi làm dự án thực tế nên đối chiếu thêm với tài liệu chính thức của Docker đúng với phiên bản đang dùng.

## 2. Tổng quan về Docker

Docker là nền tảng dùng để đóng gói, phân phối và chạy ứng dụng trong **container**. Container chứa ứng dụng, thư viện, dependency, biến môi trường và các file cần thiết để ứng dụng chạy được một cách nhất quán trên nhiều máy khác nhau.

Vấn đề Docker giải quyết rất thực tế:

- Ứng dụng chạy được trên máy lập trình viên nhưng lỗi trên server.
- Mỗi thành viên cài dependency khác phiên bản.
- Cài PostgreSQL, Redis, Elasticsearch hoặc RabbitMQ thủ công mất thời gian.
- Môi trường development, testing và production khác nhau.
- Triển khai nhiều service trên cùng một máy dễ xung đột port, thư viện và cấu hình.

Docker giúp mô tả môi trường chạy bằng code. Thay vì viết hướng dẫn dài kiểu "cài Python 3.12, tạo virtual environment, cài PostgreSQL, sửa config, chạy migration", ta có thể dùng Dockerfile và Docker Compose để người khác chạy lại môi trường gần như giống nhau.

Docker thường được dùng cho:

- Đóng gói backend API, frontend, worker và cron job.
- Chạy database, cache, search engine và message broker trong môi trường học tập hoặc development.
- Tạo môi trường kiểm thử tự động trong CI/CD.
- Triển khai microservices.
- Chuẩn hóa môi trường giữa local, staging và production.
- Chạy các công cụ hạ tầng như PostgreSQL, MySQL, MongoDB, Redis, Elasticsearch, Qdrant, Milvus, Nginx.
- Làm nền tảng để học Kubernetes và cloud-native deployment.

### 2.1. Đặc điểm nổi bật

| Đặc điểm                | Ý nghĩa                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| Đóng gói môi trường     | Ứng dụng và dependency được đóng gói thành image.                                           |
| Chạy nhất quán          | Container chạy giống nhau hơn giữa máy local, CI và server.                                 |
| Nhẹ hơn virtual machine | Container chia sẻ kernel của host nên khởi động nhanh và tốn ít tài nguyên hơn VM.          |
| Layer cache             | Build image nhanh hơn nhờ tái sử dụng các layer không đổi.                                  |
| Registry                | Image có thể được push/pull qua Docker Hub, GitHub Container Registry hoặc registry nội bộ. |
| Port mapping            | Cho phép ánh xạ port trong container ra máy host.                                           |
| Volume                  | Lưu dữ liệu bền vững ngoài vòng đời container.                                              |
| Network                 | Các container có thể giao tiếp với nhau qua mạng riêng.                                     |
| Docker Compose          | Mô tả và chạy nhiều container bằng một file YAML.                                           |

## 3. Cơ sở lý thuyết

### 3.1. Container

Container là một môi trường chạy cô lập cho process. Mỗi container thường chạy một ứng dụng hoặc một service chính, ví dụ:

- Một container chạy FastAPI.
- Một container chạy PostgreSQL.
- Một container chạy Redis.
- Một container chạy Nginx.
- Một container chạy worker xử lý background job.

Container không phải là một máy ảo đầy đủ. Nó chia sẻ kernel với hệ điều hành host, nhưng được cô lập về process, filesystem, network và tài nguyên.

Ví dụ khi chạy:

```bash
docker run nginx
```

Docker sẽ tạo một container từ image `nginx`. Bên trong container có các file và lệnh cần thiết để Nginx chạy. Bên ngoài host, ta không cần cài Nginx trực tiếp.

### 3.2. Image

Image là bản đóng gói bất biến dùng để tạo container. Có thể hiểu:

```text
Image là khuôn mẫu.
Container là instance đang chạy từ khuôn mẫu đó.
```

Ví dụ:

- Image `postgres:16` có thể tạo nhiều container PostgreSQL.
- Image `redis:7` có thể tạo nhiều container Redis.
- Image `my-api:latest` có thể tạo nhiều container backend API.

Một image thường chứa:

- Base operating system tối giản hoặc runtime.
- Thư viện hệ thống cần thiết.
- Runtime như Python, Node.js, Java hoặc Go binary.
- Source code hoặc file build của ứng dụng.
- Dependency đã cài.
- Lệnh mặc định để chạy ứng dụng.

Image nên được build một cách có kiểm soát và có version tag rõ ràng. Không nên chỉ phụ thuộc vào tag `latest` trong production vì khó biết chính xác image đang dùng là bản nào.

### 3.3. Virtual machine và container

Virtual machine và container đều giúp cô lập môi trường chạy, nhưng cách hoạt động khác nhau.

| Tiêu chí   | Virtual machine                             | Container                            |
| ---------- | ------------------------------------------- | ------------------------------------ |
| Mức cô lập | Cô lập cả hệ điều hành khách                | Cô lập process trên cùng kernel host |
| Kernel     | Mỗi VM có kernel riêng                      | Chia sẻ kernel với host              |
| Kích thước | Thường lớn hơn                              | Thường nhỏ hơn                       |
| Khởi động  | Chậm hơn                                    | Nhanh hơn                            |
| Tài nguyên | Tốn RAM/CPU hơn                             | Nhẹ hơn                              |
| Use case   | Cần cô lập mạnh hoặc chạy OS khác hoàn toàn | Đóng gói và chạy ứng dụng hiện đại   |

Container không thay thế VM trong mọi trường hợp. VM vẫn hữu ích khi cần cô lập ở mức hệ điều hành hoặc chạy nhiều kernel khác nhau. Docker phù hợp khi cần đóng gói ứng dụng và dependency để triển khai nhanh, nhẹ và nhất quán.

### 3.4. Layer và copy-on-write

Docker image được xây dựng từ nhiều layer. Mỗi lệnh quan trọng trong Dockerfile như `FROM`, `RUN`, `COPY` thường tạo thêm layer mới.

Ví dụ:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

Image có thể gồm các layer:

1. Base image `python:3.12-slim`.
2. Thư mục làm việc `/app`.
3. File `requirements.txt`.
4. Dependency Python đã cài.
5. Source code ứng dụng.
6. Metadata lệnh chạy mặc định.

Khi build lại, Docker có thể dùng cache cho các layer không đổi. Đây là lý do nên copy file dependency trước rồi mới copy toàn bộ source code. Nếu source code đổi nhưng `requirements.txt` không đổi, Docker có thể không cần cài lại dependency.

Container được tạo từ image bằng cơ chế copy-on-write. Image là bất biến, còn container có một writable layer riêng. Khi container ghi file, thay đổi nằm ở layer của container. Nếu xóa container, writable layer này cũng mất, trừ khi dữ liệu được lưu vào volume hoặc bind mount.

### 3.5. Registry

Registry là nơi lưu trữ và phân phối Docker image. Một số registry phổ biến:

- Docker Hub.
- GitHub Container Registry.
- GitLab Container Registry.
- Amazon Elastic Container Registry.
- Google Artifact Registry.
- Azure Container Registry.
- Registry nội bộ của doanh nghiệp.

Các thao tác thường gặp:

```bash
docker pull postgres:16
docker tag my-api:1.0 username/my-api:1.0
docker push username/my-api:1.0
```

Trong dự án thực tế, CI/CD thường build image, gắn tag theo version hoặc commit SHA, sau đó push image lên registry để server hoặc Kubernetes pull về chạy.

### 3.6. Docker Engine

Docker Engine là thành phần chính giúp build, chạy và quản lý container. Nó gồm:

| Thành phần        | Vai trò                                                          |
| ----------------- | ---------------------------------------------------------------- |
| Docker CLI        | Công cụ dòng lệnh như `docker run`, `docker build`, `docker ps`. |
| Docker daemon     | Dịch vụ nền quản lý image, container, network và volume.         |
| REST API          | API để CLI hoặc công cụ khác giao tiếp với Docker daemon.        |
| Container runtime | Thành phần chạy container ở mức thấp hơn.                        |

Khi chạy lệnh:

```bash
docker run nginx
```

Docker CLI gửi request đến Docker daemon. Docker daemon kiểm tra image, pull image nếu chưa có, tạo container, cấu hình network/filesystem và yêu cầu runtime chạy process chính trong container.

### 3.7. Namespace và cgroups

Docker dựa trên các cơ chế của Linux để cô lập container.

| Cơ chế           | Ý nghĩa                                                                   |
| ---------------- | ------------------------------------------------------------------------- |
| Namespace        | Cô lập process, network, mount, hostname, user và IPC.                    |
| cgroups          | Giới hạn và theo dõi tài nguyên như CPU, RAM, disk I/O.                   |
| Union filesystem | Kết hợp nhiều layer thành filesystem nhìn như một hệ thống file duy nhất. |

Nhờ namespace, process trong container có thể thấy danh sách process riêng, hostname riêng và network interface riêng. Nhờ cgroups, Docker có thể giới hạn container dùng tối đa bao nhiêu CPU hoặc RAM.

Ví dụ giới hạn tài nguyên:

```bash
docker run --memory 512m --cpus 1 nginx
```

### 3.8. Port mapping

Ứng dụng trong container thường lắng nghe trên một port bên trong container. Để truy cập từ máy host, cần ánh xạ port.

Ví dụ:

```bash
docker run -p 8080:80 nginx
```

Ý nghĩa:

```text
host port 8080 -> container port 80
```

Khi mở:

```text
http://localhost:8080
```

request sẽ được chuyển vào port `80` của container Nginx.

Nếu không map port, container vẫn có thể chạy, nhưng từ máy host không truy cập được service qua port đó theo cách thông thường.

### 3.9. Volume và bind mount

Container có writable layer riêng, nhưng layer này mất khi container bị xóa. Để lưu dữ liệu bền vững, Docker dùng volume hoặc bind mount.

| Cách lưu dữ liệu | Ý nghĩa                                  | Khi dùng                                     |
| ---------------- | ---------------------------------------- | -------------------------------------------- |
| Volume           | Docker quản lý vị trí lưu trữ            | Dữ liệu database, dữ liệu cần bền vững       |
| Bind mount       | Gắn trực tiếp thư mục host vào container | Development, mount source code, config local |
| tmpfs mount      | Lưu trong RAM, không ghi xuống disk      | Dữ liệu tạm, nhạy cảm, cache ngắn hạn        |

Ví dụ volume cho PostgreSQL:

```bash
docker run --name postgres-demo \
  -e POSTGRES_PASSWORD=postgres \
  -v postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

Nếu container bị xóa, volume `postgres_data` vẫn còn và có thể dùng lại.

### 3.10. Docker network

Docker network cho phép container giao tiếp với nhau. Khi dùng Docker Compose, các service trong cùng project thường tự nằm trong một network chung và có thể gọi nhau bằng tên service.

Ví dụ trong Compose:

```yaml
services:
  api:
    build: .
    depends_on:
      - db

  db:
    image: postgres:16
```

Container `api` có thể kết nối PostgreSQL bằng hostname:

```text
db
```

Không nên dùng `localhost` để container này gọi container khác. Bên trong container, `localhost` là chính container đó, không phải container khác và cũng không phải máy host theo nghĩa thông thường.

## 4. Kiến trúc Docker

### 4.1. Sơ đồ kiến trúc Mermaid

```mermaid
flowchart TD
    User[User / Developer] --> CLI[Docker CLI]
    CLI -->|REST API| Daemon[Docker Daemon]
    Daemon --> Images[Local Images]
    Daemon --> Containers[Containers]
    Daemon --> Networks[Docker Networks]
    Daemon --> Volumes[Docker Volumes]
    Daemon --> Runtime[Container Runtime]
    Runtime --> Kernel[Host OS Kernel]
    Daemon -->|pull / push| Registry[Image Registry]
    Dockerfile[Dockerfile] -->|docker build| Daemon
    Compose[compose.yaml] -->|docker compose up| CLI
```

Kiến trúc trên cho thấy người dùng thường làm việc qua Docker CLI hoặc Docker Compose. CLI gửi lệnh đến Docker daemon. Daemon quản lý image, container, volume, network và giao tiếp với container runtime để chạy process trong container.

### 4.2. Các thành phần quan trọng

| Thành phần     | Vai trò                                            |
| -------------- | -------------------------------------------------- |
| Docker CLI     | Công cụ để người dùng chạy lệnh Docker.            |
| Docker daemon  | Dịch vụ nền quản lý vòng đời container và image.   |
| Dockerfile     | File mô tả cách build image.                       |
| Image          | Gói bất biến chứa ứng dụng và dependency.          |
| Container      | Instance đang chạy từ image.                       |
| Registry       | Nơi lưu và phân phối image.                        |
| Volume         | Nơi lưu dữ liệu bền vững ngoài vòng đời container. |
| Network        | Lớp mạng cho container giao tiếp.                  |
| Docker Compose | Công cụ chạy nhiều container bằng file YAML.       |
| Build context  | Tập file được gửi vào quá trình build image.       |

### 4.3. Vòng đời image

```mermaid
flowchart LR
    Dockerfile[Dockerfile] --> Build[docker build]
    Build --> Image[Local Image]
    Image --> Run[docker run]
    Run --> Container[Running Container]
    Image --> Tag[docker tag]
    Tag --> Push[docker push]
    Push --> Registry[Registry]
    Registry --> Pull[docker pull]
    Pull --> OtherHost[Other Machine]
```

Vòng đời image thường bắt đầu từ Dockerfile. Sau khi build thành image, image có thể chạy local, được gắn tag, push lên registry và pull về máy khác để chạy.

### 4.4. Vòng đời container

```mermaid
stateDiagram-v2
    [*] --> Created
    Created --> Running: docker start
    Running --> Paused: docker pause
    Paused --> Running: docker unpause
    Running --> Exited: process exits / docker stop
    Exited --> Running: docker start
    Exited --> Removed: docker rm
    Created --> Removed: docker rm
```

Container có thể được tạo, chạy, dừng, khởi động lại và xóa. Khi container bị xóa, dữ liệu trong writable layer mất. Dữ liệu trong volume vẫn còn.

## 5. Vòng đời xử lý với Docker

### 5.1. Luồng build image

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant CLI as Docker CLI
    participant D as Docker Daemon
    participant F as Dockerfile
    participant C as Build Context
    participant I as Image

    Dev->>CLI: docker build -t my-api:1.0 .
    CLI->>D: Gửi Dockerfile và build context
    D->>F: Đọc từng instruction
    D->>C: Copy file cần thiết
    D->>D: Tạo layer và dùng cache nếu có
    D-->>I: Sinh image my-api:1.0
    D-->>CLI: Build thành công
```

Điểm cần chú ý là Docker gửi build context vào daemon. Nếu thư mục project có file rất lớn như dataset, log, virtual environment hoặc `node_modules`, quá trình build có thể chậm. Vì vậy nên dùng `.dockerignore`.

### 5.2. Luồng chạy container

```mermaid
sequenceDiagram
    participant User as User
    participant CLI as Docker CLI
    participant D as Docker Daemon
    participant R as Registry
    participant C as Container

    User->>CLI: docker run -p 8000:8000 my-api:1.0
    CLI->>D: Yêu cầu tạo và chạy container
    D->>D: Kiểm tra image local
    D->>R: Pull image nếu chưa có
    R-->>D: Trả image layers
    D->>D: Tạo filesystem, network, env, volume
    D->>C: Chạy process chính
    C-->>D: Ghi log stdout/stderr
    D-->>CLI: Trả container id hoặc log
```

Container sống theo process chính. Nếu process chính kết thúc, container chuyển sang trạng thái exited. Đây là lý do container không nên chạy một lệnh kết thúc ngay nếu mục tiêu là giữ service hoạt động.

### 5.3. Luồng Docker Compose

```mermaid
sequenceDiagram
    participant User as User
    participant Compose as Docker Compose
    participant D as Docker Daemon
    participant API as API Container
    participant DB as DB Container
    participant V as Volume
    participant N as Network

    User->>Compose: docker compose up
    Compose->>D: Đọc compose.yaml
    D->>N: Tạo network
    D->>V: Tạo volume nếu chưa có
    D->>DB: Tạo và chạy database container
    D->>API: Build image và chạy API container
    API->>DB: Kết nối qua hostname db
    API-->>User: Service sẵn sàng qua port mapping
```

Docker Compose giúp mô tả cả hệ thống nhỏ bằng một file. Đây là công cụ rất hữu ích khi học backend, database, cache, search engine và message queue.

## 6. Các khái niệm cốt lõi

### 6.1. Dockerfile

Dockerfile là file mô tả cách build image. Một Dockerfile thường gồm:

- Base image.
- Thư mục làm việc.
- File dependency.
- Lệnh cài dependency.
- Source code.
- Port được document bằng `EXPOSE`.
- Lệnh chạy mặc định.

Ví dụ Dockerfile cho ứng dụng Python đơn giản:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "main.py"]
```

Một số instruction phổ biến:

| Instruction   | Ý nghĩa                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| `FROM`        | Chọn base image.                                                                                             |
| `WORKDIR`     | Đặt thư mục làm việc trong image/container.                                                                  |
| `COPY`        | Copy file từ build context vào image.                                                                        |
| `ADD`         | Copy file, có thêm một số chức năng như giải nén tar hoặc tải URL, nhưng thường nên dùng `COPY` cho rõ ràng. |
| `RUN`         | Chạy lệnh trong quá trình build image.                                                                       |
| `ENV`         | Khai báo biến môi trường.                                                                                    |
| `ARG`         | Khai báo biến dùng lúc build.                                                                                |
| `EXPOSE`      | Ghi chú port ứng dụng lắng nghe.                                                                             |
| `CMD`         | Lệnh mặc định khi container chạy.                                                                            |
| `ENTRYPOINT`  | Lệnh chính cố định của container.                                                                            |
| `USER`        | Chọn user chạy process.                                                                                      |
| `HEALTHCHECK` | Khai báo cách kiểm tra container còn khỏe hay không.                                                         |

### 6.2. Build context

Build context là thư mục được gửi vào Docker daemon khi build.

Ví dụ:

```bash
docker build -t my-api .
```

Dấu `.` nghĩa là thư mục hiện tại là build context. Dockerfile chỉ có thể `COPY` file nằm trong build context.

Nếu build context chứa nhiều file thừa, build sẽ chậm và image có thể vô tình chứa dữ liệu nhạy cảm. Vì vậy nên tạo `.dockerignore`.

### 6.3. .dockerignore

`.dockerignore` hoạt động tương tự `.gitignore`, nhưng dành cho Docker build context.

Ví dụ:

```text
.git
.venv
venv
__pycache__
*.pyc
.pytest_cache
node_modules
dist
build
.env
logs
*.log
```

Lợi ích:

- Build nhanh hơn.
- Tránh copy file thừa vào image.
- Giảm kích thước image.
- Giảm nguy cơ lộ `.env`, secret hoặc file local.

### 6.4. Image tag

Image tag là nhãn để phân biệt phiên bản image.

Ví dụ:

```text
my-api:1.0
my-api:2026-06-03
my-api:dev
my-api:commit-a1b2c3d
```

Nếu không ghi tag, Docker thường hiểu là `latest`:

```bash
docker pull nginx
```

tương đương:

```bash
docker pull nginx:latest
```

Trong production, nên dùng tag cụ thể như version, git commit SHA hoặc image digest để triển khai có thể truy vết và rollback.

### 6.5. Container name

Mỗi container có ID và có thể có tên.

```bash
docker run --name web-demo nginx
```

Tên container giúp thao tác dễ hơn:

```bash
docker stop web-demo
docker logs web-demo
docker exec -it web-demo sh
```

Nếu không đặt tên, Docker tự sinh tên ngẫu nhiên. Khi học hoặc demo, đặt tên giúp dễ quản lý. Trong Docker Compose, tên service thường quan trọng hơn tên container cụ thể.

### 6.6. Environment variables

Biến môi trường giúp truyền cấu hình vào container mà không hard-code trong image.

Ví dụ:

```bash
docker run \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=school_db \
  postgres:16
```

Trong ứng dụng backend, biến môi trường thường chứa:

- `DATABASE_URL`
- `REDIS_URL`
- `JWT_SECRET`
- `LOG_LEVEL`
- `APP_ENV`
- `PORT`

Không nên đưa secret thật vào Dockerfile vì image có thể được chia sẻ hoặc lưu trong registry.

### 6.7. Port mapping

Cú pháp:

```bash
docker run -p HOST_PORT:CONTAINER_PORT image_name
```

Ví dụ:

```bash
docker run -p 8080:80 nginx
```

Một số lỗi thường gặp:

- Ứng dụng trong container lắng nghe port `8000` nhưng lại map `8080:80`.
- Port host đã bị ứng dụng khác chiếm.
- Ứng dụng chỉ bind vào `127.0.0.1` trong container thay vì `0.0.0.0`.

Với web server trong container, thường nên lắng nghe trên `0.0.0.0` để Docker có thể chuyển request từ host vào container.

### 6.8. Volume

Volume do Docker quản lý và thường dùng cho dữ liệu cần bền vững.

Tạo volume:

```bash
docker volume create postgres_data
```

Dùng volume:

```bash
docker run --name db \
  -e POSTGRES_PASSWORD=postgres \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16
```

Xem volume:

```bash
docker volume ls
docker volume inspect postgres_data
```

Volume đặc biệt quan trọng với database. Nếu chạy database mà không dùng volume, khi xóa container có thể mất dữ liệu.

### 6.9. Bind mount

Bind mount gắn một file hoặc thư mục từ host vào container.

Ví dụ mount source code vào container Python:

```bash
docker run -v .:/app -w /app python:3.12-slim python main.py
```

Bind mount phù hợp cho development vì sửa code trên host thì container thấy ngay. Tuy nhiên, bind mount phụ thuộc đường dẫn host nên kém portable hơn volume.

### 6.10. Docker network

Docker hỗ trợ nhiều loại network:

| Network driver | Ý nghĩa                                                                                      |
| -------------- | -------------------------------------------------------------------------------------------- |
| `bridge`       | Network mặc định phổ biến cho container trên một host.                                       |
| `host`         | Container dùng trực tiếp network của host, thường dùng trên Linux trong trường hợp đặc biệt. |
| `none`         | Container không có network.                                                                  |
| `overlay`      | Network nhiều host, thường dùng với Docker Swarm.                                            |
| `macvlan`      | Container xuất hiện như thiết bị riêng trong mạng vật lý.                                    |

Tạo network:

```bash
docker network create app_network
```

Chạy container trong network:

```bash
docker run --name redis --network app_network redis:7
docker run --name app --network app_network my-api
```

Container `app` có thể gọi Redis bằng hostname `redis`.

### 6.11. Logs

Docker thu log từ `stdout` và `stderr` của process chính trong container.

Xem log:

```bash
docker logs container_name
docker logs -f container_name
```

Ứng dụng chạy trong container nên ghi log ra `stdout`/`stderr` thay vì chỉ ghi vào file nội bộ. Cách này giúp Docker, Compose, Kubernetes hoặc hệ thống logging thu log dễ hơn.

### 6.12. Exec

`docker exec` dùng để chạy lệnh trong container đang chạy.

Ví dụ vào shell:

```bash
docker exec -it container_name sh
```

Hoặc với image có bash:

```bash
docker exec -it container_name bash
```

Ví dụ kiểm tra biến môi trường:

```bash
docker exec container_name env
```

`exec` hữu ích khi debug, nhưng không nên sửa file thủ công trong container rồi xem đó là cách triển khai. Nếu cần thay đổi bền vững, hãy sửa Dockerfile hoặc source code rồi build lại image.

### 6.13. Healthcheck

Healthcheck giúp Docker biết container có còn khỏe không.

Ví dụ trong Dockerfile:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
```

Trong Compose:

```yaml
services:
  api:
    build: .
    healthcheck:
      test:
        [
          "CMD",
          "python",
          "-c",
          "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')",
        ]
      interval: 30s
      timeout: 3s
      retries: 3
```

Healthcheck không tự sửa lỗi ứng dụng, nhưng giúp hệ thống giám sát và orchestrator biết trạng thái container.

## 7. Các lệnh Docker cơ bản

### 7.1. Kiểm tra Docker

```bash
docker --version
docker info
```

Chạy container thử:

```bash
docker run hello-world
```

Lệnh này pull image `hello-world` nếu máy chưa có, tạo container, in thông báo kiểm tra và kết thúc.

### 7.2. Quản lý image

Liệt kê image:

```bash
docker images
```

Pull image:

```bash
docker pull nginx:stable
```

Build image:

```bash
docker build -t my-api:1.0 .
```

Xóa image:

```bash
docker rmi my-api:1.0
```

Chỉ nên xóa image khi chắc chắn không còn container nào cần image đó.

### 7.3. Quản lý container

Chạy container foreground:

```bash
docker run nginx
```

Chạy container background:

```bash
docker run -d --name web -p 8080:80 nginx
```

Liệt kê container đang chạy:

```bash
docker ps
```

Liệt kê cả container đã dừng:

```bash
docker ps -a
```

Dừng container:

```bash
docker stop web
```

Khởi động lại container:

```bash
docker start web
```

Xóa container đã dừng:

```bash
docker rm web
```

### 7.4. Xem log và debug

Xem log:

```bash
docker logs web
docker logs -f web
```

Xem thông tin chi tiết:

```bash
docker inspect web
```

Vào shell:

```bash
docker exec -it web sh
```

Xem thống kê tài nguyên:

```bash
docker stats
```

### 7.5. Quản lý volume

```bash
docker volume ls
docker volume create app_data
docker volume inspect app_data
```

Dùng volume:

```bash
docker run -v app_data:/data busybox sh -c "echo hello > /data/message.txt"
```

Khi học, cần cẩn thận với lệnh xóa volume vì volume có thể chứa dữ liệu database.

### 7.6. Quản lý network

```bash
docker network ls
docker network create app_network
docker network inspect app_network
```

Chạy container trong network:

```bash
docker run -d --name redis --network app_network redis:7
docker run -it --rm --network app_network redis:7 redis-cli -h redis ping
```

Nếu kết nối thành công, Redis trả:

```text
PONG
```

## 8. Dockerfile trong thực tế

### 8.1. Dockerfile cho Python/FastAPI

Ví dụ ứng dụng FastAPI có cấu trúc:

```text
app/
├── main.py
├── requirements.txt
└── Dockerfile
```

`requirements.txt`:

```text
fastapi
uvicorn[standard]
```

`main.py`:

```python
from fastapi import FastAPI


app = FastAPI()


@app.get("/health")
def health():
    return {"status": "ok"}
```

Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Build và chạy:

```bash
docker build -t fastapi-demo .
docker run --name fastapi-demo -p 8000:8000 fastapi-demo
```

Truy cập:

```text
http://localhost:8000/health
```

### 8.2. Dockerfile cho Node.js

Ví dụ Dockerfile cho ứng dụng Node.js:

```dockerfile
FROM node:22-slim

WORKDIR /app

COPY package*.json .
RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

Điểm quan trọng:

- Copy `package.json` và lock file trước để tận dụng cache.
- Dùng `npm ci` trong CI/build để cài dependency theo lock file.
- Không copy `node_modules` từ host vào image.

`.dockerignore`:

```text
node_modules
npm-debug.log
.git
.env
dist
coverage
```

### 8.3. Multi-stage build

Multi-stage build giúp image cuối nhỏ hơn bằng cách tách giai đoạn build và runtime.

Ví dụ với ứng dụng Go:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app .

FROM debian:bookworm-slim

WORKDIR /app
COPY --from=builder /src/app .

EXPOSE 8080

CMD ["./app"]
```

Image cuối không cần chứa toàn bộ Go compiler và source build cache. Cách này giúp:

- Giảm kích thước image.
- Giảm bề mặt tấn công.
- Tăng tốc pull image.
- Tách rõ build-time dependency và runtime dependency.

### 8.4. CMD và ENTRYPOINT

`CMD` và `ENTRYPOINT` đều liên quan đến lệnh chạy container, nhưng khác nhau.

| Instruction  | Ý nghĩa                                                         |
| ------------ | --------------------------------------------------------------- |
| `CMD`        | Lệnh mặc định, dễ bị override khi `docker run image command`.   |
| `ENTRYPOINT` | Lệnh chính cố định hơn, tham số truyền vào thường được nối sau. |

Ví dụ `CMD`:

```dockerfile
CMD ["python", "main.py"]
```

Chạy:

```bash
docker run my-image python other.py
```

lệnh mặc định bị thay bằng `python other.py`.

Ví dụ `ENTRYPOINT`:

```dockerfile
ENTRYPOINT ["python", "main.py"]
```

`ENTRYPOINT` phù hợp khi image được thiết kế như một executable cố định.

### 8.5. ARG và ENV

`ARG` dùng trong lúc build, `ENV` tồn tại trong image và container.

Ví dụ:

```dockerfile
ARG APP_VERSION=dev
ENV APP_ENV=production

RUN echo "Building version ${APP_VERSION}"
```

Build với argument:

```bash
docker build --build-arg APP_VERSION=1.0.0 -t my-api:1.0.0 .
```

Chạy với environment variable:

```bash
docker run -e APP_ENV=development my-api:1.0.0
```

Không nên đưa secret vào `ARG` hoặc `ENV` trong Dockerfile nếu image có thể bị chia sẻ hoặc lưu ở registry.

### 8.6. User không phải root

Mặc định nhiều image chạy process bằng user `root`. Trong production, nên chạy bằng user ít quyền hơn nếu có thể.

Ví dụ:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN useradd -m appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER appuser

CMD ["python", "main.py"]
```

Chạy non-root giúp giảm rủi ro nếu ứng dụng bị khai thác lỗi bảo mật.

## 9. Docker Compose

### 9.1. Docker Compose là gì

Docker Compose là công cụ để định nghĩa và chạy nhiều container bằng một file YAML, thường tên là `compose.yaml` hoặc `docker-compose.yml`.

Thay vì chạy nhiều lệnh:

```bash
docker network create app_network
docker volume create postgres_data
docker run ...
docker run ...
```

ta có thể viết:

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
```

rồi chạy:

```bash
docker compose up
```

### 9.2. Các khái niệm trong Compose

| Khái niệm     | Ý nghĩa                                  |
| ------------- | ---------------------------------------- |
| `services`    | Danh sách container/service cần chạy.    |
| `image`       | Image dùng cho service.                  |
| `build`       | Cấu hình build image từ Dockerfile.      |
| `ports`       | Port mapping từ host vào container.      |
| `environment` | Biến môi trường truyền vào container.    |
| `volumes`     | Volume hoặc bind mount.                  |
| `networks`    | Network cho service.                     |
| `depends_on`  | Thứ tự khởi tạo service ở mức container. |
| `healthcheck` | Kiểm tra trạng thái service.             |
| `restart`     | Chính sách restart container.            |

### 9.3. Ví dụ Compose với FastAPI và PostgreSQL

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/app_db
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Chạy:

```bash
docker compose up
```

Chạy nền:

```bash
docker compose up -d
```

Dừng:

```bash
docker compose down
```

Lưu ý: `depends_on` chỉ đảm bảo container `db` được tạo/chạy trước `api`, không đảm bảo PostgreSQL đã sẵn sàng nhận kết nối. Với hệ thống cần chắc chắn, nên dùng healthcheck hoặc logic retry trong ứng dụng.

### 9.4. Compose với healthcheck

Ví dụ thêm healthcheck cho PostgreSQL:

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/app_db
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d app_db"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  postgres_data:
```

Healthcheck giúp service `api` chờ database khỏe hơn trước khi khởi động.

### 9.5. Compose cho môi trường development

Trong development, thường dùng bind mount để sửa code không cần build lại image liên tục.

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Lưu ý:

- Bind mount giúp code thay đổi ngay trong container.
- `--reload` phù hợp development, không nên dùng trong production.
- Tránh mount đè thư mục dependency nếu dependency được cài trong image.

### 9.6. Các lệnh Compose thường dùng

```bash
docker compose up
docker compose up -d
docker compose down
docker compose ps
docker compose logs
docker compose logs -f api
docker compose exec api sh
docker compose build
docker compose pull
docker compose restart api
```

Xóa container và network của Compose:

```bash
docker compose down
```

Xóa cả volume:

```bash
docker compose down -v
```

Cần cẩn thận với `down -v` vì nó có thể xóa dữ liệu database trong volume của project.

## 10. Docker trong quy trình phát triển phần mềm

### 10.1. Development

Trong development, Docker giúp:

- Chạy database và service phụ thuộc nhanh.
- Đồng bộ môi trường giữa các thành viên.
- Tránh cài quá nhiều công cụ trực tiếp lên máy.
- Dễ reset môi trường.
- Dễ thử nhiều phiên bản database hoặc runtime.

Ví dụ một project backend có thể dùng Compose để chạy:

- API service.
- PostgreSQL.
- Redis.
- Elasticsearch.
- Worker.

Lập trình viên chỉ cần chạy:

```bash
docker compose up
```

### 10.2. Testing

Docker phù hợp cho integration test vì có thể tạo môi trường sạch:

- Tạo database test.
- Chạy migration.
- Seed dữ liệu.
- Chạy test.
- Xóa container và volume test sau khi hoàn thành.

Trong CI/CD, Docker giúp pipeline chạy ổn định hơn vì dependency được mô tả rõ ràng.

### 10.3. Production

Trong production, Docker thường được dùng cùng:

- Docker Compose cho server nhỏ hoặc project đơn giản.
- Docker Swarm cho orchestration cơ bản.
- Kubernetes cho hệ thống lớn hơn.
- Cloud container service như ECS, Cloud Run, Azure Container Apps.

Khi chạy production cần quan tâm:

- Image tag cụ thể.
- Secret management.
- Log và monitoring.
- Healthcheck.
- Restart policy.
- Backup volume/database.
- Resource limit.
- Security update.
- Rollback khi deploy lỗi.

Docker giúp đóng gói ứng dụng, nhưng không tự giải quyết toàn bộ bài toán vận hành.

## 11. Thiết kế image tốt

### 11.1. Chọn base image

Base image ảnh hưởng đến kích thước, bảo mật và khả năng debug.

| Base image         | Đặc điểm                                                                      |
| ------------------ | ----------------------------------------------------------------------------- |
| `ubuntu`, `debian` | Dễ dùng, nhiều package, kích thước lớn hơn.                                   |
| `slim`             | Nhỏ hơn bản đầy đủ, vẫn tương đối dễ dùng.                                    |
| `alpine`           | Rất nhỏ, dùng musl libc, đôi khi gặp lỗi compatibility với một số dependency. |
| `distroless`       | Rất nhỏ, ít công cụ shell/debug, phù hợp production khi đã ổn định.           |

Khi học, dùng image chính thức như `python:3.12-slim`, `node:22-slim`, `postgres:16` là lựa chọn dễ tiếp cận.

### 11.2. Tối ưu layer cache

Ví dụ không tối ưu:

```dockerfile
COPY . .
RUN pip install -r requirements.txt
```

Mỗi lần source code đổi, Docker có thể phải cài lại dependency.

Tối ưu hơn:

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

Nếu `requirements.txt` không đổi, layer cài dependency được cache lại.

### 11.3. Giảm kích thước image

Một số cách:

- Dùng base image nhỏ hợp lý.
- Dùng multi-stage build.
- Không copy file thừa nhờ `.dockerignore`.
- Xóa cache package manager sau khi cài.
- Không cài tool debug trong image production nếu không cần.
- Chỉ copy artifact cuối thay vì toàn bộ source build.

Image nhỏ giúp pull nhanh hơn, deploy nhanh hơn và giảm bề mặt tấn công.

### 11.4. Một process chính trong container

Thông thường nên chạy một process chính trong mỗi container:

- `api` container chạy web server.
- `worker` container chạy background worker.
- `db` container chạy database.
- `nginx` container chạy reverse proxy.

Không nên nhồi quá nhiều service không liên quan vào một container vì khó scale, khó restart và khó quan sát. Nếu cần nhiều service, dùng Docker Compose hoặc orchestrator.

### 11.5. Stateless container

Ứng dụng container nên càng stateless càng tốt:

- Không lưu session quan trọng trong filesystem container.
- Không lưu upload quan trọng trong writable layer.
- Không phụ thuộc vào file được sửa thủ công bên trong container.
- Log ra stdout/stderr.
- Dữ liệu bền vững nên nằm ở database, object storage hoặc volume.

Stateless giúp container dễ thay thế, scale và rollback.

## 12. Dữ liệu, network và cấu hình

### 12.1. Dữ liệu bền vững

Với database trong Docker, dữ liệu cần được lưu bằng volume.

Ví dụ PostgreSQL:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Nếu chỉ chạy:

```bash
docker run postgres:16
```

mà không có volume, dữ liệu có thể mất khi xóa container.

### 12.2. Kết nối giữa container

Trong cùng Docker network, container gọi nhau bằng tên service/container.

Ví dụ:

```text
postgresql://postgres:postgres@db:5432/app_db
```

Trong đó `db` là tên service PostgreSQL trong Compose.

Không dùng:

```text
postgresql://postgres:postgres@localhost:5432/app_db
```

từ container `api` để gọi container `db`, vì `localhost` bên trong `api` là chính container `api`.

### 12.3. Kết nối từ host vào container

Muốn host truy cập service trong container, cần map port.

Ví dụ:

```yaml
services:
  api:
    ports:
      - "8000:8000"
```

Sau đó host truy cập:

```text
http://localhost:8000
```

Nếu service chỉ cần container khác gọi, không nhất thiết phải publish port ra host.

### 12.4. Cấu hình bằng biến môi trường

Một ứng dụng nên đọc cấu hình từ biến môi trường:

```text
APP_ENV=development
DATABASE_URL=postgresql://postgres:postgres@db:5432/app_db
REDIS_URL=redis://redis:6379/0
```

Trong Compose:

```yaml
services:
  api:
    environment:
      APP_ENV: development
      DATABASE_URL: postgresql://postgres:postgres@db:5432/app_db
```

Với secret thật, nên dùng secret manager hoặc cơ chế quản lý secret phù hợp thay vì commit vào repository.

## 13. Docker và CI/CD

### 13.1. Quy trình CI/CD cơ bản

```mermaid
flowchart LR
    Code[Push Code] --> CI[CI Pipeline]
    CI --> Test[Run Tests]
    Test --> Build[Build Docker Image]
    Build --> Scan[Scan Image]
    Scan --> Push[Push to Registry]
    Push --> Deploy[Deploy to Server / Kubernetes]
```

Một pipeline phổ biến:

1. Developer push code.
2. CI cài dependency hoặc dùng container để chạy test.
3. CI build Docker image.
4. Image được scan lỗi bảo mật nếu có.
5. Image được gắn tag theo version hoặc commit SHA.
6. Image được push lên registry.
7. Server hoặc orchestrator pull image mới và deploy.

### 13.2. Tag image trong CI/CD

Nên dùng tag có thể truy vết:

```text
my-api:1.4.2
my-api:2026-06-03
my-api:git-a1b2c3d
```

Không nên chỉ deploy bằng `latest` vì khó biết version nào đang chạy. Có thể dùng `latest` cho convenience, nhưng production nên lưu tag cụ thể hoặc digest.

### 13.3. Rollback

Nếu image cũ vẫn còn trong registry, rollback có thể đơn giản là chạy lại tag trước đó.

Ví dụ:

```bash
docker run my-api:1.4.1
```

hoặc cập nhật Compose/Kubernetes về image tag cũ.

Rollback dễ hơn khi:

- Mỗi lần deploy có tag riêng.
- Migration database được thiết kế cẩn thận.
- Config và secret được quản lý độc lập.
- Log và monitoring đủ rõ để phát hiện lỗi sớm.

## 14. Bảo mật trong Docker

### 14.1. Không đưa secret vào image

Không nên viết:

```dockerfile
ENV DATABASE_PASSWORD=my-production-password
```

Secret trong image có thể bị lộ qua history, registry hoặc người có quyền pull image.

Nên truyền secret qua:

- Biến môi trường từ hệ thống deploy.
- Docker secrets.
- Kubernetes secrets.
- Secret manager của cloud.
- File secret mount lúc runtime.

### 14.2. Chạy non-root

Nếu ứng dụng không cần quyền root, nên dùng user ít quyền.

```dockerfile
RUN useradd -m appuser
USER appuser
```

Điều này giúp giảm thiệt hại nếu ứng dụng bị khai thác.

### 14.3. Giảm bề mặt tấn công

Một số nguyên tắc:

- Dùng base image chính thức và cập nhật.
- Không cài package không cần thiết.
- Dùng multi-stage build.
- Scan image để phát hiện CVE.
- Không chạy container với `--privileged` nếu không thật sự cần.
- Giới hạn capability khi cần.
- Chỉ mount file/thư mục cần thiết.
- Không mount Docker socket vào container trừ khi hiểu rõ rủi ro.

### 14.4. Docker socket

Docker socket thường nằm ở:

```text
/var/run/docker.sock
```

Nếu mount socket này vào container, container có thể điều khiển Docker daemon trên host. Đây là quyền rất mạnh và có thể tương đương quyền kiểm soát host trong nhiều tình huống.

Không nên dùng:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

trừ khi thật sự cần và đã đánh giá rủi ro.

## 15. So sánh Docker với công nghệ liên quan

### 15.1. Docker và virtual machine

| Tiêu chí   | Docker container                        | Virtual machine                              |
| ---------- | --------------------------------------- | -------------------------------------------- |
| Khởi động  | Nhanh                                   | Chậm hơn                                     |
| Kích thước | Nhỏ hơn                                 | Lớn hơn                                      |
| Kernel     | Chia sẻ kernel host                     | Có kernel riêng                              |
| Cô lập     | Tốt ở mức process                       | Mạnh hơn ở mức OS                            |
| Phù hợp    | Đóng gói ứng dụng, microservices, CI/CD | Chạy OS riêng, cô lập mạnh, lab hệ điều hành |

### 15.2. Docker và Docker Compose

| Tiêu chí | Docker                                                | Docker Compose                        |
| -------- | ----------------------------------------------------- | ------------------------------------- |
| Mục đích | Build và chạy container đơn lẻ hoặc thao tác thủ công | Định nghĩa và chạy nhiều service      |
| Cấu hình | Qua lệnh CLI                                          | Qua file YAML                         |
| Phù hợp  | Test nhanh một image/container                        | Chạy stack backend + database + cache |

Docker Compose không thay thế Docker Engine. Compose là công cụ dùng Docker Engine để tạo container, network và volume theo cấu hình.

### 15.3. Docker và Kubernetes

| Tiêu chí    | Docker/Compose                       | Kubernetes                                  |
| ----------- | ------------------------------------ | ------------------------------------------- |
| Phạm vi     | Một máy hoặc môi trường nhỏ          | Cluster nhiều node                          |
| Độ phức tạp | Dễ học hơn                           | Phức tạp hơn                                |
| Tự phục hồi | Hạn chế hơn                          | Mạnh hơn                                    |
| Scaling     | Thủ công hoặc đơn giản               | Tự động và linh hoạt hơn                    |
| Use case    | Development, staging nhỏ, server đơn | Production lớn, microservices, cloud-native |

Docker giúp hiểu container. Kubernetes giúp điều phối container ở quy mô lớn.

## 16. Các lỗi thiết kế thường gặp

### 16.1. Không dùng volume cho database

Chạy database trong container nhưng không mount volume có thể làm mất dữ liệu khi xóa container. Database gần như luôn cần volume hoặc storage bền vững.

### 16.2. Dùng `localhost` sai ngữ cảnh

Từ container `api`, `localhost` là chính container `api`, không phải container `db`. Trong Compose, nên dùng tên service như `db`, `redis`, `elasticsearch`.

### 16.3. Copy toàn bộ project vào image quá sớm

Dockerfile kiểu:

```dockerfile
COPY . .
RUN pip install -r requirements.txt
```

làm cache dependency dễ bị invalidated. Nên copy file dependency trước, cài dependency, rồi mới copy source.

### 16.4. Quên `.dockerignore`

Không có `.dockerignore` có thể làm build context rất lớn và vô tình copy secret, log, virtual environment hoặc `node_modules` vào image.

### 16.5. Dùng tag `latest` trong production

`latest` không đảm bảo là phiên bản mới nhất theo nghĩa nghiệp vụ, và cũng không giúp truy vết chính xác. Production nên dùng tag cụ thể hoặc digest.

### 16.6. Lưu dữ liệu quan trọng trong writable layer

Dữ liệu ghi trực tiếp vào filesystem container sẽ mất khi container bị xóa. Nên dùng volume, database, object storage hoặc hệ thống lưu trữ phù hợp.

### 16.7. Chạy container bằng root khi không cần

Chạy root làm tăng rủi ro bảo mật. Nếu ứng dụng không cần quyền root, nên tạo user riêng trong image và dùng `USER`.

### 16.8. Nhồi nhiều service vào một container

Một container vừa chạy API, vừa chạy database, vừa chạy worker sẽ khó scale, khó restart và khó debug. Nên tách service thành nhiều container và dùng Compose hoặc orchestrator.

### 16.9. Không giới hạn tài nguyên

Container có thể dùng quá nhiều CPU/RAM nếu không được kiểm soát. Trong production, nên đặt resource limit phù hợp ở Docker, Compose, Kubernetes hoặc nền tảng deploy.

### 16.10. Không kiểm tra image size và CVE

Image quá lớn làm deploy chậm và có thể chứa nhiều package lỗi thời. Nên tối ưu image và scan bảo mật định kỳ.

## 17. Bài tập thực hành

### Bài 1: Chạy container đầu tiên

Thực hiện:

- Chạy `hello-world`.
- Chạy `nginx` ở port `8080`.
- Mở trình duyệt truy cập `http://localhost:8080`.
- Xem log container.
- Dừng và xóa container.

### Bài 2: Chạy PostgreSQL bằng Docker

Chạy PostgreSQL với:

- User `postgres`.
- Password `postgres`.
- Database `school_db`.
- Volume `postgres_data`.
- Port `5432`.

Sau đó xóa container, tạo lại container cùng volume và kiểm tra dữ liệu còn hay không.

### Bài 3: Viết Dockerfile cho FastAPI

Tạo ứng dụng FastAPI đơn giản có endpoint `/health`, viết Dockerfile, build image và chạy container ở port `8000`.

Yêu cầu:

- Dùng `python:3.12-slim`.
- Cài dependency từ `requirements.txt`.
- App lắng nghe `0.0.0.0`.
- Có `.dockerignore`.

### Bài 4: Docker Compose cho backend và database

Viết `compose.yaml` gồm:

- Service `api` build từ Dockerfile.
- Service `db` dùng image PostgreSQL.
- Volume lưu dữ liệu PostgreSQL.
- Biến môi trường `DATABASE_URL`.
- Healthcheck cho database.

### Bài 5: Multi-stage build

Chọn một ứng dụng build cần compiler hoặc bundler, ví dụ Go, React hoặc Java. Viết Dockerfile multi-stage để image runtime cuối chỉ chứa artifact cần chạy.

### Bài 6: Debug network

Tạo network `app_network`, chạy Redis trong network đó, sau đó chạy một container khác để ping hoặc dùng `redis-cli` kết nối Redis bằng hostname `redis`.

## 18. Lộ trình học đề xuất

1. Nắm khái niệm image, container, Dockerfile, registry.
2. Thực hành `docker run`, `docker ps`, `docker logs`, `docker exec`, `docker stop`, `docker rm`.
3. Học port mapping, environment variable, volume và network.
4. Viết Dockerfile cho ứng dụng đơn giản.
5. Tối ưu Dockerfile bằng layer cache và `.dockerignore`.
6. Học Docker Compose để chạy backend, database và cache.
7. Thực hành healthcheck, logs và debug container.
8. Học multi-stage build và non-root user.
9. Tìm hiểu registry, image tag và CI/CD.
10. Học thêm Kubernetes sau khi đã vững Docker và Compose.

## 19. Kết luận

Docker là công cụ nền tảng trong phát triển và triển khai phần mềm hiện đại. Điểm quan trọng nhất không chỉ là biết chạy vài lệnh `docker run`, mà là hiểu cách image được build, container được cô lập, dữ liệu được lưu bằng volume, service giao tiếp qua network và nhiều container được điều phối bằng Docker Compose.

Khi dùng Docker đúng cách, dự án dễ chạy lại hơn, môi trường nhất quán hơn và quy trình triển khai rõ ràng hơn. Tuy nhiên, Docker không tự động làm ứng dụng an toàn, nhẹ, nhanh hay dễ vận hành. Cần thiết kế Dockerfile, volume, network, secret, logging và image tag một cách có chủ đích.

## 20. Tài liệu tham khảo

- Docker Documentation: https://docs.docker.com/
- Docker Engine: https://docs.docker.com/engine/
- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Docker Compose: https://docs.docker.com/compose/
- Docker Build: https://docs.docker.com/build/
- Docker Hub: https://hub.docker.com/
- Open Container Initiative: https://opencontainers.org/

## Popular commands in docker

### IMAGE

```bash
docker images
docker pull <image>
docker build -t <name> .
docker rmi <image>
```

### CONTAINER

```bash
docker ps
docker ps -a

docker run <image>

docker stop <container>
docker start <container>
docker restart <container>

docker rm <container>
```

### DEBUG

```bash
docker logs -f <container>
docker exec -it <container> bash
```

### COMPOSE

```bash
docker compose up -d
docker compose up -d --build

docker compose ps

docker compose logs -f

docker compose exec <service> bash

docker compose down
```
