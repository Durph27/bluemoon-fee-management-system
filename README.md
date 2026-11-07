# BlueMoon Fee Management System

Hệ thống quản lý thu phí chung cư: React + JavaScript + CSS + Vite,
Spring Boot + Java 21, PostgreSQL. Backend Sprint 1 có RBAC, căn hộ/cư trú,
loại phí/kỳ thu, bảng kê thủ công, thu tiền/biên lai và quỹ bảo trì.
Frontend hiện vẫn là trang kiểm tra kết nối.

Xem [tài liệu API contract](docs/api-contract.md) và [quy ước thiết kế API](docs/api/design.md) để
khởi tạo admin, tích hợp API và kiểm thử. Các API khác còn ở mức thiết kế được
phân biệt bằng `x-implemented` trong OpenAPI.

## Yêu cầu

- JDK 21 trở lên, cấu hình `JAVA_HOME` trỏ tới thư mục JDK.
- Node.js 22.12+ (hoặc Node.js 24) và npm.
- PostgreSQL đang chạy, biết tài khoản và mật khẩu.
- Maven được wrapper tải tự động; lần chạy đầu cần Internet.

## 1. Cấu hình môi trường

Từ thư mục gốc bằng PowerShell, chỉ copy nếu chưa có file:

```powershell
Copy-Item .env.example .env
Copy-Item frontend/.env.example frontend/.env
```

Sửa `.env` ở gốc:

```properties
DB_HOST=localhost
DB_PORT=5432
DB_NAME=bluemoon_db
DB_USERNAME=postgres
DB_PASSWORD=YOUR_POSTGRES_PASSWORD
SERVER_PORT=8080
CORS_ALLOWED_ORIGINS=http://localhost:5173
```

Điền mật khẩu thật tại `DB_PASSWORD`. Spring Boot đọc file tự động khi chạy từ gốc
hoặc `backend/`. Dùng cú pháp Java properties: không bọc giá trị bằng dấu nháy;
nếu mật khẩu chứa dấu `\`, ghi thành `\\`. Biến môi trường hệ điều hành được ưu
tiên hơn file. Không commit `.env`.

`frontend/.env` mặc định gọi `/api`, chuyển tiếp đến `http://localhost:8080`.
Nếu đổi cổng backend, sửa `BACKEND_URL` ở file này và khởi động lại Vite.
Biến `VITE_` công khai trong trình duyệt; không đặt mật khẩu tại đó.

## 2. Tạo database

Trong pgAdmin: kết nối PostgreSQL bằng tài khoản `postgres`, mở Query Tool trên
database `postgres`, chạy lệnh sau một lần nếu chưa có `bluemoon_db`:

```sql
CREATE DATABASE bluemoon_db;
```

Hoặc dùng psql, nhập mật khẩu khi được hỏi:

```powershell
psql -h localhost -p 5432 -U postgres -d postgres -c "CREATE DATABASE bluemoon_db;"
```

Nếu psql chưa trong PATH, dùng đường dẫn đầy đủ tới `bin/psql.exe` của PostgreSQL.
Database có sẵn thì bỏ qua bước tạo. Không cần xóa dữ liệu hoặc sửa xác thực.

## 3. Chạy backend

Terminal thứ nhất:

```powershell
cd backend
.\mvnw.cmd spring-boot:run
```

macOS/Linux: `cd backend && ./mvnw spring-boot:run`.

Kiểm tra từ terminal khác:

```powershell
Invoke-RestMethod http://localhost:8080/api/hello
Invoke-RestMethod http://localhost:8080/api/health
```

Health trả `status=UP`, `database=UP` mới xác nhận PostgreSQL thật vì backend
thực thi `SELECT 1`. Database chưa sẵn sàng có thể khiến backend không khởi động.

## 4. Chạy frontend

Terminal thứ hai, từ thư mục gốc:

```powershell
cd frontend
npm.cmd ci
npm.cmd run dev
```

Mở <http://localhost:5173>. Trang chủ hiển thị trạng thái React, backend và PostgreSQL.
Trên macOS/Linux dùng `npm` thay cho `npm.cmd`.

## 5. Sao lưu và phục hồi database

Dự án cung cấp công cụ `tools/backup_restore.py` hỗ trợ sao lưu và phục hồi database PostgreSQL. Script tự động đọc thông số kết nối từ file `.env` ở thư mục gốc (hoặc biến môi trường hệ thống).

### Sao lưu (Backup)

Tạo bản sao lưu định dạng nén (`.dump`, khuyến nghị) vào thư mục `backups/`:

```powershell
python tools/backup_restore.py backup
```

Tùy chọn:
- `--format plain`: Xuất file SQL thuần (`.sql`).
- `-o <path>` hoặc `--output <path>`: Tùy chọn đường dẫn file/thư mục lưu bản sao lưu.
- `-d <dbname>`: Ghi đè tên database mục tiêu (mặc định lấy từ `DB_NAME` trong `.env`).
- `--schema-only`: Chỉ sao lưu cấu trúc bảng (schema), không sao lưu dữ liệu.
- `--data-only`: Chỉ sao lưu dữ liệu, không sao lưu cấu trúc bảng.

### Phục hồi (Restore)

Phục hồi từ bản sao lưu gần nhất hoặc chỉ định file:

```powershell
# Tự động chọn bản sao lưu mới nhất trong thư mục backups/
python tools/backup_restore.py restore

# Phục hồi từ file cụ thể
python tools/backup_restore.py restore backups/bluemoon_db_20261107_162924.dump

# Bỏ qua xác nhận (phù hợp cho script tự động hóa)
python tools/backup_restore.py restore -y backups/bluemoon_db_20261107_162924.dump
```

Script tự động nhận diện định dạng (`.dump` dùng `pg_restore`, `.sql` dùng `psql`), mặc định dọn dẹp đối tượng cũ trước khi phục hồi (`--clean --if-exists`), và hỗ trợ cờ `--create-db` nếu database mục tiêu chưa tồn tại.

### Xem danh sách bản sao lưu

```powershell
python tools/backup_restore.py list
```

## Build và test

```powershell
# Tại frontend/
npm.cmd run build
npm.cmd run lint

# Tại backend/
.\mvnw.cmd test
```

Backend test dùng H2 trong bộ nhớ, chạy độc lập với PostgreSQL và mật khẩu local.
Test kiểm tra health qua HTTP, CORS, API hello và tình huống database lỗi.
Để kiểm tra PostgreSQL thật, chạy backend rồi gọi `/api/health` như trên.

## Cấu trúc

```text
frontend/src/
  api/          Axios client, API calls
  assets/       Tài nguyên tĩnh
  components/   Component dùng chung
  pages/        Trang giao diện
  hooks/        React hooks
  App.jsx
backend/src/main/java/com/bluemoon/backend/
  config/ controller/ service/ repository/ entity/ dto/ exception/
  BackendApplication.java
backend/src/main/resources/application.properties
backend/pom.xml
backend/mvnw.cmd
docs/
  api/
    data-dictionary.md
    design.md
    endpoints.md
    openapi.json
  api-contract.md
  BlueMoon_ERD.drawio
  BlueMoon_UML.asta
  diagrams.md
tools/
  backup_restore.py
  sync_model_diagrams.py
  sync_sprint1_contract.py
```

## Xử lý lỗi

- JAVA_HOME: trỏ đến thư mục JDK, không phải bin/ hoặc JRE; mở lại terminal.
- Connection refused: kiểm tra service PostgreSQL, DB_HOST, DB_PORT.
- Password authentication failed: kiểm tra DB_USERNAME, DB_PASSWORD.
- Database does not exist: tạo database hoặc sửa DB_NAME.
- Frontend chưa kết nối: kiểm tra backend 8080, BACKEND_URL, tải lại trang.
- Cổng 5173 bị chiếm: dừng tiến trình khác hoặc đổi cổng Vite và cập nhật CORS.
- Health HTTP 503: API còn chạy nhưng truy vấn database thất bại; kiểm tra log backend.

Tài liệu: [API contract](docs/api-contract.md), [quy ước và nghiệp vụ API](docs/api/design.md),
[danh mục endpoint](docs/api/endpoints.md), [đối chiếu thực thể database](docs/api/data-dictionary.md).

### Sơ đồ thiết kế

Sơ đồ hiện hành: [ERD bằng draw.io](docs/BlueMoon_ERD.drawio), [UML bằng Astah](docs/BlueMoon_UML.asta).
Ảnh trong `docs/ERD_Preview` và `docs/UML_Preview` được xuất từ các file nguồn hiện hành.
Xem [mục lục, ánh xạ schema và cách tái tạo](docs/diagrams.md).
