# VortexETE MCU — SN32F407

## Giới thiệu dự án

MCU SN32F407 thu thập mực nước, tốc độ/hướng gió qua RS485 Modbus, nhiệt độ/độ ẩm từ DHT11 và xung cảm biến mưa. MCU điều khiển FPGA ACG525 chụp ảnh, nhận kết quả xử lý nước/ROI qua SPI, kiểm tra CRC và tính nhất quán, sau đó gửi dữ liệu qua modem A7680C bằng MQTT.

Đây là mã MCU V14 (`build_tag = 0x57443134`, WD14), dùng cùng FPGA WATER_FIRST/schema `0x43`. Project chỉ có một `main.c` và một backend SPI GPIO. Gói này không chứa website.

Khi cảm biến nước có dữ liệu hợp lệ, camera được phép chụp khi **không có xung mưa hợp lệ ít nhất 5 phút và mực nước lớn hơn 100 mm**. Khi cảm biến nước lỗi, chưa có mẫu hoặc mẫu cũ từ 15 giây, camera chạy dự phòng mà không cần chờ điều kiện mưa. Việc bật/tắt camera dùng điều khiển của FPGA; không phải cắt nguồn vật lý.

FPGA trả mức theo độ phủ nước trong ROI: dưới 5% là NORMAL; từ 5% đến dưới 25% là WARNING; từ 25% là CRITICAL. Khung hợp lệ không có pixel nước được trả NORMAL với ROI bằng 0. Khung lỗi được giữ là không hợp lệ. Độ phủ ảnh chưa phải độ sâu nước hay độ tin cậy mô hình; hai giá trị chưa tính được tiếp tục gửi `null`.

### Bố trí thư mục

| Đường dẫn | Nội dung |
|---|---|
| `VortexETE_MCU.uvprojx` | Project Keil, target `Target_1` |
| `VortexETE_MCU.uvoptx`, `EventRecorderStub.scvd` | Tùy chọn debug và file hỗ trợ project |
| `firmware/` | Mã ứng dụng và driver đang sử dụng |
| `RTE/` | Startup, system clock và cấu hình CMSIS |
| `tests/` | Kiểm thử MCU trên máy tính và dấu kiểm nguồn gốc |

Mở `.uvprojx` bằng Keil, chọn ArmClang 6.24, SONiX.SN32F4_DFP 1.1.1 và CMSIS 6.2.0; Rebuild All rồi Download. Giữ microlib và bộ nhớ đã cấu hình: RAM 8 KiB, Flash `0x7FFC`, stack `0x600`, heap 0. VS Code dùng để sửa mã; build/nạp vẫn qua Keil với gói này.

## Hướng dẫn chỉnh code

### 1. Chỉnh cấu hình tại `firmware/app_config.h`

| Nhóm | Thông số | Mặc định và ý nghĩa |
|---|---|---|
| Điều kiện camera | `CAMERA_RAIN_QUIET_MS` | `300000u`: 5 phút từ xung mưa hợp lệ cuối |
| Điều kiện camera | `CAMERA_WATER_HIGH_THRESHOLD_MILLI_MM` | `100000u`: nước phải **lớn hơn** 100 mm; đơn vị 0,001 mm |
| Độ mới mẫu nước | `CAMERA_SENSOR_STALE_MS` | `15000u`: từ 15 giây trở lên coi là cũ |
| Camera dự phòng | `CAMERA_WATER_FAILOVER_ENABLE` | `1`: bật khi mẫu nước lỗi/chưa có/cũ; đặt `0` để tắt dự phòng |
| Chu kỳ dự phòng | `CAMERA_FALLBACK_INTERVAL_MS` | `10000u`: Normal/Warning chờ 10 giây; Critical dùng chu kỳ Critical |
| Thử không cảm biến | `CAMERA_TEST_WITHOUT_SENSORS` | `0`: vận hành theo cảm biến; `1`: bỏ gate cảm biến để thử camera |
| Chu kỳ thử | `CAMERA_TEST_INTERVAL_MS` | `1000u`: dùng trong chế độ thử không cảm biến |
| Chu kỳ thông thường | `FPGA_NORMAL_DEMO_INTERVAL_MS`, `FPGA_WARNING_INTERVAL_MS`, `FPGA_CRITICAL_INTERVAL_MS` | `10000u`, `60000u`, `1000u`, tính sau kết quả trước; còn cộng thời gian xử lý |
| Chờ FPGA | `FPGA_SCAN_TIMEOUT_MS`, `FPGA_RETRY_BACKOFF_MS` | `45000u`, `30000u` |
| Độ mới kết quả | `FPGA_RESULT_STALE_MS` | `90000u` |
| Bus cảm biến | `SENSOR_BAUD`, `SENSOR_PARITY`, `SENSOR_STOP_BITS` | 9600, N, 1; áp dụng chung UART RS485 |
| Địa chỉ cảm biến | `WATER_MODBUS_ID`, `WIND_MODBUS_ID`, `DIRECTION_MODBUS_ID` | Nước 1; gió YJ-FS 5; hướng gió 6 |
| Đọc YJ-FS | `WIND_MODBUS_FUNCTION`, `WIND_MODBUS_REGISTER` | Function 3, thanh ghi 0, đọc 1 thanh ghi |
| Chờ YJ-FS | `WIND_TX_TIMEOUT_MS`, `WIND_REPLY_TIMEOUT_MS`, `WIND_COMPLETE_ON_LENGTH` | 500 ms chờ/phát; 1000 ms nhận sau phát; `1` kết thúc khi đủ khung |
| Quy đổi nước | `WATER_MILLI_MM_PER_RAW` | `1000u`: 1 raw = 1 mm, phải khớp tài liệu cảm biến thực |
| Quy đổi gió | `WIND_MILLI_MPS_PER_RAW` | `100u`: 1 raw = 0,1 m/s; chưa xác nhận hệ số YJ-FS thực |
| Quy đổi hướng | `DIRECTION_MILLI_DEG_PER_RAW` | `45000u`: 1 raw = 45 độ, kiểm tra theo cảm biến thực |
| Mưa | `RAIN_UM_PER_TIP` | `200u`: giả định 0,2 mm/xung; thay khi hiệu chuẩn |
| Chống dội mưa | `RAIN_DEBOUNCE_MS` | `1000u`: khoảng tối thiểu giữa các cạnh được nhận |
| DHT11 | `DHT11_WARMUP_MS`, `DHT11_READ_INTERVAL_MS`, `DHT11_STALE_MS` | 2 giây khởi động, đọc mỗi 5 giây, cũ sau 15 giây |
| SIM/MQTT | `SIM_APN`, `SIM_UART_BAUD`, `MQTT_BROKER`, `MQTT_PORT` | APN rỗng, UART 115200, `broker.emqx.io:1883` |
| Định danh | `MQTT_DEVICE_ID`, `MQTT_CLIENT_ID` | Trạm 1; đặt client ID riêng cho từng MCU |
| Telemetry | `MQTT_PUBLISH_INTERVAL_MS` | `10000u` |

Ví dụ đổi thời gian không mưa thành 1 phút và ngưỡng nước thành 200 mm:

```c
#define CAMERA_RAIN_QUIET_MS 60000u
#define CAMERA_WATER_HIGH_THRESHOLD_MILLI_MM 200000u
```

Sửa định nghĩa đang có, tránh thêm định nghĩa trùng. Nếu Keil có `-D` ghi đè macro `#ifndef`, cần sửa cả giá trị ghi đè đó. Rebuild và nạp lại sau khi chỉnh.

### 2. Chọn đúng file khi đổi chức năng

| Muốn chỉnh | File cần xem |
|---|---|
| Thứ tự khởi tạo, lịch phục vụ | `firmware/main.c` |
| Điều kiện chụp, dự phòng, chu kỳ, đóng gói kết quả MQTT | `firmware/fpga_decision.c` |
| SPI read/write, ID, CRC, kiểm tra kết quả và ngưỡng 5/25 | `firmware/fpga_spi.c`, `fpga_spi.h` |
| Phát xung SPI GPIO, chân SPI, gửi lệnh B9 | `firmware/spi_gpio_master.c`, `spi0_master.h` |
| Khung Modbus, đọc gió/nước/hướng, CRC và timeout | `firmware/rs485.c`, `rs485.h` |
| UART1, chiều RS485, tùy chọn DE | `firmware/rs485_sn32f407.c`, `rs485_sn32f407_config.h` |
| Đếm mưa và chu kỳ giờ/ngày | `firmware/rain.c`, `rain_sn32f407.c` |
| Giải mã DHT11 và ngắt GPIO2 | `firmware/dht11.c`, `dht11_sn32f407.c`, `rain_sn32f407.c` |
| Giá trị telemetry, đổi raw sang đơn vị | `firmware/telemetry.c` |
| Luồng AT/MQTT, đọc điện áp và sóng SIM | `firmware/sim_a7680c.c`, `sim_uart_sn32f407.c` |
| P2.1 lên HIGH điều khiển TXS OE/EN | `firmware/txs_spi_p21.h` |

SPI hiện dùng P1.0 SCK, P1.8 CS_n, P1.2 MOSI, P1.1 MISO; Mode 0, MSB trước. RS485 dùng P2.2 TX/P2.3 RX; SIM dùng P3.1 TX/P3.2 RX; mưa P2.8; DHT11 P2.9. Đổi chân phải sửa driver và dây tương ứng. P2.1 chỉ điều khiển OE/EN, không cấp nguồn VCCA/VCCB cho TXS; mức HIGH phụ thuộc điện áp I/O MCU và phải phù hợp chân OE thực tế. Manual DE mặc định có thể trùng P1.0 SCK, cần đổi cấu hình và dây trước khi bật.

Khi đổi mức 5/25, sửa đồng thời FPGA và kiểm tra `FPGA_Level()`/bộ kiểm tra trường ngưỡng trong `fpga_spi.c`. Khi đổi kích thước ảnh hoặc schema, phải đổi cả parser MCU; không chỉ sửa FPGA. Giữ một backend SPI trong target để tránh lỗi multiply defined.

### 3. Các hành vi cần giữ khi sửa

- Lệnh B9 đầu chương trình đưa Flash Puya dùng chung SPI vào ngủ theo cách đã dùng trên kit; cờ gửi xong không phải ACK và không phải cách ly phần cứng. Không thêm AB/66/99 khi đang liên lạc FPGA dùng chung CS.
- Đóng gate khi đang xử lý: khung hiện tại được hoàn tất/đọc/ACK, chỉ ngừng START mới. `FPGA_RequestScan()` vẫn chịu gate.
- DHT dùng mức DATA thật của mỗi cạnh; foreground nhường trong cửa sổ nhận DHT. Không tự đảo mức cạnh hoặc tắt ngắt dài.
- Mưa do MCU tính: 200 µm/xung mặc định; kỳ 1 giờ/24 giờ theo uptime, không theo ngày lịch; reset mất bộ đếm. Tràn hàng đợi làm kỳ bị ảnh hưởng không hợp lệ.
- `voltage` lấy VBAT do modem báo qua CBC; `wave_level` từ CSQ, không phải điện áp adapter hay phần trăm tốc độ mạng. Không có số mới thì gửi `null`.
- SPI/camera hoạt động khi SIM chưa kết nối; gửi MQTT cần modem được cấp nguồn và có mạng. MCU kiểm CRC/ngưỡng không có nghĩa đã xác nhận độ chính xác ảnh ngoài thực tế.

### 4. Kiểm tra sau khi chỉnh

Trên máy có GCC và Python 3, trong thư mục project:

```bash
python3 tests/check_package.py
bash tests/run_decision_tests.sh
```

`check_package.py` kiểm tính nguyên vẹn gói phát hành; sau khi chủ động sửa nguồn, hash thay đổi là bình thường. Bộ kiểm thử còn lại kiểm C/driver/chính sách bằng mô hình. Sau đó Rebuild Keil, kiểm map Flash/RAM rồi thử trên kit. Chưa build/nạp Keil cho lần tinh gọn này.

Đọc Watch: `fpga_diag_id = AC 52 54 07 05`, `fpga_results_received` tăng, `fpga_result_valid = 1`, `fpga_last_error = 0`; `fpga_water_percent_x100 / 100` là %. `camera_gate_reason` cho biết điều kiện cho phép/chặn camera; `camera_fallback_active` cho biết đang dự phòng. `fpga_decisions_sent` chỉ đếm modem/broker xác nhận publish, không xác nhận nơi nhận đã lưu.

YJ-FS phải gửi `05 03 00 00 00 01 85 8E`. Nếu `wind_diag_ready=1`, phase=4, TX status=0, RX count=0 và status TIMEOUT thì UART chưa nhận byte trong cửa sổ chờ. Cảm biến đứng yên vẫn phải trả giá trị 0 hợp lệ; chưa xác nhận nguyên nhân vật lý timeout trên kit.
