# VortexETE FPGA — ACG525, camera WATER_FIRST

## Giới thiệu dự án

FPGA ACG525 nhận lệnh từ MCU SN32F407 qua SPI, đánh thức/cấu hình camera OV5640, thu ảnh RGB565 VGA 640×480 và chuyển thành ảnh xám 256×256. Mạng INT8 nhận diện nước quét toàn ảnh trước; nếu có pixel được dự đoán là nước, mạng INT8 nhận diện mặt đường tiếp tục tạo ROI rồi tính độ phủ nước trong ROI.

FPGA giữ kết quả để MCU đọc, kiểm CRC và ACK. Kết quả gồm frame ID, số pixel nước trong ROI, số pixel ROI, số pixel nước toàn ảnh, độ phủ và mức cảnh báo. Đây là project camera thật, không lấy ảnh ROM cố định làm đầu vào vận hành. Gói này không chứa MCU hay website; ghép với MCU V14/schema `0x43`.

| Độ phủ nước trong ROI | Mức |
|---|---|
| Dưới 5% | NORMAL = 0 |
| Từ 5% đến dưới 25% | WARNING = 1 |
| Từ 25% | CRITICAL = 2 |

Khung hợp lệ có **0 pixel nước toàn ảnh** bỏ lượt suy luận ROI, xóa ROI cũ và trả NORMAL với độ phủ 0; không chia 0/0. Khung có nước nhưng ROI rỗng/quá nhỏ hoặc ảnh không đạt kiểm tra tương phản trả lỗi/UNKNOWN = 3. Quyết định dựa trên đầu ra mô hình, chưa phải số đo độ sâu nước hoặc bảo đảm độ chính xác thực địa. Hai mô hình nhỏ được phát triển từ tập ảnh hạn chế; không tự tạo confidence hay số cm từ độ phủ.

### Bố trí thư mục

| Đường dẫn | Nội dung |
|---|---|
| `ACG525_VORTEX_CAMERA_AI_COMPACT.gprj` | Project Gowin, top `vortex_camera_ai_compact` |
| `src/compact/` | Top, ràng buộc chân `.cst`, thời gian `.sdc` |
| `src/camera_init/` | Cấu hình OV5640 bằng SCCB/I²C |
| `src/gowin_pll/`, `src/reset_release.v` | Clock và nhả reset |
| `src/vision/` | Thu/resize ảnh, hai mạng INT8, ROI, quyết định, RAM và SPI |
| `src/no_ssram.gsc` | Ràng buộc tổng hợp đang dùng |
| `impl/*_process_config.json` | Tùy chọn triển khai cần giữ; không phải kết quả build |
| `tests/` | Testbench, vector nhỏ và kiểm tra nguồn gốc |

Mở `.gprj` bằng Gowin. Giữ device **GW5A-LV25UG324ES** đúng file project gốc và đối chiếu chip trên kit; không tự chọn sang biến thể khác. Clock vào 50 MHz, camera XCLK 25 MHz. Synthesize → Place & Route → kiểm tài nguyên/timing → nạp `.fs` vừa tạo. VS Code dùng để chỉnh RTL, không thay công cụ tổng hợp/nạp Gowin.

## Hướng dẫn chỉnh code

### 1. Chọn file theo chức năng

| Muốn chỉnh | File cần xem |
|---|---|
| Kết nối module, kích thước camera/ảnh, điều khiển ngủ | `src/compact/vortex_camera_ai_compact.v` |
| Chân camera, SPI, LED và chuẩn điện áp I/O | `src/compact/vortex_compact.cst` |
| Clock và quan hệ giữa các miền clock | `src/compact/vortex_compact.sdc`, `src/gowin_pll/gowin_pll.v` |
| Thanh ghi camera RGB565/VGA | `src/camera_init/ov5640_init_table_rgb.v` |
| Luồng cấu hình camera | `src/camera_init/camera_init.v`, `i2c_control/` |
| Đổi ảnh DVP sang xám/resize và bắt trọn khung | `src/vision/gray_resize_snapshot.v`, `vision_snapshot.v` |
| Kết nối pipeline và truyền tham số | `src/vision/vision_system.v` |
| WATER_FIRST, chạy mạng nước/ROI, lưu ảnh/mask | `src/vision/vision_engine.v` |
| Trọng số và tính toán mạng nước | `src/vision/water_int8_core.v` |
| Trọng số và tính toán mạng ROI mặt đường | `src/vision/road_int8_core.v` |
| Trình tự WAKE/capture/infer/hold/ACK, kiểm ảnh và mức | `src/vision/vision_control.v` |
| Đóng gói thanh ghi kết quả và CRC | `src/vision/vision_spi.v` |
| Nhận lệnh/đọc ghi SPI và lọc SCK | `src/vision/water_spi_transport.v`, `spi_sck_filter.v` |
| RAM hai miền clock, ánh xạ SDPB | `src/vision/vortex_bram8.v` |

### 2. Chỉnh ngưỡng và thời gian

`vision_system.v` truyền `WARNING_PERCENT=5` và `CRITICAL_PERCENT=25` cho cả bộ quyết định lẫn bộ đóng gói SPI. Khi đổi ngưỡng, sửa tại điểm truyền tham số chung này hoặc truyền tham số từ top; không sửa riêng một module con. Ví dụ ngưỡng mới 10%/30%:

```verilog
parameter integer WARMUP_FRAMES=10, WARNING_PERCENT=10, CRITICAL_PERCENT=30,
```

Đồng thời sửa MCU `firmware/fpga_spi.c`: hàm `FPGA_Level()` và điều kiện kiểm hai trường ngưỡng. MCU hiện bắt buộc 5/25 nên thay riêng FPGA sẽ bị từ chối kết quả. Cập nhật giá trị kỳ vọng của testbench trước khi chạy lại.

| Tham số | Nơi chỉnh | Mặc định |
|---|---|---|
| `WARMUP_FRAMES` | `vision_system.v`, truyền xuống snapshot | 10 khung ổn định trước capture |
| `CONFIG_TIMEOUT` | `vision_system.v`, truyền xuống control | `CLK_HZ*12` chu kỳ, tức 12 giây ở 50 MHz |
| `CAPTURE_TIMEOUT` | `vision_system.v`, truyền xuống control | `CLK_HZ*12` chu kỳ |
| `MIN_CONTRAST` | `vision_control.v` | 8 mức xám |
| `MIN_ROI_PIXELS` | `vision_control.v` | `(WIDTH*HEIGHT+199)/200` = 328 với 256×256 |
| `CAMERA_SLEEP_ENABLE` | Top `vortex_camera_ai_compact.v` | 1 |

Điều kiện **ít mưa/không mưa và nước cao** cùng chu kỳ yêu cầu chụp thuộc MCU `app_config.h`, không thuộc bộ quyết định mức ảnh của FPGA. Camera ngủ bằng PWDN/reset cấu hình; XCLK vẫn được phát trong top hiện tại, không phải cắt hoàn toàn nguồn camera.

### 3. Chỉnh kích thước, camera hoặc mô hình

Top đang đặt camera 640×480, ảnh xử lý 256×256. Nếu đổi, phải sửa đồng bộ bảng thanh ghi camera, tham số camera/top/snapshot/engine, dung lượng và địa chỉ RAM, trường kích thước SPI và bộ kiểm MCU. MCU hiện cố định 256×256 và ROI tối thiểu 328; chỉ sửa một tham số RTL chưa đủ.

Trọng số INT8 nằm trong hai core RTL, không cần file ảnh/ROI ROM khi build. Thay mô hình cần xuất đúng đặc trưng, thứ tự trọng số, lượng tử hóa và kiểm bằng vector tham chiếu trước khi tổng hợp. Gói gọn không kèm bộ huấn luyện/teacher ONNX; các dữ liệu đó vẫn thuộc gói FPGA nguồn trước đây. Không sửa vài trọng số rồi coi kết quả đánh giá cũ còn áp dụng.

### 4. Chỉnh SPI và chân kết nối

SPI dùng Mode 0, MSB trước, CS_n tích cực LOW; thiết kế hiện đặt giới hạn SCK không quá 100 kHz. Giữ setup CS/MOSI và hold CS ít nhất 2 µs theo giao tiếp MCU đang dùng. Chân trong `.cst`:

| Tín hiệu | FPGA | Đầu P6 | MCU |
|---|---|---|---|
| SCK | L13 | 22 | P1.0 |
| CS_n | H18 | 24 | P1.8 |
| MOSI | K18 | 26 | P1.2 |
| MISO | L18 | 28 | P1.1 |

Các chân camera cũng nằm trong cùng `.cst`; PCLK hiện K15/P6.15, RESET_N H14/P6.14. Đổi chân phải sửa ràng buộc và dây cùng lúc, đồng thời kiểm bank voltage. GND hai kit nối chung; chuyển mức phải phù hợp điện áp I/O thực tế.

Giao tiếp hiện có ID `AC 52 54 07 05`, capability `7A`, schema `43`, vùng đọc 60 byte. Read: `03 AH AL dummy` rồi mới nhận dữ liệu; write: `02 AH AL value`. START/ACK/CLEAR ghi tại `0x08` với giá trị 1/2/3. Payload nằm ở `0x20..0x39`, CRC16 ở `0x3A..0x3B` (seed FFFF, polynomial 1021, tính trên payload 26 byte). Đổi bố cục hay CRC phải sửa cả MCU; giữ kết quả ổn định đến ACK.

Các miền PCLK camera và clock logic dùng RAM hai clock, đồng bộ và handshake. Giữ cơ chế này khi sửa; không nối bus nhiều bit trực tiếp qua miền clock. `.sdc` hiện đặt PCLK dự kiến 10 ns và nhóm bất đồng bộ; đây chưa phải số đo PCLK/input delay thực tế. Cần kiểm timing theo camera và bo sau triển khai.

### 5. Kiểm tra sau khi chỉnh

Máy có Python 3 và Icarus Verilog (`iverilog`, `vvp` trong PATH):

```bash
python3 tests/run_checks.py
```

Bộ kiểm tra gồm WATER_FIRST, control, capture qua clock độc lập và SPI, resize VGA, SCCB và kết nối top. `python3 tests/check_package.py` kiểm hash và đường dẫn để xác nhận nguyên vẹn bản phát hành; khi chủ động sửa nguồn, hash thay đổi là bình thường. Sau khi sửa, có thể chạy `run_water_first_checks.py` hoặc từng testbench trực tiếp để kiểm logic rồi cập nhật dấu kiểm của bản phát hành mới.

Mô phỏng không thay thế Gowin Synthesize/Place & Route, kiểm BSRAM/timing và thử camera thật. Không có bitstream `.fs` dựng sẵn trong gói. LED theo bit logic top: LED0 PLL lock, LED1 BUSY, LED2 READY, LED3 nhịp PCLK; trạng thái sáng vật lý phụ thuộc cực tính LED trên bo.
