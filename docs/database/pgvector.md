# pgvector: Cơ sở lý thuyết, kiến trúc và thực hành

## 1. Mục tiêu tài liệu

Tài liệu này trình bày pgvector theo hướng lý thuyết kết hợp thực hành, giúp người học nắm được:

- pgvector là gì và vì sao nó quan trọng khi muốn thêm khả năng vector search vào PostgreSQL.
- Cách lưu vector embedding trực tiếp trong bảng PostgreSQL cùng dữ liệu nghiệp vụ.
- Các kiểu dữ liệu cốt lõi của pgvector như `vector`, `halfvec`, `bit` và `sparsevec`.
- Các toán tử khoảng cách như L2, inner product, cosine distance, L1, Hamming và Jaccard.
- Cách tạo bảng, insert vector, truy vấn nearest neighbor và kết hợp với metadata filtering.
- Cách dùng index HNSW và IVFFlat để tăng tốc tìm kiếm vector.
- Cách dùng pgvector trong các hệ thống semantic search, recommendation và Retrieval-Augmented Generation.
- Cách kết nối pgvector với Python ở mức cơ bản.
- Các lỗi thiết kế thường gặp khi dùng pgvector trong dự án thực tế.

Tài liệu này phù hợp cho người đã biết PostgreSQL cơ bản và muốn dùng PostgreSQL làm nơi lưu trữ embedding cho ứng dụng AI. Một số tính năng của pgvector thay đổi theo phiên bản, vì vậy khi triển khai production nên đối chiếu thêm với tài liệu chính thức của pgvector và phiên bản PostgreSQL đang sử dụng.

## 2. Tổng quan về pgvector

pgvector là một extension mã nguồn mở cho PostgreSQL, cho phép lưu trữ và tìm kiếm vector embedding trực tiếp trong database quan hệ. Thay vì phải tách dữ liệu nghiệp vụ vào PostgreSQL và vector embedding sang một vector database riêng, pgvector cho phép đặt cả hai trong cùng một hệ quản trị dữ liệu.

Trong các ứng dụng AI hiện đại, dữ liệu như văn bản, hình ảnh, âm thanh hoặc sản phẩm thường được chuyển thành vector embedding bằng mô hình machine learning. Các vector này biểu diễn ý nghĩa hoặc đặc trưng của dữ liệu gốc. pgvector bổ sung kiểu dữ liệu vector và các toán tử khoảng cách vào PostgreSQL để có thể truy vấn các bản ghi gần nghĩa nhất với vector truy vấn.

pgvector thường được dùng cho:

- Semantic search trong tài liệu, bài viết, sản phẩm hoặc câu hỏi.
- Retrieval-Augmented Generation trong chatbot và ứng dụng LLM.
- Recommendation system dựa trên độ tương đồng giữa người dùng, sản phẩm hoặc nội dung.
- Lưu embedding cạnh dữ liệu nghiệp vụ như `users`, `products`, `documents`, `orders`.
- Hybrid search kết hợp full-text search của PostgreSQL với vector search.
- Prototype nhanh hệ thống AI mà không muốn vận hành thêm một vector database riêng.

### 2.1. Đặc điểm nổi bật

| Đặc điểm | Ý nghĩa |
| --- | --- |
| PostgreSQL extension | pgvector chạy bên trong PostgreSQL, dùng chung SQL, transaction, backup, replication và quyền truy cập. |
| Vector search | Hỗ trợ tìm kiếm nearest neighbor theo khoảng cách giữa các vector embedding. |
| Exact và approximate search | Có thể quét chính xác toàn bảng hoặc dùng index ANN để tăng tốc trên dữ liệu lớn. |
| Nhiều kiểu vector | Hỗ trợ `vector`, `halfvec`, `bit` và `sparsevec` cho nhiều nhu cầu lưu trữ khác nhau. |
| HNSW và IVFFlat | Hỗ trợ hai loại index phổ biến cho approximate nearest neighbor. |
| Metadata filtering bằng SQL | Có thể dùng `WHERE`, `JOIN`, `GROUP BY`, full-text search và các index PostgreSQL thông thường. |
| Dễ tích hợp | Dùng được với bất kỳ ngôn ngữ nào có PostgreSQL client, ví dụ Python, Node.js, Java, Go hoặc Rust. |
| Phù hợp RAG nhỏ và vừa | Rất tiện khi ứng dụng đã dùng PostgreSQL và khối lượng vector chưa cần một hệ thống chuyên biệt. |

## 3. Cơ sở lý thuyết

### 3.1. Vector embedding

Vector embedding là cách biểu diễn dữ liệu thành một dãy số thực. Mỗi vector thường có nhiều chiều, ví dụ 384, 768, 1024, 1536 hoặc 3072 chiều tùy mô hình embedding.

Ví dụ một câu văn:

```text
"pgvector dùng PostgreSQL để lưu embedding"
```

có thể được mô hình embedding chuyển thành vector:

```text
[0.12, -0.34, 0.08, ..., 0.91]
```

Vector này không lưu trực tiếp từng từ, mà biểu diễn ý nghĩa tổng quát của câu. Hai câu có ý nghĩa gần nhau thường có vector gần nhau trong không gian vector.

### 3.2. Vector search trong PostgreSQL

PostgreSQL truyền thống mạnh ở dữ liệu có cấu trúc:

```sql
SELECT *
FROM products
WHERE category = 'laptop'
  AND price BETWEEN 10000000 AND 20000000;
```

Truy vấn này tìm dữ liệu dựa trên điều kiện chính xác. Nhưng nếu người dùng tìm:

```text
"máy tính xách tay nhẹ cho sinh viên"
```

thì dữ liệu phù hợp có thể chứa mô tả:

```text
"laptop mỏng nhẹ, pin lâu, phù hợp học tập"
```

Hai câu không trùng từ khóa hoàn toàn, nhưng gần nhau về ý nghĩa. pgvector xử lý bài toán này bằng cách:

1. Chuyển mô tả sản phẩm thành embedding và lưu vào cột `embedding`.
2. Chuyển câu hỏi của người dùng thành query embedding.
3. Sắp xếp bản ghi theo khoảng cách giữa `embedding` và query embedding.
4. Trả về top-k kết quả gần nhất.

### 3.3. Similarity search

Similarity search là quá trình tìm các vector gần nhất với vector truy vấn. Trong pgvector, truy vấn nearest neighbor thường có dạng:

```sql
SELECT id, title
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Trong ví dụ này:

- `embedding` là cột vector đã lưu trong bảng.
- `'[0.11,0.21,0.29]'::vector` là vector truy vấn.
- `<=>` là toán tử cosine distance.
- `LIMIT 5` nghĩa là lấy 5 kết quả gần nhất.

Với vector search, kết quả thường là danh sách được xếp hạng theo độ gần, không phải danh sách khớp chính xác tuyệt đối.

### 3.4. Distance metric và operator

Distance metric là công thức đo khoảng cách hoặc độ tương đồng giữa hai vector. pgvector biểu diễn metric thông qua các toán tử SQL.

| Operator | Ý nghĩa | Trường hợp sử dụng |
| --- | --- | --- |
| `<->` | L2 distance, còn gọi là Euclidean distance. | Dữ liệu vector cần khoảng cách hình học. |
| `<#>` | Negative inner product. | Mô hình embedding được huấn luyện cho inner product hoặc vector đã normalize. |
| `<=>` | Cosine distance. | Semantic search với text embedding. |
| `<+>` | L1 distance, còn gọi là taxicab distance. | Một số bài toán đặc thù cần tổng sai khác tuyệt đối. |
| `<~>` | Hamming distance cho binary vector. | Tìm kiếm bit vector, image hash hoặc dữ liệu nhị phân. |
| `<%>` | Jaccard distance cho binary vector. | So sánh tập hợp hoặc bit vector theo mức giao nhau. |

Điểm cần nhớ:

- pgvector trả về **distance**, nên giá trị nhỏ hơn thường nghĩa là gần hơn.
- Với cosine similarity, có thể tính `1 - cosine_distance`.
- Toán tử `<#>` trả về negative inner product để PostgreSQL có thể dùng index scan theo thứ tự tăng dần.

Ví dụ tính cosine similarity:

```sql
SELECT
    id,
    title,
    1 - (embedding <=> '[0.11,0.21,0.29]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

### 3.5. Exact search và Approximate Nearest Neighbor

Mặc định, nếu chưa tạo vector index, PostgreSQL có thể tìm nearest neighbor bằng cách so sánh query vector với toàn bộ các vector trong bảng. Đây là exact search, có độ chính xác đầy đủ nhưng có thể chậm khi số lượng bản ghi lớn.

Khi có nhiều vector, pgvector hỗ trợ approximate nearest neighbor thông qua index:

- `hnsw`
- `ivfflat`

ANN đánh đổi một phần độ chính xác để lấy tốc độ. Trong semantic search và RAG, cách này thường đủ tốt nếu được benchmark và tinh chỉnh bằng dữ liệu thật.

### 3.6. HNSW

HNSW là viết tắt của Hierarchical Navigable Small World. Đây là index dạng graph nhiều tầng cho nearest neighbor search.

Ý tưởng chính:

- Mỗi vector là một node trong graph.
- Các vector gần nhau được nối với nhau.
- Khi tìm kiếm, thuật toán đi qua graph để đến vùng có vector gần query nhất.
- HNSW thường có tốc độ truy vấn tốt và recall cao, nhưng tốn bộ nhớ và thời gian build index hơn IVFFlat.

Trong pgvector, HNSW có thể được tạo ngay cả khi bảng chưa có dữ liệu vì không cần bước training.

### 3.7. IVFFlat

IVFFlat chia không gian vector thành nhiều cụm, thường gọi là list. Khi truy vấn, hệ thống tìm một số list gần query nhất rồi chỉ quét trong các list đó.

Đặc điểm chính:

- Build nhanh hơn HNSW.
- Dùng ít bộ nhớ hơn HNSW.
- Cần có dữ liệu trước khi tạo index để quá trình clustering có ý nghĩa.
- Recall phụ thuộc nhiều vào số `lists` khi tạo index và `probes` khi truy vấn.

IVFFlat phù hợp khi muốn cân bằng giữa tốc độ build index, bộ nhớ và chất lượng truy vấn.

### 3.8. Hybrid search

Hybrid search là cách kết hợp nhiều tín hiệu tìm kiếm, thường là:

- Full-text search để bắt từ khóa chính xác.
- Vector search để bắt ý nghĩa.
- Metadata filtering để lọc quyền truy cập, loại tài liệu, thời gian hoặc domain.

PostgreSQL có sẵn full-text search, B-tree index, GIN index, JOIN và transaction. Đây là lợi thế lớn của pgvector: vector search có thể sống cùng dữ liệu quan hệ và các khả năng SQL truyền thống.

## 4. Kiến trúc pgvector trong PostgreSQL

### 4.1. Sơ đồ kiến trúc Mermaid

```mermaid
flowchart TD
    App[Application: API, Chatbot, Search Service] -->|SQL Query| PG[PostgreSQL Server]
    PG --> Parser[Parser / Planner / Executor]
    Parser --> Table[Table: documents, products, chunks]
    Table --> Scalar[Scalar Columns: title, content, tenant_id, category]
    Table --> Vector[Vector Column: embedding]
    Scalar --> BTree[B-tree / GIN / Full-text Index]
    Vector --> HNSW[HNSW Index]
    Vector --> IVFFlat[IVFFlat Index]
    Parser --> Join[JOIN, WHERE, ORDER BY, LIMIT]
    BTree --> Result[Ranked Rows]
    HNSW --> Result
    IVFFlat --> Result
    Join --> Result
```

Sơ đồ trên cho thấy pgvector không phải là một database tách biệt. Nó bổ sung kiểu dữ liệu, operator và index vào PostgreSQL. Ứng dụng vẫn gửi SQL đến PostgreSQL, còn PostgreSQL planner quyết định cách thực thi truy vấn.

### 4.2. Các thành phần quan trọng

| Thành phần | Vai trò |
| --- | --- |
| PostgreSQL Server | Hệ quản trị database chính, xử lý SQL, transaction, WAL, backup và quyền truy cập. |
| Extension `vector` | Extension do pgvector cung cấp, được bật bằng `CREATE EXTENSION vector`. |
| Vector column | Cột lưu embedding, ví dụ `embedding vector(1536)`. |
| Scalar columns | Các cột metadata như `title`, `source`, `tenant_id`, `created_at`, `category`. |
| Distance operators | Các toán tử như `<->`, `<#>`, `<=>`, `<+>`, `<~>`, `<%>`. |
| Operator class | Cấu hình metric cho index, ví dụ `vector_cosine_ops` hoặc `vector_l2_ops`. |
| HNSW index | Index ANN dạng graph, thường cho recall và tốc độ truy vấn tốt. |
| IVFFlat index | Index ANN dạng phân cụm, build nhanh hơn nhưng cần có dữ liệu trước. |
| PostgreSQL index thường | B-tree, GIN, BRIN hoặc full-text index dùng cho filtering và hybrid search. |

## 5. Cài đặt và bật pgvector

### 5.1. Chạy PostgreSQL có pgvector bằng Docker

Cách nhanh nhất để học là dùng image PostgreSQL đã cài sẵn pgvector:

```bash
docker run --name pgvector-demo \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=rag_db \
  -p 5432:5432 \
  -d pgvector/pgvector:pg18
```

Trong đó:

- `POSTGRES_USER` là user mặc định.
- `POSTGRES_PASSWORD` là mật khẩu.
- `POSTGRES_DB` là database được tạo sẵn.
- `5432` là port mặc định của PostgreSQL.
- `pgvector/pgvector:pg18` là image PostgreSQL có pgvector cài sẵn. Có thể thay `pg18` bằng tag tương ứng với version PostgreSQL bạn dùng.

### 5.2. Kết nối bằng psql

Nếu đã cài `psql`, có thể kết nối:

```bash
psql -h localhost -p 5432 -U postgres -d rag_db
```

Bật extension trong database hiện tại:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Kiểm tra version extension:

```sql
SELECT extversion
FROM pg_extension
WHERE extname = 'vector';
```

Lưu ý: tên extension là `vector`, không phải `pgvector`.

### 5.3. Cài từ package hoặc source

Trong production, cách cài phụ thuộc hệ điều hành và cách cài PostgreSQL:

| Cách cài | Ghi chú |
| --- | --- |
| Docker | Dễ nhất để học và chạy demo. |
| APT/Yum | Phù hợp server Linux dùng package PostgreSQL chính thức. |
| Homebrew | Phù hợp macOS. |
| Source build | Cần `make`, `pg_config` và PostgreSQL development headers. |
| Managed PostgreSQL | Một số nhà cung cấp cloud đã hỗ trợ pgvector sẵn. |

Sau khi cài extension ở cấp server, vẫn cần chạy `CREATE EXTENSION vector;` trong từng database muốn sử dụng.

## 6. Các kiểu dữ liệu cốt lõi

### 6.1. `vector`

`vector` là kiểu dữ liệu chính để lưu dense vector bằng số thực single precision.

Tạo bảng có vector 3 chiều:

```sql
CREATE TABLE items (
    id BIGSERIAL PRIMARY KEY,
    embedding vector(3)
);
```

Insert dữ liệu:

```sql
INSERT INTO items (embedding)
VALUES
    ('[1,2,3]'),
    ('[4,5,6]');
```

`vector(n)` yêu cầu mọi vector trong cột có đúng `n` chiều. Đây là cách nên dùng nếu một bảng chỉ lưu embedding từ một model cố định.

### 6.2. `halfvec`

`halfvec` lưu vector ở half precision để giảm dung lượng lưu trữ và kích thước index.

```sql
CREATE TABLE items_half (
    id BIGSERIAL PRIMARY KEY,
    embedding halfvec(3)
);
```

`halfvec` hữu ích khi:

- Dataset lớn và RAM là giới hạn chính.
- Chấp nhận giảm một phần độ chính xác số học.
- Muốn index nhỏ hơn để tăng khả năng nằm trong bộ nhớ.

### 6.3. `bit`

`bit` dùng cho binary vector.

```sql
CREATE TABLE image_hashes (
    id BIGSERIAL PRIMARY KEY,
    embedding bit(64)
);
```

Binary vector thường dùng với:

- Image hash.
- Binary quantization.
- Một số pipeline cần biểu diễn cực gọn.

Truy vấn Hamming distance:

```sql
SELECT *
FROM image_hashes
ORDER BY embedding <~> '101010'
LIMIT 5;
```

### 6.4. `sparsevec`

`sparsevec` dùng cho sparse vector, tức vector có rất nhiều phần tử bằng 0 và chỉ lưu các phần tử khác 0.

```sql
CREATE TABLE sparse_items (
    id BIGSERIAL PRIMARY KEY,
    embedding sparsevec(1000)
);
```

Insert sparse vector:

```sql
INSERT INTO sparse_items (embedding)
VALUES ('{1:0.5,20:1.2,800:0.7}/1000');
```

Format `{index:value}/dimensions` nghĩa là:

- Vector có tổng cộng `1000` chiều.
- Chỉ các chiều `1`, `20` và `800` có giá trị khác 0.
- Chỉ số bắt đầu từ 1, giống SQL array.

### 6.5. Chọn kiểu dữ liệu

| Kiểu | Phù hợp với | Ghi chú |
| --- | --- | --- |
| `vector(n)` | Dense embedding phổ biến. | Lựa chọn mặc định cho text embedding. |
| `halfvec(n)` | Dataset lớn, cần giảm RAM và index size. | Có thể giảm nhẹ độ chính xác. |
| `bit(n)` | Binary vector hoặc quantization. | Dùng với Hamming/Jaccard. |
| `sparsevec(n)` | Sparse embedding hoặc sparse retrieval. | Chỉ lưu phần tử khác 0. |

Với người mới học RAG, thường nên bắt đầu bằng `vector(n)`.

## 7. Thao tác SQL cơ bản

### 7.1. Tạo bảng tài liệu

Ví dụ bảng `documents` lưu nội dung chunk và embedding:

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    document_id TEXT NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    source TEXT,
    category TEXT,
    tenant_id BIGINT,
    embedding vector(3) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

Trong dự án thực tế, `embedding vector(3)` thường là `embedding vector(384)`, `vector(768)`, `vector(1536)` hoặc số chiều tương ứng với embedding model.

### 7.2. Thêm dữ liệu

```sql
INSERT INTO documents (
    document_id,
    title,
    content,
    source,
    category,
    tenant_id,
    embedding
)
VALUES
(
    'doc_001',
    'Giới thiệu PostgreSQL',
    'PostgreSQL là hệ quản trị cơ sở dữ liệu quan hệ mã nguồn mở.',
    'postgresql.md',
    'database',
    1,
    '[0.10,0.20,0.30]'
),
(
    'doc_002',
    'Giới thiệu pgvector',
    'pgvector cho phép lưu và tìm kiếm vector embedding trong PostgreSQL.',
    'pgvector.md',
    'database',
    1,
    '[0.12,0.22,0.31]'
);
```

### 7.3. Truy vấn nearest neighbor

Tìm 5 tài liệu gần nhất theo cosine distance:

```sql
SELECT
    id,
    title,
    content,
    embedding <=> '[0.11,0.21,0.29]'::vector AS distance
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Vì pgvector sắp xếp theo distance tăng dần, kết quả ở đầu danh sách là kết quả gần nhất.

### 7.4. Tính similarity score

Nếu muốn hiển thị cosine similarity thay vì cosine distance:

```sql
SELECT
    id,
    title,
    1 - (embedding <=> '[0.11,0.21,0.29]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Lưu ý: để index vector được dùng, phần `ORDER BY` nên là distance operator trực tiếp và có `LIMIT`.

### 7.5. Tìm kiếm kèm filter

Tìm trong một category:

```sql
SELECT
    id,
    title,
    category,
    embedding <=> '[0.11,0.21,0.29]'::vector AS distance
FROM documents
WHERE category = 'database'
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Tìm theo tenant để tránh lộ dữ liệu giữa người dùng:

```sql
SELECT
    id,
    title,
    tenant_id,
    embedding <=> '[0.11,0.21,0.29]'::vector AS distance
FROM documents
WHERE tenant_id = 1
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

### 7.6. Update, delete và upsert vector

Cập nhật vector:

```sql
UPDATE documents
SET embedding = '[0.13,0.20,0.28]'
WHERE id = 1;
```

Xóa dữ liệu:

```sql
DELETE FROM documents
WHERE id = 1;
```

Upsert theo khóa tự nhiên:

```sql
ALTER TABLE documents
ADD CONSTRAINT documents_document_id_title_key UNIQUE (document_id, title);

INSERT INTO documents (
    document_id,
    title,
    content,
    category,
    tenant_id,
    embedding
)
VALUES (
    'doc_001',
    'Giới thiệu PostgreSQL',
    'Nội dung mới đã được cập nhật.',
    'database',
    1,
    '[0.14,0.18,0.27]'
)
ON CONFLICT (document_id, title)
DO UPDATE SET
    content = EXCLUDED.content,
    category = EXCLUDED.category,
    tenant_id = EXCLUDED.tenant_id,
    embedding = EXCLUDED.embedding;
```

## 8. Index và tối ưu truy vấn

### 8.1. Khi nào cần index vector

Khi bảng nhỏ, exact search có thể đủ nhanh:

```sql
SELECT *
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Khi bảng lớn hơn, ví dụ hàng trăm nghìn hoặc hàng triệu vector, nên tạo index để giảm latency. Tuy nhiên, vector index thường là approximate index, nên cần kiểm tra trade-off giữa tốc độ và recall.

### 8.2. HNSW index

Tạo HNSW index cho cosine distance:

```sql
CREATE INDEX documents_embedding_hnsw_cosine_idx
ON documents
USING hnsw (embedding vector_cosine_ops);
```

Một số operator class phổ biến:

| Metric | Operator class |
| --- | --- |
| L2 distance | `vector_l2_ops` |
| Inner product | `vector_ip_ops` |
| Cosine distance | `vector_cosine_ops` |
| L1 distance | `vector_l1_ops` |

Tạo HNSW index với tham số:

```sql
CREATE INDEX documents_embedding_hnsw_cosine_idx
ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

Trong đó:

- `m` là số kết nối tối đa mỗi layer.
- `ef_construction` là kích thước danh sách ứng viên khi build index.
- Giá trị cao hơn thường tăng recall nhưng tốn thời gian build và insert hơn.

Tăng recall khi query:

```sql
SET hnsw.ef_search = 100;
```

Nếu chỉ muốn áp dụng cho một truy vấn:

```sql
BEGIN;
SET LOCAL hnsw.ef_search = 100;

SELECT id, title
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;

COMMIT;
```

### 8.3. IVFFlat index

Tạo IVFFlat index cho cosine distance:

```sql
CREATE INDEX documents_embedding_ivfflat_cosine_idx
ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

IVFFlat nên được tạo sau khi bảng đã có một lượng dữ liệu đủ đại diện. Nếu tạo quá sớm khi bảng còn ít dữ liệu, chất lượng clustering có thể kém.

Tăng số list được quét khi query:

```sql
SET ivfflat.probes = 10;
```

Nếu chỉ muốn áp dụng cho một truy vấn:

```sql
BEGIN;
SET LOCAL ivfflat.probes = 10;

SELECT id, title
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;

COMMIT;
```

Gợi ý ban đầu:

- Với dưới 1 triệu rows, có thể bắt đầu `lists` khoảng `rows / 1000`.
- Với trên 1 triệu rows, có thể bắt đầu `lists` khoảng căn bậc hai của số rows.
- `probes` có thể bắt đầu khoảng căn bậc hai của `lists`, sau đó benchmark thêm.

### 8.4. HNSW và IVFFlat khác nhau thế nào

| Tiêu chí | HNSW | IVFFlat |
| --- | --- | --- |
| Cấu trúc | Graph nhiều tầng. | Chia vector thành các list/cụm. |
| Cần dữ liệu trước khi tạo index | Không bắt buộc. | Nên có dữ liệu trước. |
| Tốc độ query | Thường tốt hơn ở cùng mức recall. | Thường thấp hơn HNSW ở cùng mức recall. |
| Tốc độ build | Chậm hơn. | Nhanh hơn. |
| Bộ nhớ | Thường tốn hơn. | Thường tiết kiệm hơn. |
| Tham số query | `hnsw.ef_search`. | `ivfflat.probes`. |
| Phù hợp | Recall cao, query nhanh, dataset vừa với tài nguyên. | Build nhanh, ít RAM hơn, chấp nhận tuning `lists` và `probes`. |

Không có index tốt nhất cho mọi trường hợp. Cần benchmark trên dữ liệu và truy vấn thật.

### 8.5. Index cho metadata filtering

Vector index không thay thế index thông thường. Nếu query thường lọc theo `tenant_id`, `category`, `created_at`, nên tạo thêm index PostgreSQL:

```sql
CREATE INDEX documents_tenant_id_idx
ON documents (tenant_id);

CREATE INDEX documents_category_idx
ON documents (category);

CREATE INDEX documents_tenant_category_idx
ON documents (tenant_id, category);
```

Với full-text search:

```sql
ALTER TABLE documents
ADD COLUMN search_vector tsvector
GENERATED ALWAYS AS (
    to_tsvector('simple', coalesce(title, '') || ' ' || coalesce(content, ''))
) STORED;

CREATE INDEX documents_search_vector_idx
ON documents
USING gin (search_vector);
```

### 8.6. Partial index và partition

Nếu mỗi truy vấn chỉ tìm trong một nhóm nhỏ cố định, có thể dùng partial index:

```sql
CREATE INDEX documents_database_embedding_hnsw_idx
ON documents
USING hnsw (embedding vector_cosine_ops)
WHERE category = 'database';
```

Với multi-tenancy lớn, có thể cân nhắc partition:

```sql
CREATE TABLE tenant_documents (
    id BIGSERIAL,
    tenant_id BIGINT NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    embedding vector(3) NOT NULL,
    PRIMARY KEY (tenant_id, id)
) PARTITION BY LIST (tenant_id);
```

Partition giúp cô lập dữ liệu và giảm số rows cần quét trong một số workload, nhưng cũng làm schema và vận hành phức tạp hơn.

### 8.7. EXPLAIN và kiểm tra query plan

Dùng `EXPLAIN` để xem PostgreSQL có dùng index không:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title
FROM documents
WHERE tenant_id = 1
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Một truy vấn vector thường dễ dùng index hơn khi:

- Có `ORDER BY embedding <operator> query_vector`.
- Có `LIMIT`.
- Operator trong query khớp với operator class của index.
- Không bọc distance operator trong biểu thức khiến planner không nhận ra đường index phù hợp.

Ví dụ dễ dùng index:

```sql
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Ví dụ có thể không dùng vector index:

```sql
ORDER BY 1 - (embedding <=> '[0.11,0.21,0.29]'::vector) DESC
LIMIT 5;
```

## 9. Ví dụ sử dụng pgvector bằng Python

### 9.1. Cài thư viện

Có thể dùng `psycopg` để kết nối PostgreSQL và `pgvector` để đăng ký adapter cho kiểu vector:

```bash
pip install "psycopg[binary]" pgvector
```

### 9.2. Kết nối và bật extension

```python
import psycopg
from pgvector.psycopg import register_vector


conn = psycopg.connect(
    "postgresql://postgres:postgres@localhost:5432/rag_db"
)
conn.execute("CREATE EXTENSION IF NOT EXISTS vector")
register_vector(conn)
conn.commit()
```

### 9.3. Tạo bảng và thêm dữ liệu

```python
conn.execute("""
    CREATE TABLE IF NOT EXISTS documents (
        id BIGSERIAL PRIMARY KEY,
        document_id TEXT NOT NULL,
        title TEXT NOT NULL,
        content TEXT NOT NULL,
        category TEXT,
        tenant_id BIGINT,
        embedding vector(3) NOT NULL,
        created_at TIMESTAMPTZ DEFAULT now()
    )
""")

documents = [
    (
        "doc_001",
        "Giới thiệu PostgreSQL",
        "PostgreSQL là cơ sở dữ liệu quan hệ mã nguồn mở.",
        "database",
        1,
        [0.10, 0.20, 0.30],
    ),
    (
        "doc_002",
        "Giới thiệu pgvector",
        "pgvector thêm vector search vào PostgreSQL.",
        "database",
        1,
        [0.12, 0.22, 0.31],
    ),
    (
        "doc_003",
        "Cache với Redis",
        "Redis thường được dùng để cache dữ liệu truy cập nhiều.",
        "cache",
        1,
        [0.80, 0.10, 0.20],
    ),
]

with conn.cursor() as cur:
    cur.executemany(
        """
        INSERT INTO documents (
            document_id,
            title,
            content,
            category,
            tenant_id,
            embedding
        )
        VALUES (%s, %s, %s, %s, %s, %s)
        """,
        documents,
    )
conn.commit()
```

Trong ví dụ thực tế, danh sách `[0.10, 0.20, 0.30]` sẽ đến từ embedding model, không phải nhập tay.

### 9.4. Tìm kiếm vector

```python
query_embedding = [0.11, 0.21, 0.29]

rows = conn.execute(
    """
    SELECT
        id,
        title,
        content,
        embedding <=> %s AS distance
    FROM documents
    ORDER BY embedding <=> %s
    LIMIT 5
    """,
    (query_embedding, query_embedding),
).fetchall()

for row in rows:
    print(row)
```

Kết quả trả về các dòng gần query embedding nhất theo cosine distance.

### 9.5. Tìm kiếm kèm filter

```python
query_embedding = [0.11, 0.21, 0.29]
tenant_id = 1
category = "database"

rows = conn.execute(
    """
    SELECT
        id,
        title,
        content,
        1 - (embedding <=> %s) AS similarity
    FROM documents
    WHERE tenant_id = %s
      AND category = %s
    ORDER BY embedding <=> %s
    LIMIT 5
    """,
    (query_embedding, tenant_id, category, query_embedding),
).fetchall()

for row in rows:
    print(row)
```

Filter theo `tenant_id` là bước quan trọng trong ứng dụng nhiều người dùng. Không nên chỉ dựa vào vector search rồi mới lọc dữ liệu ở tầng ứng dụng.

### 9.6. Tạo index bằng Python

```python
conn.execute("""
    CREATE INDEX IF NOT EXISTS documents_embedding_hnsw_cosine_idx
    ON documents
    USING hnsw (embedding vector_cosine_ops)
""")
conn.commit()
```

Với production, nên cân nhắc `CREATE INDEX CONCURRENTLY` để giảm blocking khi bảng đang được ghi:

```python
with psycopg.connect(
    "postgresql://postgres:postgres@localhost:5432/rag_db",
    autocommit=True,
) as conn:
    conn.execute("""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS documents_embedding_hnsw_cosine_idx
        ON documents
        USING hnsw (embedding vector_cosine_ops)
    """)
```

`CREATE INDEX CONCURRENTLY` không chạy bên trong transaction block thông thường, nên ví dụ trên bật `autocommit=True`.

## 10. pgvector trong hệ thống RAG

### 10.1. Sơ đồ RAG với pgvector

```mermaid
flowchart TD
    Docs[Documents] --> Chunking[Chunking]
    Chunking --> Embedding[Embedding Model]
    Embedding --> PG[(PostgreSQL + pgvector)]
    Docs --> Metadata[Metadata: source, tenant, category]
    Metadata --> PG

    User[User Question] --> QueryEmbedding[Query Embedding]
    QueryEmbedding --> PG
    PG --> Context[Relevant Chunks]
    Context --> Prompt[Prompt]
    Prompt --> LLM[Large Language Model]
    User --> LLM
    LLM --> Answer[Answer]
```

Trong RAG, pgvector đóng vai trò là lớp truy xuất tri thức nằm trong PostgreSQL. Ứng dụng tìm các chunk liên quan bằng vector search, sau đó đưa nội dung chunk vào prompt để LLM tạo câu trả lời dựa trên dữ liệu của hệ thống.

### 10.2. Quy trình RAG cơ bản

1. Thu thập tài liệu như PDF, Markdown, HTML hoặc dữ liệu từ database.
2. Chia tài liệu thành các chunk nhỏ.
3. Tạo embedding cho từng chunk.
4. Lưu `content`, `source`, `document_id`, `tenant_id`, metadata và `embedding` vào PostgreSQL.
5. Khi người dùng đặt câu hỏi, tạo embedding cho câu hỏi.
6. Truy vấn pgvector để lấy top-k chunk gần nhất, có filter phân quyền.
7. Đưa các chunk vào prompt làm context.
8. LLM tạo câu trả lời dựa trên context.

### 10.3. Schema mẫu cho RAG

```sql
CREATE TABLE rag_documents (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    document_id TEXT NOT NULL,
    chunk_index INTEGER NOT NULL,
    title TEXT,
    content TEXT NOT NULL,
    source TEXT,
    metadata JSONB DEFAULT '{}'::jsonb,
    embedding vector(1536) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    UNIQUE (tenant_id, document_id, chunk_index)
);

CREATE INDEX rag_documents_tenant_id_idx
ON rag_documents (tenant_id);

CREATE INDEX rag_documents_metadata_idx
ON rag_documents
USING gin (metadata);

CREATE INDEX rag_documents_embedding_hnsw_idx
ON rag_documents
USING hnsw (embedding vector_cosine_ops);
```

Truy vấn retrieval:

```sql
SELECT
    id,
    document_id,
    chunk_index,
    title,
    content,
    source,
    1 - (embedding <=> $1::vector) AS similarity
FROM rag_documents
WHERE tenant_id = $2
ORDER BY embedding <=> $1::vector
LIMIT 8;
```

### 10.4. Vì sao cần chunking

Tài liệu dài thường không nên được embedding thành một vector duy nhất, vì vector đó dễ làm mất chi tiết. Chunking giúp:

- Giữ thông tin cụ thể hơn.
- Tăng khả năng tìm đúng đoạn liên quan.
- Kiểm soát độ dài context đưa vào LLM.
- Giảm nhiễu khi tài liệu chứa nhiều chủ đề.

Kích thước chunk thường phụ thuộc vào loại tài liệu và embedding model. Với văn bản, có thể bắt đầu từ 300 đến 800 token mỗi chunk, sau đó điều chỉnh dựa trên chất lượng retrieval.

### 10.5. Hybrid search trong RAG

Một hệ thống RAG tốt không nhất thiết chỉ dùng vector search. Có thể kết hợp full-text search và vector search.

Ví dụ full-text search:

```sql
SELECT id, title, content
FROM rag_documents, plainto_tsquery('simple', 'postgres vector') query
WHERE search_vector @@ query
ORDER BY ts_rank_cd(search_vector, query) DESC
LIMIT 10;
```

Ví dụ vector search:

```sql
SELECT id, title, content
FROM rag_documents
WHERE tenant_id = 1
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 10;
```

Sau đó ứng dụng có thể hợp nhất hai danh sách bằng Reciprocal Rank Fusion, reranker hoặc cross-encoder. pgvector thuận tiện ở điểm cả full-text search, metadata và vector đều nằm trong PostgreSQL.

## 11. So sánh pgvector với vector database chuyên dụng

| Tiêu chí | pgvector | Qdrant/Milvus hoặc vector database chuyên dụng |
| --- | --- | --- |
| Bản chất | Extension trong PostgreSQL. | Hệ thống chuyên biệt cho vector search. |
| Dữ liệu nghiệp vụ | Lưu chung trong PostgreSQL rất thuận tiện. | Thường cần đồng bộ với database nghiệp vụ khác. |
| SQL và JOIN | Dùng đầy đủ SQL, JOIN, transaction. | Tùy hệ thống, thường không mạnh bằng PostgreSQL về relational query. |
| Vận hành | Đơn giản nếu hệ thống đã dùng PostgreSQL. | Cần vận hành thêm service hoặc cluster riêng. |
| Quy mô rất lớn | Có thể cần tuning, partition, replica hoặc sharding. | Thường thiết kế tốt hơn cho khối lượng vector rất lớn. |
| Metadata filtering | Dùng index và kiểu dữ liệu PostgreSQL. | Có cơ chế filtering riêng. |
| RAG nhỏ và vừa | Rất phù hợp. | Cũng phù hợp nhưng có thêm chi phí vận hành. |
| Workload chuyên vector | Có thể không tối ưu bằng hệ thống chuyên dụng ở quy mô lớn. | Thường mạnh hơn về distributed vector search. |

pgvector không thay thế hoàn toàn Qdrant hoặc Milvus. Nó phù hợp nhất khi:

- Ứng dụng đã dùng PostgreSQL.
- Muốn đơn giản hóa kiến trúc.
- Dữ liệu vector và dữ liệu nghiệp vụ cần transaction hoặc JOIN.
- Quy mô dữ liệu nằm trong khả năng vận hành của PostgreSQL.

Với dataset rất lớn, QPS cao, yêu cầu distributed vector search phức tạp hoặc nhiều loại workload vector chuyên sâu, nên benchmark pgvector với các vector database chuyên dụng trước khi quyết định.

## 12. Thiết kế dữ liệu trong pgvector

### 12.1. Chọn bảng và cột

Có hai cách thiết kế phổ biến.

Cách 1: Lưu embedding trực tiếp trong bảng nghiệp vụ:

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    price NUMERIC(12, 2),
    category TEXT,
    embedding vector(1536)
);
```

Cách này đơn giản khi mỗi sản phẩm chỉ có một embedding.

Cách 2: Tách bảng embedding riêng:

```sql
CREATE TABLE product_embeddings (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    model_name TEXT NOT NULL,
    embedding vector(1536) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (product_id, model_name)
);
```

Cách này phù hợp khi:

- Một bản ghi có nhiều embedding từ nhiều model.
- Cần re-embed dữ liệu theo version model.
- Muốn quản lý vòng đời embedding riêng với dữ liệu nghiệp vụ.

### 12.2. Chọn dimension

Dimension phải khớp với embedding model. Nếu model trả về 1536 chiều, cột nên là:

```sql
embedding vector(1536)
```

Không nên insert vector sai số chiều. Nếu đổi embedding model, cần có chiến lược migrate:

- Tạo cột mới, ví dụ `embedding_v2 vector(3072)`.
- Tạo bảng embedding riêng theo `model_name`.
- Re-embed dữ liệu theo batch.
- Tạo index mới và chuyển traffic dần sang embedding mới.

### 12.3. Thiết kế metadata

Metadata nên dùng kiểu dữ liệu rõ ràng nếu trường đó thường xuyên lọc hoặc join.

| Trường | Kiểu gợi ý | Vai trò |
| --- | --- | --- |
| `tenant_id` | `BIGINT` hoặc `UUID` | Phân quyền và multi-tenancy. |
| `document_id` | `TEXT`, `BIGINT` hoặc `UUID` | Liên kết chunk với tài liệu gốc. |
| `chunk_index` | `INTEGER` | Giữ thứ tự chunk trong tài liệu. |
| `category` | `TEXT` hoặc bảng riêng | Lọc theo loại nội dung. |
| `created_at` | `TIMESTAMPTZ` | Lọc theo thời gian. |
| `metadata` | `JSONB` | Lưu thông tin linh hoạt ít dùng để join. |

Không nên nhét mọi thứ vào `JSONB` nếu thường xuyên lọc bằng các trường đó. PostgreSQL xử lý cột typed tốt hơn cho ràng buộc, index, join và thống kê.

### 12.4. Chọn metric

Việc chọn metric nên dựa trên embedding model:

- Nếu tài liệu model khuyến nghị cosine, dùng `<=>` và `vector_cosine_ops`.
- Nếu model được huấn luyện cho dot product, dùng `<#>` và `vector_ip_ops`.
- Nếu vector đã normalize và model phù hợp inner product, inner product có thể nhanh hơn trong một số trường hợp.
- Nếu dữ liệu là tọa độ hoặc đặc trưng hình học, cân nhắc L2 với `<->`.

Điều quan trọng là query operator phải khớp với index operator class. Nếu tạo index `vector_cosine_ops` nhưng truy vấn bằng `<->`, PostgreSQL sẽ không dùng index đó cho L2.

### 12.5. Chiến lược multi-tenancy

Với ứng dụng nhiều tenant, truy vấn luôn nên filter theo tenant:

```sql
SELECT id, title, content
FROM rag_documents
WHERE tenant_id = 42
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 8;
```

Các lựa chọn thiết kế:

| Cách | Khi nên dùng |
| --- | --- |
| Một bảng chung có `tenant_id` | Đơn giản, phù hợp đa số ứng dụng nhỏ và vừa. |
| Partial index theo tenant lớn | Khi một vài tenant có dữ liệu lớn hoặc workload riêng. |
| Partition theo tenant | Khi cần cô lập dữ liệu mạnh hơn hoặc pruning tốt hơn. |
| Database/schema riêng | Khi yêu cầu cô lập vận hành hoặc compliance cao. |

Không nên truy vấn vector toàn bảng rồi lọc tenant ở ứng dụng, vì vừa chậm vừa có rủi ro lộ dữ liệu.

## 13. Tối ưu và vận hành

### 13.1. Bulk load

Khi nạp nhiều embedding ban đầu:

- Insert theo batch thay vì từng dòng.
- Dùng `COPY` nếu dataset lớn.
- Tạo vector index sau khi load dữ liệu ban đầu.
- Chạy `ANALYZE` sau khi load để cập nhật thống kê.

Ví dụ:

```sql
ANALYZE documents;
```

### 13.2. Tạo index trong production

Với bảng đang phục vụ traffic, nên cân nhắc:

```sql
CREATE INDEX CONCURRENTLY documents_embedding_hnsw_cosine_idx
ON documents
USING hnsw (embedding vector_cosine_ops);
```

`CREATE INDEX CONCURRENTLY` giảm blocking write, nhưng chạy lâu hơn và không được đặt trong transaction block thông thường.

### 13.3. Memory và cấu hình PostgreSQL

Vector search có thể tốn nhiều RAM hơn truy vấn SQL thông thường. Một số cấu hình cần quan tâm:

| Cấu hình | Ý nghĩa |
| --- | --- |
| `shared_buffers` | Bộ nhớ cache dữ liệu của PostgreSQL. |
| `work_mem` | Bộ nhớ cho sort/hash trong từng operation. |
| `maintenance_work_mem` | Bộ nhớ dùng cho thao tác maintenance như tạo index. |
| `max_parallel_maintenance_workers` | Số worker song song khi tạo index. |
| `max_parallel_workers_per_gather` | Tăng khả năng parallel scan khi exact search. |

Không nên tăng memory theo cảm tính. Cần đo bằng dữ liệu thật, theo dõi RAM server và kiểm tra query plan.

### 13.4. Monitoring

Nên theo dõi:

- Latency của truy vấn vector.
- Recall của approximate search so với exact search.
- Kích thước bảng và index.
- Số lượng dead tuples.
- Tần suất autovacuum.
- Query chậm qua `pg_stat_statements`.

Kiểm tra kích thước index:

```sql
SELECT pg_size_pretty(pg_relation_size('documents_embedding_hnsw_cosine_idx'));
```

So sánh approximate search với exact search trong một transaction:

```sql
BEGIN;
SET LOCAL enable_indexscan = off;

SELECT id, title
FROM documents
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;

COMMIT;
```

### 13.5. Vacuum và reindex

Vì pgvector vẫn chạy trong PostgreSQL, các vấn đề vận hành PostgreSQL vẫn quan trọng:

- Bảng có nhiều update/delete cần autovacuum tốt.
- HNSW index có thể cần thời gian vacuum lâu hơn.
- Có thể cân nhắc `REINDEX INDEX CONCURRENTLY` trước khi vacuum trong một số trường hợp index lớn.

Ví dụ:

```sql
REINDEX INDEX CONCURRENTLY documents_embedding_hnsw_cosine_idx;
VACUUM documents;
```

## 14. Các lỗi thiết kế thường gặp

### 14.1. Quên bật extension

Nếu chưa chạy:

```sql
CREATE EXTENSION vector;
```

PostgreSQL sẽ không nhận kiểu dữ liệu `vector`. Cần bật extension trong đúng database đang dùng.

### 14.2. Vector dimension không khớp

Nếu cột là `vector(1536)` nhưng ứng dụng insert vector 768 chiều, truy vấn sẽ lỗi. Lỗi này thường xảy ra khi đổi embedding model nhưng không migrate schema.

### 14.3. Dùng sai metric

Embedding model khác nhau có thể khuyến nghị metric khác nhau. Nếu dùng sai metric, kết quả retrieval có thể kém dù database hoạt động đúng.

### 14.4. Tạo index không khớp query

Nếu tạo index:

```sql
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops);
```

nhưng query:

```sql
ORDER BY embedding <-> '[0.11,0.21,0.29]'::vector
```

thì index cosine không phục vụ L2 distance. Query operator và operator class phải cùng metric.

### 14.5. Viết ORDER BY làm mất khả năng dùng index

Nên viết:

```sql
ORDER BY embedding <=> '[0.11,0.21,0.29]'::vector
LIMIT 5;
```

Tránh dùng biểu thức similarity trực tiếp trong `ORDER BY` nếu muốn tận dụng vector index:

```sql
ORDER BY 1 - (embedding <=> '[0.11,0.21,0.29]'::vector) DESC
LIMIT 5;
```

Có thể tính similarity trong `SELECT`, nhưng vẫn order bằng distance operator.

### 14.6. Tạo IVFFlat khi bảng còn quá ít dữ liệu

IVFFlat cần dữ liệu để chia cụm. Nếu tạo index khi bảng còn gần như rỗng, recall có thể kém. Nên load dữ liệu ban đầu trước rồi mới tạo IVFFlat.

### 14.7. Không dùng filter phân quyền

Trong ứng dụng nhiều người dùng, nếu không filter theo `tenant_id`, `owner_id` hoặc rule phân quyền, hệ thống có thể trả về chunk của người khác.

### 14.8. Chunk quá dài hoặc quá ngắn

Chunk quá dài làm mất chi tiết, chunk quá ngắn thiếu ngữ cảnh. Cần kiểm thử kích thước chunk, overlap và top-k trên dữ liệu thật.

### 14.9. Lưu metadata thiếu thông tin

Nếu chỉ lưu vector mà không lưu `content`, `source`, `document_id`, `chunk_index`, ứng dụng sẽ khó hiển thị kết quả, tạo prompt hoặc truy ngược về tài liệu gốc.

### 14.10. Không benchmark recall

Vector index approximate có thể trả kết quả khác exact search. Cần đánh giá bằng bộ câu hỏi mẫu, kiểm tra top-k, recall, latency và chất lượng câu trả lời cuối cùng của RAG.

### 14.11. Xem pgvector như giải pháp vô hạn quy mô

pgvector rất tiện và mạnh, nhưng vẫn nằm trong giới hạn vận hành của PostgreSQL. Với dữ liệu vector cực lớn, QPS rất cao hoặc yêu cầu distributed ANN chuyên sâu, cần benchmark với vector database chuyên dụng.

## 15. Bài tập thực hành

### Bài 1: Tạo bảng semantic search

Tạo bảng `articles` gồm:

- `id`
- `title`
- `content`
- `category`
- `embedding vector(3)`

Thêm ít nhất 5 dòng dữ liệu mẫu và viết truy vấn tìm 3 bài gần nhất với một query vector.

### Bài 2: Tìm kiếm có filter

Từ bảng `articles`, viết truy vấn:

- Chỉ tìm trong `category = 'database'`.
- Sắp xếp theo cosine distance.
- Trả về `title`, `content` và similarity score.

### Bài 3: Tạo HNSW index

Tạo HNSW index cho cột `embedding` dùng cosine distance. Sau đó dùng:

```sql
EXPLAIN (ANALYZE, BUFFERS)
```

để xem query plan.

### Bài 4: Thiết kế bảng RAG

Thiết kế bảng `rag_chunks` có các trường:

- `tenant_id`
- `document_id`
- `chunk_index`
- `content`
- `metadata`
- `embedding`

Thêm unique constraint phù hợp để tránh insert trùng chunk.

### Bài 5: So sánh exact và approximate search

Chạy cùng một query:

- Một lần dùng index bình thường.
- Một lần tắt index scan bằng `SET LOCAL enable_indexscan = off`.

So sánh kết quả top-k và latency.

## 16. Lộ trình học đề xuất

1. Ôn lại PostgreSQL cơ bản: table, index, transaction, `EXPLAIN`, backup và role.
2. Hiểu vector embedding và các metric như cosine, L2, inner product.
3. Tạo bảng có `embedding vector(n)` và thực hành nearest neighbor search.
4. Thêm metadata filtering bằng `WHERE`, B-tree index và GIN index.
5. Tạo HNSW index, kiểm tra query plan và tinh chỉnh `hnsw.ef_search`.
6. Thử IVFFlat, so sánh `lists`, `probes`, recall và latency.
7. Xây dựng RAG pipeline nhỏ bằng Python.
8. Benchmark bằng dữ liệu thật trước khi quyết định thiết kế production.

## 17. Kết luận

pgvector là lựa chọn rất thực dụng khi muốn đưa vector search vào hệ sinh thái PostgreSQL. Nó cho phép lưu embedding cạnh dữ liệu nghiệp vụ, dùng SQL quen thuộc, tận dụng transaction, backup, replication, role, JOIN và full-text search của PostgreSQL.

Về mặt kỹ thuật, pgvector bổ sung kiểu dữ liệu vector, các toán tử khoảng cách và index HNSW/IVFFlat để hỗ trợ exact và approximate nearest neighbor search. Trong các ứng dụng AI như RAG, semantic search và recommendation, pgvector thường là lựa chọn tốt khi muốn giảm độ phức tạp hệ thống.

Khi thiết kế với pgvector, cần quan tâm đến embedding model, dimension, metric, index operator class, metadata filtering, multi-tenancy, chunking và benchmark recall. Chất lượng hệ thống phụ thuộc nhiều vào dữ liệu và pipeline embedding, không chỉ vào việc tạo một cột `vector`.

## 18. Tài liệu tham khảo

- pgvector GitHub Repository: https://github.com/pgvector/pgvector
- pgvector README: https://github.com/pgvector/pgvector/blob/master/README.md
- pgvector Python: https://github.com/pgvector/pgvector-python
- PostgreSQL Documentation: https://www.postgresql.org/docs/
- PostgreSQL Full Text Search: https://www.postgresql.org/docs/current/textsearch.html
- PostgreSQL CREATE EXTENSION: https://www.postgresql.org/docs/current/sql-createextension.html
- HNSW Paper: https://arxiv.org/abs/1603.09320
