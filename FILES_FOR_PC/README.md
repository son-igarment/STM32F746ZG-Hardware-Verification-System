# Chương trình CLI Linux cho kiểm chứng STM32F756ZG
Dự án thuộc quyền sở hữu của **Phạm Lê Ngọc Sơn**.

Ứng dụng kiểm thử các ngoại vi STM32F756ZG qua UDP và lưu kết quả vào SQLite.

## Mục lục
1. [Tổng quan](#tổng-quan)
2. [Biên dịch](#biên-dịch)
3. [Cấu hình mạng](#cấu-hình-mạng)
4. [Cách dùng](#cách-dùng)
5. [Ví dụ sử dụng](#ví-dụ-sử-dụng)
6. [Cơ sở dữ liệu SQLite3](#cơ-sở-dữ-liệu-sqlite3)

## Tổng quan
Chương trình gửi lệnh kiểm thử tới vi điều khiển STM32F756ZG qua UDP và ghi kết quả vào cơ sở dữ liệu cục bộ. Hỗ trợ kiểm thử nhiều ngoại vi gồm UART, SPI, I2C, ADC và Timer.

### Cấu trúc tệp
Chú thích:
- tệp thường
- \[Thư mục\]
```
FILES_FOR_PC/                     # Mã server chạy trên PC
|-- debug_tools/
|   `-- dummy_server.c            # Server giả lập khi chưa có UUT
|-- main.c                        # Tệp main của chương trình
|-- Makefile                      # Biên dịch với GNU make
|-- README.md                     # <--- Tệp này
|-- tests_db.c                    # Mã nguồn SQLite
|-- tests_db.h                    # Header SQLite
`-- usage_example.sh              # Ví dụ sử dụng
```

## Biên dịch
### Trước khi biên dịch
1. Nếu dùng địa chỉ IP khác, chỉnh `UUT_ADDR` trong `main.c`.
2. Nếu muốn lưu database tại thư mục hiện tại thay vì thư mục home, thêm dòng `#define LOCAL_DB_PATH 1` vào `tests_db.h`. Khi đó database sẽ được lưu tại nơi bạn chạy chương trình, hãy `cd` đúng thư mục trước khi chạy.

### Yêu cầu
- Trình biên dịch GCC
- Tiện ích Make
- Thư viện C chuẩn
- Hệ thống POSIX (Linux/Unix)

### Cách build
```
# Biên dịch chương trình
make

# Xóa file sinh ra
make clean
```

## Cấu hình mạng
### Kết nối với bo STM32
Bo STM32 được đặt IP tĩnh `10.0.1.100`. Nếu mạng bạn không có thiết bị nào dùng IP này, cứ giữ nguyên. Chỉ cần đảm bảo PC cùng subnet:

1. Tìm tên giao diện Ethernet bằng `ip addr show` (ví dụ `eth0` hoặc `enp3s0`). Thay `<your-eth-interface>` bằng tên tìm được.
2. Thêm IP mới cho giao diện (dải `10.0.1.x`, trừ `10.0.1.100`, ví dụ `10.0.1.101`):
   ```
   sudo ip addr add 10.0.1.101/24 dev <your-eth-interface>
   ```
3. Bật giao diện:
   ```
   sudo ip link set <your-eth-interface> up
   ```
4. Kiểm tra lại:
   ```
   ip addr show <your-eth-interface>
   ```

### Dùng server giả lập
Thư mục `debug_tools` chứa server giả lập để kiểm thử phần mềm PC khi chưa có bo STM32. Server này lắng nghe và trả thành công cho mọi yêu cầu test. PC software được viết trước firmware STM32 nên công cụ này hỗ trợ thử nghiệm không cần bo thật.

Để dùng server giả lập:
1. Đổi `UUT_ADDR` trong `main.c` thành `127.0.0.1` và biên dịch theo hướng dẫn trên.
2. Biên dịch `dummy_server.c`:
   ```
   gcc dummy_server.c -o dummy_server
   ```
3. Chạy `./dummy_server`
4. Mở terminal khác và chạy `./hw_tester` với tùy chọn bất kỳ.

## Cách dùng
### Cú pháp lệnh
```
./hw_tester [OPTIONS]
./hw_tester [COMMAND]
```

### Tùy chọn kiểm thử

| Cờ | Mô tả | Hỗ trợ thông điệp |
|----|-------|-------------------|
| `-u [msg]` | Chạy test UART | Có thể kèm thông điệp |
| `-s [msg]` | Chạy test SPI | Có thể kèm thông điệp |
| `-i [msg]` | Chạy test I2C | Có thể kèm thông điệp |
| `-a` | Chạy test ADC | Không hỗ trợ thông điệp |
| `-t` | Chạy test Timer | Không hỗ trợ thông điệp |
| `--all [msg]` | Chạy mọi test | Thông điệp áp dụng cho UART/SPI/I2C |

### Tùy chọn bổ sung

| Cờ | Mô tả |
|----|-------|
| `-n <int>` | Số vòng lặp (0-255) |
| `-h, --help` | Hiển thị trợ giúp |

### Lệnh lấy dữ liệu

| Lệnh | Mô tả |
|------|-------|
| `get <id1> <id2> ...` | In dữ liệu test theo ID |
| `export` | Xuất toàn bộ dữ liệu dạng CSV |

### Quy tắc và ràng buộc
Khi dùng các tùy chọn kiểm thử (không áp dụng cho `get` hoặc `export`):
1. **Cờ bắt buộc**: Phải có ít nhất một cờ test (`-u`, `-s`, `-i`, `-a`, `-t` hoặc `--all`)
2. **Không lặp**: Mỗi cờ chỉ xuất hiện một lần
3. **Quy tắc thông điệp**:
   - Cờ `a` và `t` không nhận thông điệp
   - Khi gộp cờ có `a` hoặc `t` thì không được theo sau bởi thông điệp
   - Các cờ giao tiếp (`u`, `s`, `i`) có thể dùng chung thông điệp khi gộp
4. **Giới hạn vòng lặp**: 0-255

Khi dùng lệnh `get` phải cung cấp ít nhất một test ID.

Lệnh `export` phải đứng một mình.

### Mã thoát
Chạy thành công trả 0. Nếu lỗi, chương trình trả về:

| Tên lỗi | Mã | Mô tả |
|---------|----|-------|
| ARGS_ERROR | 1 | Lỗi phân tích tham số dòng lệnh |
| UDP_ERROR | 2 | Lỗi giao tiếp UDP |
| SQLITE_ERROR | 3 | Lỗi cơ sở dữ liệu SQLite3 |


## Ví dụ sử dụng
### Kiểm thử cơ bản
```
# Test UART với thông điệp mặc định
./hw_tester -u

# Test SPI với thông điệp tùy chỉnh
./hw_tester -s "Custom SPI test"

# Test nhiều ngoại vi
./hw_tester -u -s -i

# Test ADC và Timer (không thông điệp)
./hw_tester -a -t
```

### Gộp cờ
```
# Test UART, SPI, I2C với thông điệp chung
./hw_tester -usi "Shared test message"

# Test ADC và Timer cùng lúc
./hw_tester -at

# Kết hợp gộp và test ADC riêng
./hw_tester -si "Communication test" -a
```

### Vòng lặp và chạy tất cả test
```
# Chạy tất cả test, mỗi test 10 vòng lặp
./hw_tester --all -n 10

# Chạy UART 5 lần với thông điệp tùy chỉnh
./hw_tester -u "Iteration test" -n 5

# Chạy mọi test với thông điệp chung cho các ngoại vi giao tiếp
./hw_tester --all "Universal test message"
```

### Lệnh lấy dữ liệu
```
# In kết quả một test cụ thể
./hw_tester get 123

# In kết quả nhiều test
./hw_tester get 12 34 56

# Xuất toàn bộ kết quả ra CSV
./hw_tester export > tests.csv
```
## Cơ sở dữ liệu SQLite3
Chương trình ghi metadata và kết quả từng test vào SQLite3 để theo dõi và xem lại.

### Lược đồ
| Cột | Kiểu | Mô tả |
|-----|------|------|
| `test_id` | INTEGER | Khóa chính tự tăng |
| `timestamp` | TEXT | Thời gian ISO 8601 |
| `duration` | REAL | Thời lượng test (giây) |
| `result` | INTEGER | Mã kết quả (1=PASS, 0=FAIL) |

### Vị trí lưu trữ
Nếu không định nghĩa `LOCAL_DB_PATH`, database nằm tại:
```
~/HW_tester/records.db
```
### Lấy dữ liệu
Xem phần [**Cách dùng**](#cách-dùng) và [**Ví dụ sử dụng**](#ví-dụ-sử-dụng).

Định dạng database và CSV xuất ra tương thích với các công cụ như SQLite3 CLI, Excel, module sqlite3 của Python và nhiều công cụ phân tích khác.
