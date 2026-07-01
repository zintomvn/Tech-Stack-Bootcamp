# File Storage: Cơ sở lý thuyết và thiết kế hệ thống lưu trữ file

## 1. Mục tiêu tài liệu

Tài liệu này trình bày lý thuyết nền tảng về **file storage** trong hệ thống phần mềm, đặc biệt là các ứng dụng web, backend API, hệ thống dữ liệu và hệ thống AI/RAG.

Sau khi đọc tài liệu, người học cần nắm được:

- File storage là gì và vì sao không nên lưu mọi loại dữ liệu vào database.
- Sự khác nhau giữa file storage, object storage, block storage, database và CDN.
- Các thành phần cốt lõi của một hệ thống lưu trữ file: file, path/key, metadata, quyền truy cập, checksum, version và lifecycle.
- Luồng xử lý phổ biến khi upload, lưu trữ, tải xuống và xóa file.
- Cách thiết kế quan hệ giữa database và file storage.
- Các vấn đề quan trọng về bảo mật, hiệu năng, chi phí, backup và vận hành.
- Những lỗi thiết kế thường gặp khi xây dựng chức năng upload/download file.

## 2. File storage là gì?

**File storage** là cách lưu trữ dữ liệu dưới dạng file, ví dụ ảnh, video, PDF, Word, Excel, file nén, log, dataset, model, audio hoặc tài liệu người dùng upload. File thường được truy cập bằng tên, đường dẫn hoặc key logic.

Trong một ứng dụng thực tế, file storage thường dùng để lưu:

- Ảnh đại diện, ảnh sản phẩm, ảnh bài viết.
- Tài liệu PDF, hợp đồng, hóa đơn, chứng chỉ.
- Video, audio, file media lớn.
- File import/export như CSV, Excel, JSON, parquet.
- File backup, log, report định kỳ.
- Tài liệu gốc cho hệ thống tìm kiếm, AI hoặc RAG.
- Model checkpoint, embedding file, artifact của pipeline machine learning.

Điểm quan trọng: file storage không thay thế database. File storage mạnh khi lưu dữ liệu dạng byte lớn, còn database mạnh khi lưu dữ liệu có cấu trúc, cần query, transaction, join, index hoặc kiểm soát quan hệ nghiệp vụ.

Ví dụ thiết kế hợp lý:

```text
Database:
- file_id
- owner_id
- bucket/container
- object_key/path
- original_filename
- content_type
- size
- checksum
- status
- created_at

File storage:
- Nội dung thật của file dưới dạng bytes
```

## 3. Vì sao cần file storage riêng?

Nhiều người mới học backend thường đặt câu hỏi: "Có thể lưu file trực tiếp vào database không?" Câu trả lời là có thể, nhưng không phải lúc nào cũng nên.

Database có thể lưu binary data bằng kiểu như `BLOB`, `BYTEA` hoặc `VARBINARY`. Cách này phù hợp với file nhỏ, số lượng ít, cần transaction chặt với dữ liệu nghiệp vụ. Tuy nhiên, với hệ thống có nhiều file hoặc file lớn, lưu trực tiếp trong database thường gây ra các vấn đề:

- Database phình to nhanh, backup và restore chậm.
- Query nghiệp vụ bị ảnh hưởng bởi dữ liệu binary lớn.
- Chi phí lưu trữ database thường cao hơn storage chuyên dụng.
- Khó phục vụ download/upload file lớn hiệu quả.
- Khó dùng CDN, signed URL, lifecycle policy hoặc tier lưu trữ lạnh.
- Tăng tải cho database connection pool.

Vì vậy, kiến trúc phổ biến là:

```text
Ứng dụng lưu metadata trong database.
Nội dung file được lưu trong file storage hoặc object storage.
Database chỉ giữ đường dẫn/key để tìm lại file.
```

## 4. Phân biệt các loại lưu trữ

### 4.1. File storage

File storage tổ chức dữ liệu theo mô hình file và thư mục. Người dùng hoặc ứng dụng truy cập file bằng path như:

```text
/uploads/users/42/avatar.png
/data/reports/2026/06/sales.xlsx
```

File storage truyền thống thường có các khái niệm quen thuộc:

- Folder/directory.
- File name.
- File path.
- Permission.
- Owner/group.
- Read/write/delete.
- Rename/move.

Ví dụ: ổ đĩa local của server, NAS, NFS, SMB, Google Cloud Filestore, Amazon EFS, Azure Files.

### 4.2. Object storage

Object storage lưu dữ liệu thành các object trong bucket/container. Mỗi object thường gồm:

- Nội dung file.
- Object key/name.
- Metadata.
- Version hoặc generation.

Object storage không thật sự có thư mục vật lý theo kiểu filesystem. Ký tự `/` trong object key chỉ là một phần của tên, giúp giao diện hiển thị giống folder.

Ví dụ object key:

```text
users/42/avatar.png
reports/2026/06/sales.xlsx
rag/documents/doc_123/source.pdf
```

Các dịch vụ object storage phổ biến:

- Google Cloud Storage.
- Amazon S3.
- Azure Blob Storage.
- MinIO.

Object storage thường phù hợp hơn file storage truyền thống khi cần lưu lượng lớn, độ bền cao, khả năng mở rộng tốt, truy cập qua HTTP API và tích hợp CDN.

### 4.3. Block storage

Block storage cung cấp ổ đĩa cấp thấp cho máy chủ hoặc máy ảo. Hệ điều hành sẽ format filesystem lên block storage rồi dùng như ổ cứng.

Ví dụ:

- Persistent Disk trên Google Cloud.
- Amazon EBS.
- Azure Managed Disks.

Block storage phù hợp cho:

- Ổ đĩa hệ điều hành.
- Database cần I/O thấp độ trễ.
- Ứng dụng cần filesystem local gắn với một máy chủ.

Block storage không phải lựa chọn tốt để chia sẻ file trực tiếp cho nhiều ứng dụng phân tán nếu không có tầng filesystem hoặc dịch vụ quản lý phù hợp.

### 4.4. Database

Database lưu dữ liệu có cấu trúc và hỗ trợ query. Database phù hợp với:

- User, order, product, payment, permission.
- Metadata file.
- Trạng thái xử lý.
- Quan hệ giữa file và entity nghiệp vụ.
- Audit log, nếu log có cấu trúc cần truy vấn.

Database không nên bị biến thành nơi chứa toàn bộ file lớn nếu hệ thống cần mở rộng.

### 4.5. CDN

CDN không phải nơi lưu trữ chính. CDN là lớp cache phân phối nội dung gần người dùng hơn để tăng tốc tải file và giảm tải origin storage.

Luồng thường gặp:

```text
User -> CDN -> File/Object Storage
```

CDN phù hợp cho file public hoặc file có thể cache:

- Ảnh sản phẩm.
- File tĩnh của website.
- Video/audio public.
- Tài liệu public.

Với file private, vẫn có thể dùng CDN nhưng cần cơ chế ký URL, cookie bảo mật hoặc kiểm soát truy cập cẩn thận.

## 5. Thành phần cốt lõi của hệ thống file storage

### 5.1. File data

File data là nội dung thật của file, được lưu dưới dạng bytes. Storage thường không cần hiểu bên trong file là gì. Một file PDF, ảnh PNG hoặc file CSV đều chỉ là chuỗi bytes ở tầng storage.

Ứng dụng mới là nơi quyết định:

- File có được phép upload không.
- File thuộc người dùng nào.
- File dùng cho mục đích gì.
- File có cần xử lý thêm không.
- File được phép public hay private.

### 5.2. File name

File name là tên gốc hoặc tên hiển thị của file. Ví dụ:

```text
avatar.png
invoice-june.pdf
sales-report.xlsx
```

Không nên dùng trực tiếp file name do người dùng nhập làm tên lưu trữ duy nhất, vì:

- Dễ trùng tên.
- Có thể chứa ký tự nguy hiểm hoặc không tương thích.
- Có thể tiết lộ thông tin nhạy cảm.
- Khó phân quyền và audit.
- Có thể bị lợi dụng trong path traversal nếu xử lý sai.

Cách tốt hơn là lưu `original_filename` trong database để hiển thị, còn tên lưu trong storage nên được tạo an toàn bằng UUID, ULID, hash hoặc cấu trúc key có kiểm soát.

### 5.3. Path, key và prefix

Trong file storage truyền thống, path thể hiện vị trí file trong cây thư mục:

```text
/uploads/users/42/avatar.png
```

Trong object storage, object key đóng vai trò tương tự path logic:

```text
users/42/uploads/2026/06/30/01J2ABCDEF-avatar.png
```

Một key tốt nên:

- Tránh trùng lặp.
- Dễ phân nhóm theo user, tenant, ngày tháng hoặc loại dữ liệu.
- Hỗ trợ lifecycle rule và batch processing.
- Không chứa dữ liệu nhạy cảm không cần thiết.
- Không phụ thuộc hoàn toàn vào tên file người dùng cung cấp.

Ví dụ key tốt:

```text
tenants/acme/users/u_123/uploads/2026/06/30/file_01J2ABC.pdf
products/p_456/images/img_01J2DEF.webp
rag/documents/doc_789/original/source.pdf
exports/orders/2026/06/30/export_01J2GHI.csv
```

Ví dụ key kém:

```text
file.pdf
test.png
data.csv
upload.xlsx
```

### 5.4. Metadata

Metadata là thông tin mô tả file. Metadata có hai nhóm:

| Nhóm metadata | Ví dụ |
| --- | --- |
| Metadata kỹ thuật | size, content type, checksum, extension, created time |
| Metadata nghiệp vụ | owner_id, document_id, invoice_id, status, access level |

Một số metadata phổ biến:

| Metadata | Ý nghĩa |
| --- | --- |
| `Content-Type` | Kiểu nội dung, ví dụ `image/png`, `application/pdf`, `text/csv`. |
| `Content-Length` | Kích thước file. |
| `Content-Disposition` | Gợi ý browser hiển thị inline hay tải xuống. |
| `Cache-Control` | Chính sách cache cho browser/CDN. |
| `ETag` | Định danh phiên bản hoặc trạng thái nội dung do storage cung cấp. |
| `Checksum` | Giá trị kiểm tra toàn vẹn dữ liệu, ví dụ MD5, SHA-256 hoặc CRC32C. |

Nguyên tắc thực tế:

```text
Metadata cần query, lọc, phân quyền, audit -> lưu trong database.
Metadata phục vụ HTTP/cache/storage behavior -> lưu ở storage object metadata.
```

### 5.5. MIME type và extension

Extension là phần mở rộng của tên file, ví dụ `.png`, `.pdf`, `.csv`. MIME type là kiểu nội dung chuẩn trong HTTP, ví dụ:

```text
image/png
application/pdf
text/csv
application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

Không nên tin tuyệt đối vào extension do người dùng gửi. Một file tên `avatar.png` vẫn có thể chứa nội dung không phải ảnh. Khi bảo mật quan trọng, cần kiểm tra MIME type, magic bytes hoặc dùng thư viện phân tích file.

### 5.6. Checksum

Checksum dùng để kiểm tra tính toàn vẹn của file. Nếu checksum trước và sau khi upload khác nhau, file có thể đã bị lỗi trong quá trình truyền hoặc ghi.

Các checksum hay gặp:

- MD5.
- SHA-256.
- CRC32C.

Checksum có thể dùng để:

- Phát hiện file hỏng.
- Tránh upload trùng lặp.
- Kiểm tra dữ liệu sau khi migrate.
- Xác nhận file tải xuống đúng nội dung.

Lưu ý: checksum không thay thế phân quyền hoặc kiểm tra virus. Checksum chỉ chứng minh dữ liệu có thay đổi hay không so với một giá trị tham chiếu.

### 5.7. Permission và ownership

Mỗi file thường cần xác định:

- Ai là chủ sở hữu.
- Ai có quyền đọc.
- Ai có quyền ghi/cập nhật.
- Ai có quyền xóa.
- File là public hay private.
- Quyền truy cập có hết hạn không.

Trong ứng dụng backend, quyền truy cập thường không nên phụ thuộc hoàn toàn vào storage. Backend nên kiểm tra quyền nghiệp vụ trước khi cấp link tải hoặc cho phép thao tác file.

Ví dụ:

```text
User yêu cầu download contract.pdf
Backend kiểm tra user có thuộc contract đó không
Nếu hợp lệ, backend trả file hoặc tạo signed URL ngắn hạn
```

### 5.8. Versioning

Versioning là khả năng lưu nhiều phiên bản của cùng một file hoặc object. Khi file bị ghi đè, storage vẫn giữ bản cũ.

Versioning hữu ích khi:

- Cần khôi phục file bị ghi nhầm.
- Cần audit lịch sử tài liệu.
- Cần rollback artifact hoặc model.
- Cần chống xóa nhầm.

Tuy nhiên, versioning làm tăng chi phí vì các bản cũ vẫn chiếm dung lượng. Cần kết hợp lifecycle rule để dọn phiên bản cũ sau một thời gian hợp lý.

### 5.9. Lifecycle

Lifecycle là chính sách tự động quản lý vòng đời file:

- Xóa file tạm sau vài giờ hoặc vài ngày.
- Chuyển file ít truy cập sang storage class rẻ hơn.
- Xóa bản version cũ sau một khoảng thời gian.
- Giữ log hoặc backup theo chính sách retention.

Ví dụ:

```text
File upload tạm: xóa sau 24 giờ nếu không được xác nhận.
File report: giữ 1 năm rồi chuyển sang archive.
File log: giữ 90 ngày rồi xóa.
File backup: giữ 30 ngày bản hằng ngày, 12 tháng bản hằng tháng.
```

## 6. Luồng upload file cơ bản

Một luồng upload file điển hình:

```text
1. Client chọn file.
2. Client gửi request upload đến backend hoặc upload trực tiếp lên storage bằng signed URL.
3. Backend xác thực người dùng.
4. Backend kiểm tra quyền upload.
5. Backend kiểm tra loại file, kích thước, tên file và metadata.
6. File được ghi vào storage.
7. Backend lưu metadata vào database.
8. Backend trả về file_id hoặc URL truy cập.
```

Có hai mô hình upload phổ biến.

### 6.1. Upload qua backend

```text
Client -> Backend -> File Storage
```

Ưu điểm:

- Backend kiểm soát toàn bộ luồng.
- Dễ validate, scan, log và phân quyền.
- Phù hợp với file nhỏ hoặc hệ thống đơn giản.

Nhược điểm:

- Backend chịu tải băng thông lớn.
- Upload file lớn có thể làm nghẽn worker.
- Cần cấu hình timeout, streaming và giới hạn dung lượng cẩn thận.

### 6.2. Upload trực tiếp lên storage bằng signed URL

```text
Client -> Backend: xin quyền upload
Backend -> Client: trả signed URL
Client -> Storage: upload trực tiếp
Storage/Event -> Backend: xác nhận hoặc xử lý tiếp
```

Ưu điểm:

- Giảm tải cho backend.
- Phù hợp với file lớn.
- Tận dụng hạ tầng storage/CDN tốt hơn.

Nhược điểm:

- Luồng phức tạp hơn.
- Cần kiểm soát content type, size, thời hạn URL và trạng thái upload.
- Cần cơ chế xác nhận upload thành công và dọn file upload dở.

## 7. Luồng download file

Có ba cách download thường gặp.

### 7.1. Backend stream file

```text
Client -> Backend -> Storage -> Backend -> Client
```

Backend đọc file từ storage rồi stream về client.

Phù hợp khi:

- File private.
- Cần kiểm tra quyền rất chặt.
- Cần ghi audit log chính xác.
- File không quá lớn hoặc traffic không quá cao.

Hạn chế:

- Backend tốn băng thông.
- Cần xử lý range request nếu file lớn/video.
- Khó scale hơn so với tải trực tiếp từ storage/CDN.

### 7.2. Redirect hoặc signed URL

```text
Client -> Backend: xin download
Backend kiểm tra quyền
Backend -> Client: trả signed URL ngắn hạn
Client -> Storage/CDN: tải file
```

Phù hợp khi:

- File private nhưng cần tải hiệu quả.
- Muốn backend kiểm tra quyền trước, sau đó để storage phục vụ nội dung.
- File lớn hoặc traffic cao.

Signed URL nên có thời hạn ngắn và chỉ cấp sau khi backend đã kiểm tra quyền nghiệp vụ.

### 7.3. Public URL hoặc CDN URL

```text
Client -> CDN/Storage public URL
```

Phù hợp với file public:

- Logo.
- Ảnh sản phẩm public.
- File tĩnh website.
- Tài liệu public.

Không nên dùng public URL cho file người dùng, hợp đồng, hóa đơn, dữ liệu cá nhân hoặc file nội bộ.

## 8. Quan hệ giữa database và file storage

Một mẫu thiết kế phổ biến là bảng `files` trong database:

```sql
CREATE TABLE files (
    id UUID PRIMARY KEY,
    owner_id UUID NOT NULL,
    storage_provider TEXT NOT NULL,
    bucket TEXT NOT NULL,
    object_key TEXT NOT NULL,
    original_filename TEXT NOT NULL,
    content_type TEXT,
    size_bytes BIGINT,
    checksum TEXT,
    visibility TEXT NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

Trong đó:

| Cột | Ý nghĩa |
| --- | --- |
| `id` | ID nội bộ dùng trong ứng dụng. |
| `owner_id` | Chủ sở hữu file. |
| `storage_provider` | Local, GCS, S3, Azure Blob, MinIO. |
| `bucket` | Bucket/container nơi lưu file. |
| `object_key` | Tên object hoặc path thật trong storage. |
| `original_filename` | Tên file gốc để hiển thị cho người dùng. |
| `content_type` | MIME type. |
| `size_bytes` | Kích thước file. |
| `checksum` | Giá trị kiểm tra toàn vẹn. |
| `visibility` | Public, private, internal. |
| `status` | Pending, uploaded, processing, ready, failed, deleted. |

### 8.1. Vấn đề transaction

Database transaction và file storage write thường không nằm trong cùng một transaction ACID. Điều này tạo ra tình huống lệch trạng thái:

- File đã upload thành công nhưng metadata chưa ghi vào database.
- Metadata đã tạo nhưng file upload thất bại.
- User xóa record trong database nhưng file chưa được xóa khỏi storage.

Cách xử lý thực tế:

- Dùng trạng thái `pending`, `ready`, `failed`, `deleted`.
- Dùng background job để dọn orphan file.
- Dùng idempotency key để tránh tạo trùng.
- Ghi log và retry khi xóa file thất bại.
- Không coi việc xóa file vật lý là điều kiện duy nhất để hoàn tất request xóa nghiệp vụ.

Ví dụ luồng upload an toàn:

```text
1. Tạo record files với status = pending.
2. Upload file vào storage với object_key đã định.
3. Kiểm tra upload thành công, size và checksum.
4. Cập nhật record thành status = ready.
5. Job định kỳ xóa các file pending quá hạn.
```

## 9. Thiết kế tên file và object key

Một object key tốt nên cân bằng giữa khả năng đọc hiểu và an toàn.

### 9.1. Theo user hoặc tenant

```text
tenants/{tenant_id}/users/{user_id}/uploads/{yyyy}/{mm}/{dd}/{file_id}.pdf
```

Ưu điểm:

- Dễ phân vùng dữ liệu theo tenant.
- Dễ audit và dọn dẹp.
- Dễ áp dụng policy theo prefix.

### 9.2. Theo loại dữ liệu

```text
avatars/{user_id}/{file_id}.webp
products/{product_id}/images/{image_id}.jpg
invoices/{yyyy}/{mm}/{invoice_id}.pdf
```

Ưu điểm:

- Dễ hiểu theo nghiệp vụ.
- Dễ viết lifecycle rule riêng cho từng nhóm.

### 9.3. Theo hash nội dung

```text
sha256/ab/cd/abcdef123456...
```

Ưu điểm:

- Tránh lưu trùng nội dung.
- Hữu ích cho cache, artifact, package, dataset.

Nhược điểm:

- Khó đọc với người vận hành.
- Cần bảng metadata để ánh xạ hash với file nghiệp vụ.

### 9.4. Nguyên tắc đặt tên

Nên:

- Dùng ID ổn định như UUID, ULID hoặc snowflake ID.
- Giữ extension nếu cần phục vụ download hoặc preview.
- Chuẩn hóa tên file hiển thị riêng với key lưu trữ.
- Tránh ký tự điều khiển, path traversal như `../`, hoặc ký tự khó xử lý.
- Không đưa email, số điện thoại, số căn cước hoặc thông tin nhạy cảm vào key nếu không cần.

Không nên:

- Dùng nguyên tên file người dùng làm key duy nhất.
- Ghi đè file cũ mà không có version hoặc precondition.
- Đặt toàn bộ file vào một folder/prefix nếu hệ thống có lượng file rất lớn.
- Trộn file tạm, file chính thức và file backup trong cùng một prefix không có quy tắc.

## 10. Bảo mật trong file storage

File upload là một bề mặt tấn công phổ biến. Thiết kế file storage cần xem bảo mật là phần lõi, không phải phần thêm sau.

### 10.1. Kiểm soát loại file

Không nên chỉ kiểm tra extension. Nên kiểm tra:

- Extension cho phép.
- MIME type.
- Magic bytes nếu cần.
- Kích thước tối đa.
- Số lượng file tối đa trong một request.
- Kích thước ảnh/video thực tế nếu file là media.

Ví dụ:

```text
Cho phép avatar:
- image/jpeg
- image/png
- image/webp
- max 5 MB
- kích thước ảnh tối đa 4096 x 4096
```

### 10.2. Chống path traversal

Path traversal xảy ra khi attacker lợi dụng tên file để truy cập đường dẫn ngoài thư mục cho phép:

```text
../../etc/passwd
..\..\windows\system32\config
```

Không nên nối trực tiếp tên file người dùng vào đường dẫn filesystem. Nếu dùng local storage, cần normalize path và đảm bảo path cuối cùng vẫn nằm trong thư mục storage được phép.

### 10.3. Public access

Không nên public toàn bộ bucket/container chỉ để hiển thị vài file public. Một lỗi public bucket có thể làm lộ toàn bộ tài liệu người dùng.

Nguyên tắc:

- File private mặc định.
- Chỉ public file thật sự public.
- Dùng signed URL cho file private cần chia sẻ tạm thời.
- Tách bucket public và private nếu hệ thống đủ lớn.
- Bật audit log cho thao tác nhạy cảm.

### 10.4. Virus scan và xử lý file không tin cậy

Với hệ thống cho phép người dùng upload tài liệu, nên cân nhắc:

- Quét malware bằng antivirus hoặc dịch vụ scan.
- Chạy xử lý file trong môi trường sandbox.
- Không thực thi file upload.
- Không render trực tiếp HTML/SVG không kiểm soát.
- Tạo bản preview an toàn thay vì mở file gốc nếu rủi ro cao.

### 10.5. Mã hóa

File storage nên hỗ trợ mã hóa:

- Encryption at rest: mã hóa khi lưu trên đĩa.
- Encryption in transit: dùng HTTPS/TLS khi truyền.
- Customer-managed keys nếu tổ chức có yêu cầu kiểm soát khóa.

Mã hóa không thay thế phân quyền. Nếu ai đó có quyền đọc hợp lệ, họ vẫn có thể tải file đã giải mã qua dịch vụ storage.

## 11. Hiệu năng và khả năng mở rộng

### 11.1. Streaming

Khi upload/download file lớn, backend nên dùng streaming thay vì đọc toàn bộ file vào memory.

Không tốt:

```text
Đọc toàn bộ file 2 GB vào RAM rồi mới ghi storage.
```

Tốt hơn:

```text
Đọc từng chunk nhỏ và stream trực tiếp đến storage hoặc client.
```

### 11.2. Multipart hoặc resumable upload

Với file lớn, nên dùng upload nhiều phần hoặc resumable upload. Cách này giúp:

- Tiếp tục upload khi mạng chập chờn.
- Upload song song nhiều chunk.
- Giảm rủi ro phải upload lại từ đầu.

### 11.3. Cache và CDN

File public hoặc ít thay đổi nên được cache:

- Dùng `Cache-Control`.
- Dùng CDN.
- Dùng file name có version/hash để cache lâu.

Ví dụ:

```text
/assets/app.8f3a1c.js
/products/p_123/images/img_456.webp
```

Nếu file thay đổi nhưng URL giữ nguyên, cần chiến lược cache invalidation rõ ràng.

### 11.4. Range request

Range request cho phép client tải một phần của file. Tính năng này quan trọng với:

- Video streaming.
- Audio.
- PDF lớn.
- Resume download.

Nếu backend tự stream file, cần kiểm tra framework có hỗ trợ HTTP range request không.

## 12. Chi phí lưu trữ

Chi phí file storage thường gồm:

- Dung lượng lưu trữ theo GB/tháng.
- Số request đọc/ghi/list/delete.
- Băng thông egress ra internet hoặc giữa region.
- Phí retrieval nếu dùng storage class lạnh.
- Phí operation của CDN hoặc lifecycle.
- Chi phí backup/versioning.

Các cách kiểm soát chi phí:

- Xóa file tạm và file orphan định kỳ.
- Dùng lifecycle policy.
- Resize/compress ảnh khi phù hợp.
- Dùng CDN cho file public có traffic cao.
- Không bật versioning vô thời hạn nếu không có chính sách dọn.
- Chọn region gần compute và người dùng chính.
- Theo dõi dung lượng theo tenant/project để phát hiện tăng bất thường.

## 13. Backup, retention và disaster recovery

File storage cần được đưa vào chiến lược backup, không chỉ database.

Các câu hỏi cần trả lời:

- Nếu user xóa nhầm file, có khôi phục được không?
- Nếu ứng dụng xóa nhầm một prefix lớn, có cơ chế rollback không?
- File cần giữ tối thiểu bao lâu theo yêu cầu pháp lý?
- Có cần lưu bản sao ở region khác không?
- Backup database có khớp với trạng thái file storage không?
- Khi restore database về thời điểm cũ, các file tương ứng còn tồn tại không?

Các cơ chế thường dùng:

- Versioning.
- Soft delete.
- Object lock hoặc retention policy.
- Replication sang region/bucket khác.
- Backup định kỳ metadata trong database.
- Job kiểm tra tính nhất quán giữa database và storage.

## 14. File storage trong hệ thống AI/RAG

Trong hệ thống RAG, file storage thường lưu tài liệu gốc:

```text
User upload PDF/DOCX/TXT
File gốc được lưu vào storage
Metadata được lưu vào database
Pipeline đọc file, trích xuất text
Text được chia chunk
Embedding được lưu vào vector database
Khi trả lời, hệ thống tham chiếu lại source file hoặc page
```

Một thiết kế phổ biến:

```text
Storage:
rag/documents/{document_id}/original/source.pdf
rag/documents/{document_id}/extracted/text.json
rag/documents/{document_id}/preview/page_001.png

Database:
documents
document_files
chunks
embeddings metadata
processing_jobs
```

Lưu ý:

- Không nên chỉ lưu chunk mà bỏ file gốc nếu cần audit hoặc reprocess.
- Cần lưu version của pipeline để biết chunk được tạo bằng logic nào.
- Khi tài liệu bị xóa, cần xóa cả file gốc, text đã trích xuất, chunk và embedding liên quan.
- File private phải được kiểm soát khi hệ thống sinh citation hoặc link nguồn.

## 15. Các lỗi thiết kế thường gặp

| Lỗi | Hậu quả |
| --- | --- |
| Lưu file lớn trực tiếp vào database mà không có lý do rõ ràng | Database phình to, backup chậm, hiệu năng giảm. |
| Dùng tên file người dùng làm path thật | Trùng tên, path traversal, khó audit. |
| Public cả bucket/container | Lộ dữ liệu private. |
| Không giới hạn kích thước upload | Tốn storage, nghẽn backend, dễ bị abuse. |
| Không kiểm tra content type | Cho phép upload file nguy hiểm hoặc sai định dạng. |
| Không lưu metadata trong database | Khó query, phân quyền, audit và dọn dẹp. |
| Không xử lý transaction lệch giữa database và storage | File orphan hoặc record trỏ tới file không tồn tại. |
| Không có lifecycle policy | File tạm, version cũ và backup chiếm dung lượng mãi mãi. |
| Không dùng signed URL cho file private | Backend quá tải hoặc phải public file không cần thiết. |
| Không có audit log | Khó điều tra ai đã đọc, tải, xóa hoặc thay đổi file. |

## 16. Checklist thiết kế file storage

Khi thiết kế chức năng lưu file, nên trả lời các câu hỏi sau:

- File thuộc loại nào: ảnh, video, tài liệu, log, backup hay dataset?
- File là public, private hay internal?
- Ai được upload, đọc, sửa, xóa?
- File tối đa bao nhiêu MB/GB?
- Có cần preview, thumbnail, OCR hoặc trích xuất text không?
- Metadata nào cần lưu trong database?
- Object key/path được đặt theo quy tắc nào?
- Có cần versioning không?
- File bị xóa là xóa mềm hay xóa vĩnh viễn?
- File tạm được dọn sau bao lâu?
- Có cần antivirus scan không?
- Có cần signed URL hoặc CDN không?
- Có cần backup/replication không?
- Khi database và storage lệch trạng thái thì xử lý thế nào?
- Có đo được dung lượng và chi phí theo user/tenant/project không?

## 17. Kết luận

File storage là thành phần quan trọng trong hầu hết hệ thống hiện đại. Thiết kế tốt không chỉ là "upload file vào một thư mục", mà còn phải quan tâm đến metadata, phân quyền, bảo mật, hiệu năng, chi phí, backup và vòng đời dữ liệu.

Nguyên tắc tổng quát:

```text
Database lưu metadata và quan hệ nghiệp vụ.
File storage lưu nội dung file.
Backend kiểm soát quyền và trạng thái.
Lifecycle, backup và audit giúp hệ thống vận hành bền vững.
```

Nếu hệ thống nhỏ, local file storage có thể đủ cho giai đoạn đầu. Khi hệ thống cần mở rộng, nhiều server, file lớn, signed URL, CDN, độ bền cao hoặc lifecycle tự động, nên cân nhắc object storage hoặc managed file storage trên cloud.
