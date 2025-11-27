# Hệ thống Kiểm chứng Phần cứng STM32F746ZG
Dự án thuộc quyền sở hữu của **Phạm Lê Ngọc Sơn**.

Khung kiểm thử client–server để xác thực các ngoại vi của STM32F746ZG trong môi trường sản xuất.

[Xem thư viện ảnh](media/Gallery.md)

## Mục lục
1. [Tổng quan dự án](#tổng-quan-dự-án)
2. [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
3. [Cấu trúc dự án](#cấu-trúc-dự-án)
4. [Hướng dẫn sử dụng](#hướng-dẫn-sử-dụng)
5. [Giao thức truyền thông](#giao-thức-truyền-thông)
6. [Phương pháp kiểm thử](#phương-pháp-kiểm-thử)

## Tổng quan dự án
Dự án xây dựng hệ thống kiểm chứng phần cứng phục vụ kiểm tra chất lượng sản xuất cho bo mạch STM32F746ZG. Hệ thống tự động xác thực tính đúng đắn của các ngoại vi thông qua giao tiếp UDP.

### Kiến trúc

![Sơ đồ khối của dự án](https://github.com/LeahShl/STM32F756ZG_HW_Verification/blob/main/media/block_diagram.png "Sơ đồ khối của dự án")

Hệ thống theo kiến trúc client–server:
- **Server (chương trình kiểm thử trên PC)**: ứng dụng C chạy Linux, điều phối kiểm thử và ghi log
- **Client (firmware kiểm thử UUT)**: firmware STM32F746ZG thực thi các bài test ngoại vi
- **Giao tiếp**: UDP với giao thức tùy biến trên Ethernet (stack LWIP)

### Ngoại vi được kiểm thử

| Ngoại vi | Phương pháp | Đặc tính chính |
|----------|-------------|----------------|
| **UART** | Loopback UART4 ↔ UART5 | DMA một phần, kiểm tra CRC |
| **SPI** | Loopback (slave ↔ master) | DMA, kiểm tra CRC |
| **I2C** | Loopback (slave ↔ master) | DMA, kiểm tra CRC |
| **ADC** | So sánh điện áp chuẩn | Độ phân giải 12-bit, giá trị tham chiếu định trước |
| **Timer** | Đếm xung/đo thời gian | DMA, đo đếm chính xác |

## Yêu cầu hệ thống

### Phần cứng
- Bo phát triển STM32F746ZG
- PC có cổng Ethernet
- Dây Ethernet nối PC và STM32
- Dây/đầu nối cho UART, SPI, I2C, ADC

### Phần mềm
- **Phía PC**: Linux với GCC và Make
- **Phía STM32**: STM32CubeIDE hoặc toolchain ARM tương thích
- Cấu hình mạng: PC và STM32 cùng subnet

## Cấu trúc dự án
Chú thích:
- tệp thường
- \[Thư mục\]

```
HW_Verification/
|-- FILES_FOR_PC/                     # Mã server chạy trên PC
|   |-- debug_tools/
|   |   `-- dummy_server.c           # Server giả lập khi chưa có UUT
|   |-- main.c                       # Tệp main của chương trình
|   |-- Makefile                     # Biên dịch với GNU make
|   |-- README.md                    # Readme cho mã PC
|   |-- tests_db.c                   # Mã nguồn SQLite
|   |-- tests_db.h                   # Header SQLite
|   `-- usage_example.sh             # Ví dụ sử dụng cho PC
|
|-- README.md                        # <--- Tệp này
|
`-- STM32CUBE_project/               # Mã STM32 (các tệp sinh tự động được lược bỏ)
    |-- Core/
    |-- Src/
    |   `-- main.c                   # Tệp main STM32
    |-- HW_Verification.ioc          # Cấu hình MX
    `-- UUT/                         # Mã cho thiết bị UUT
        |-- Inc/
        |   |-- adc_test.h
        |   |-- hw_verif_crc.h
        |   |-- hw_verif.h
        |   |-- hw_verif_udp.h
        |   |-- i2c_test.h
        |   |-- spi_test.h
        |   |-- test_consts.h
        |   |-- tests.h
        |   |-- timer_test.h
        |   `-- uart_test.h
        `-- Src/
            |-- adc_test.c
            |-- hw_verif.c
            |-- hw_verif_crc.c
            |-- hw_verif_udp.c
            |-- i2c_test.c
            |-- spi_test.c
            |-- tests.c
            |-- timer_test.c
            `-- uart_test.c
```

## Hướng dẫn sử dụng

### 1. Biên dịch chương trình PC
1. Mở terminal và `cd` vào `FILES_FOR_PC`
2. Chạy `make`

### 2. Cấu hình mạng
Bo STM32 được đặt IP tĩnh `10.0.1.100`. Nếu mạng của bạn không có thiết bị nào dùng IP này, bạn có thể giữ nguyên. Chỉ cần bảo đảm PC cùng mạng:

1. Xác định tên giao diện Ethernet bằng `ip addr show` (ví dụ `eth0` hoặc `enp3s0`). Thay `<your-eth-interface>` bằng tên tìm được.
2. Thêm địa chỉ IP mới cho giao diện (trong dải `10.0.1.x`, tránh `10.0.1.100`, ví dụ `10.0.1.101`):
   ```
   sudo ip addr add 10.0.1.101/24 dev <your-eth-interface>
   ```
3. Kích hoạt giao diện:
   ```
   sudo ip link set <your-eth-interface> up
   ```
4. Kiểm tra lại:
   ```
   ip addr show <your-eth-interface>
   ```

### 3. Biên dịch firmware STM32
1. Mở STM32CubeIDE
2. Chọn `File > Open Projects from Filesystem...`
3. Bên cạnh `Import source` chọn `Directory...`
4. Chọn thư mục `STM32CUBE_project`
5. Bấm `Finish`
6. Kết nối bo STM32 với PC qua cổng USB ST-Link
7. Build dự án

### 4. Đấu nối phần cứng
#### Ethernet
**Kết nối**: Cắm cáp Ethernet trực tiếp giữa bo STM32 và PC.

#### UART
Cấu hình GPIO:
| Ngoại vi | Chân | GPIO |
|----------|------|------|
| UART4 | TX | PC10 |
| UART4 | RX | PC11 |
| UART5 | TX | PC12 |
| UART5 | RX | PD2 |

**Kết nối**: Nối PC10 (UART4_TX) với PD2 (UART5_RX) và PC11 (UART4_RX) với PD12 (UART5_TX).

#### I2C
I2C1 là master, I2C2 là slave.
| Ngoại vi | Chân | GPIO |
|----------|------|------|
| I2C1 | SCL | PB8 |
| I2C1 | SDA | PB9 |
| I2C2 | SCL | PB10 |
| I2C2 | SDA | PB11 |

**Kết nối**: PB8 (I2C1_SCL) với PB10 (I2C2_SCL) và PB9 (I2C1_SDA) với PB11 (I2C2_SDA).

#### SPI
SPI1 là master, SPI4 là slave. Dùng Hardware NSS.
| Ngoại vi | Chân | GPIO |
|----------|------|------|
| SPI1 | NSS | PA4 |
| SPI1 | SCK | PA5 |
| SPI1 | MISO | PA6 |
| SPI1 | MOSI | PB5 |
| SPI4 | NSS | PE4 |
| SPI4 | SCK | PE2 |
| SPI4 | MISO | PE5 |
| SPI4 | MOSI | PE6 |

**Kết nối**: PA4 (SPI1_NSS) ↔ PE4 (SPI4_NSS), PA5 (SPI1_SCK) ↔ PE2 (SPI4_SCK), PA6 (SPI1_MISO) ↔ PE5 (SPI4_MISO), PB5 (SPI1_MOSI) ↔ PE6 (SPI4_MOSI).

#### ADC
| Ngoại vi | Chân | GPIO |
|----------|------|------|
| ADC1 | IN0 | PA0 |

**Kết nối**: Đưa PA0 (ADC1/0) lên mức 3V3.

#### Timer
Không cần đấu nối thêm.

### 5. Chạy kiểm thử
Xem thêm `FILES_FOR_PC/usage_example.sh` và [README của chương trình PC](https://github.com/LeahShl/STM32F756ZG_HW_Verification/blob/main/FILES_FOR_PC/README.md).

1. Chạy test
   ```
   ./hw_verif --all                          # Chạy tất cả các bài test
   ./hw_verif --all -n 10                    # Chạy tất cả với số vòng lặp chỉ định
   ./hw_verif -usi "bitpatternfoobar"        # Chạy các test giao tiếp với thông điệp tùy chỉnh
   ./hw_verif -uit                           # Chạy các test giao tiếp với thông điệp mặc định
   ```
2. In kết quả theo ID
   ```
   ./hw_verif get 123                        # Một bài test
   ./hw_verif get 12 34 56                   # Nhiều bài test
   ```
3. Xuất toàn bộ dữ liệu ra CSV (mặc định in stdout)
   ```
   ./hw_verif export > data.csv
   ```
4. Hiển thị trợ giúp
   ```
   ./hw_verif --help
   ```

## Giao thức truyền thông

### Gói lệnh (PC → STM32)

| Trường | Kích thước | Ý nghĩa |
|--------|------------|---------|
| Test ID | 4 byte | Mã bài test |
| Peripheral | 1 byte | Loại ngoại vi |
| Iterations | 1 byte | Số vòng lặp |
| Bit Length | 1 byte | Độ dài thông điệp |
| Bit Pattern | 0–255 byte | Thông điệp/bit pattern (tùy chọn) |

### Gói phản hồi (STM32 → PC)

| Trường | Kích thước | Ý nghĩa |
|--------|------------|---------|
| Test ID | 4 byte | Mã bài test |
| Result | 1 byte | 0x01 = PASS, 0xFF = FAIL |

## Phương pháp kiểm thử
### Ngoại vi giao tiếp (UART, SPI, I2C) – kiểm tra loopback có CRC
1. Gửi một bit pattern từ ngoại vi này sang ngoại vi kia.
2. Nhận lại bit pattern.
3. So sánh với bản gốc bằng CRC.
4. Ngoại vi đạt nếu tất cả vòng lặp đều đạt.

### Kiểm thử ADC
1. Một lần đo đạt nếu mẫu ADC nằm trong sai số cho phép.
2. Ngoại vi đạt nếu tất cả vòng lặp đều đạt.

### Kiểm thử Timer
1. TIM2 (advanced) đếm lên.
2. TIM6 (basic) được reset mỗi 10 ms.
3. Lấy mẫu TIM2 tại mỗi lần TIM6 reset cho đến khi thu đủ N_SAMPLES (=10).
4. Một lần đo đạt nếu mọi mẫu cách nhau 54.000 xung.
5. Ngoại vi đạt nếu tất cả vòng lặp đều đạt.
