# Location Directory Manager

Ứng dụng C++20 quản lý địa điểm yêu thích theo [đặc tả](docs/specifiction.txt),
gồm client desktop **Qt 6/QML**, server **Linux epoll/TLS 1.3**, PostgreSQL và SQLite offline.

Đã triển khai: đăng ký/đăng nhập/đăng xuất, phiên có thời hạn, hồ sơ và tìm bạn,
CRUD địa điểm, danh mục, tìm kiếm/bộ lọc, yêu thích, thẻ, ghi chú, chia sẻ theo username,
thông báo khi đăng nhập, nhận/từ chối chia sẻ, dấu nguồn chia sẻ, đồng bộ hai chiều,
giữ bản xung đột, sao lưu và khôi phục có lịch sử.

## Chạy server bằng Docker (Ubuntu 24.04 + PostgreSQL 17)

Tại thư mục gốc trong Bash/WSL:

```bash
cp .env.example .env
# Sửa DB_PASSWORD trong .env thành mật khẩu ngẫu nhiên, chỉ dùng ký tự URL-safe.
bash scripts/dev-cert.sh
sudo chown 10001:10001 certs/server.key
docker compose up --build -d
docker compose logs -f server
```

Server nghe cổng **7443**. PostgreSQL chỉ truy cập trong mạng Compose, dữ liệu lưu volume.
Schema khởi tạo tự động khi volume DB trống. Chứng chỉ tạo bằng script chỉ phục vụ local;
production cần chứng chỉ/hostname thật. Không commit `.env`, private key hoặc database.

## Build toàn bộ trên Ubuntu/WSL

```bash
sudo apt-get update
sudo apt-get install -y g++ cmake ninja-build pkg-config \
  libpqxx-dev libsodium-dev libssl-dev libspdlog-dev nlohmann-json3-dev libgtest-dev \
  qt6-base-dev qt6-declarative-dev qml6-module-qtquick \
  qml6-module-qtquick-controls qml6-module-qtquick-layouts \
  qml6-module-qtquick-templates qml6-module-qtqml-workerscript
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j4
ctest --test-dir build --output-on-failure
./build/client/ldm-client
```

Trong client: host `localhost`, port `7443`, CA là **đường dẫn tuyệt đối tới
`certs/server.crt`** (đường dẫn Linux khi chạy trong WSL, Windows khi chạy native).
Kết nối TLS → đăng ký tài khoản (username 3–32 ký tự chữ/số/underscore, mật khẩu ít nhất
10 byte) → đăng nhập. Tạo hai tài khoản để thử chia sẻ. Bản chia sẻ có nhãn người gửi.

Chọn **Mở offline** với username đã dùng để truy cập SQLite trên máy cá nhân.
Chỉnh sửa vẫn được giữ khi ngắt mạng; đăng nhập lại và đồng bộ khi có mạng.
Thông báo cập nhật khi đăng nhập và mỗi 15 giây. Sao lưu cần chờ dữ liệu đồng bộ xong.

Chỉ build server: `-DBUILD_CLIENT=OFF`. Chỉ build client Windows: `-DBUILD_SERVER=OFF`.
Có thể dùng Qt Creator với Qt 6.4+ và CMake. Với vcpkg, truyền
`-DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake`;
feature `desktop` bổ sung Qt. Compiler/triplet của Qt và vcpkg phải tương thích.

## Server không dùng Docker

Cài PostgreSQL 17+, tạo database/user riêng, chạy migration bằng chủ sở hữu database:

```bash
export DATABASE_URL='postgresql://locations:your_password@localhost:5432/locations'
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f database/migrations/001_initial.sql
export TLS_CERT="$PWD/certs/server.crt"
export TLS_KEY="$PWD/certs/server.key"
export PORT=7443
./build/server/ldm-server
```

Đặt `DATABASE_URL` trước lệnh migration. Giữ `ldm-backup-worker` cùng thư mục với
`ldm-server`; backup dùng fork/exec và socketpair gọi helper này.
Log stdout chứa lỗi request và thống kê số kết nối mỗi phút; dừng bằng SIGINT/SIGTERM.

## Kiểm thử tích hợp và tải

Dùng database **riêng cho kiểm thử** đã chạy migration. Test tạo tài khoản mới với suffix
ngẫu nhiên và giữ dữ liệu để kiểm tra; không dùng database production.

```bash
export DATABASE_URL='postgresql://test_user:test_password@localhost/test_database'
python3 tests/integration/test_flow.py
```

Test tự tạo chứng chỉ tạm, chọn port trống, chạy server và dừng sau khi kiểm tra.
`scripts/setup-test-db.sql` là tùy chọn tạo DB local bằng superuser, chứa mật khẩu
chỉ dùng cho kiểm thử; không sử dụng khi triển khai.

Load tester cần server đang chạy và tài khoản đã đăng ký:

```bash
export LDM_LOAD_PASSWORD='your_test_password'
./build/server/ldm-load localhost 7443 certs/server.crt test_user 100 10
```

Kiểm tra bộ nhớ:

```bash
cmake -S . -B build-asan -G Ninja -DBUILD_CLIENT=OFF -DENABLE_ASAN=ON
cmake --build build-asan -j4
ctest --test-dir build-asan --output-on-failure
LDM_BUILD_DIR=build-asan python3 tests/integration/test_flow.py
# Hoặc sau khi cài valgrind, với bản build Debug không ASan:
bash scripts/test-valgrind.sh
```

## Tài liệu

- [Giao thức và danh sách API](docs/protocol.md)
- [Kiến trúc, cách đồng bộ và giới hạn hiện tại](docs/architecture.md)
- [Schema PostgreSQL](database/migrations/001_initial.sql)
- [Kết quả kiểm tra local và môi trường đã dùng](docs/testing.md)

Server triển khai dùng Ubuntu 24.04 qua Docker. Kiểm tra local có thể dùng WSL Linux;
build Windows native áp dụng cho client, vì server dùng API riêng Linux.
"# LocationDirectoryManager" 
