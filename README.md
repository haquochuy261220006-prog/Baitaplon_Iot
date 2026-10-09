# Hệ thống giám sát nhiệt độ – ESP32 (Wokwi) + Verilog + Python + Web

Đồ án gồm 4 thành phần chạy trên cùng một luồng dữ liệu:

| Thành phần | Công cụ | Vai trò |
|---|---|---|
| Mạch mô phỏng + firmware | Wokwi trong VS Code, PlatformIO | ESP32 đọc cảm biến DHT22, điều khiển LED, gửi dữ liệu qua MQTT |
| Giao diện web | HTML/JS | Hiển thị nhiệt độ, độ ẩm, biểu đồ và cảnh báo theo thời gian thực |
| Python | paho-mqtt | Ghi log CSV, xuất dữ liệu cho Verilog, đối chiếu kết quả mô phỏng |
| Verilog | Icarus Verilog, GTKWave | Mô tả phần cứng của logic cảnh báo (máy trạng thái + hysteresis) |

```
ESP32 + DHT22 (Wokwi) --MQTT--> broker.hivemq.com --+--> web/index.html   (hiển thị)
                                                    +--> python/logger.py --> temps.hex --> Verilog --> verify.py
```

## Chạy nhanh (khuyên dùng)

1. Cài Python (tích *Add Python to PATH*). Muốn chạy phần Verilog thì cài thêm Icarus Verilog: `winget install --id Icarus.Verilog`.
2. Giải nén, **nhấp đúp `chay.bat`**.
3. Chọn `1` (demo bằng Python, không cần Wokwi) hoặc `2` (dùng Wokwi thật).

`chay.bat` tự cài paho-mqtt, tự tạo topic riêng và điền vào mọi file, mở trang web, ghi log 80 giây, rồi chạy mô phỏng Verilog và đối chiếu với Python. Với lựa chọn `2`, hãy mở dự án trong VS Code, Build và Start Simulator (mục 4, Bước 1) trước khi chọn.

## 1. Cấu trúc thư mục

```
giam-sat-nhiet-do/
├── README.md
├── chay.bat              nhấp đúp để chạy toàn bộ
├── platformio.ini        cấu hình build ESP32 và thư viện
├── wokwi.toml            trỏ tới file firmware cho Wokwi
├── diagram.json          sơ đồ mạch: ESP32, DHT22, LED đỏ, LED xanh
├── src/main.cpp          firmware ESP32
├── web/index.html        giao diện web
├── python/
│   ├── setup.py          tự tạo topic riêng
│   ├── sim_esp32.py      giả lập ESP32 bằng Python
│   ├── logger.py         nhận MQTT, ghi CSV, xuất temps.hex
│   ├── verify.py         chạy Verilog và so với mô hình Python
│   └── requirements.txt
└── verilog/
    ├── temp_monitor.v    mô-đun giám sát nhiệt độ
    ├── tb_temp_monitor.v testbench
    └── temps.hex         dữ liệu mẫu (0.1 °C, hex)
```

## 2. Cài đặt phần mềm

Cần Internet vì dữ liệu đi qua broker MQTT công cộng.

1. **Visual Studio Code**: https://code.visualstudio.com
2. Trong VS Code, cài các extension:
   - **PlatformIO IDE**
   - **Wokwi Simulator**
   - **Verilog-HDL/SystemVerilog** (tùy chọn, để tô màu cú pháp)
   - **Live Server** (tùy chọn, để mở web)
3. **Python 3.9 trở lên**: https://www.python.org (khi cài trên Windows, tích chọn *Add Python to PATH*).
4. **Icarus Verilog** và **GTKWave**:
   - Windows: tải bản cài đặt tại https://bleyer.org/icarus (có kèm GTKWave), tích chọn thêm vào PATH.
   - Ubuntu/Debian: `sudo apt install iverilog gtkwave`
   - macOS: `brew install icarus-verilog gtkwave`
   - Kiểm tra bằng lệnh `iverilog -V`.
5. Cài thư viện Python:
   ```
   cd python
   pip install -r requirements.txt
   ```
6. Lấy license Wokwi (miễn phí): trong VS Code nhấn `F1`, chọn **Wokwi: Request a Free License**, đăng nhập theo hướng dẫn.

## 3. Cấu hình trước khi chạy (bắt buộc)

Broker công cộng dùng chung cho mọi người, nên cần một topic riêng. Đổi chuỗi `CHANGE_ME` thành tên của bạn (ví dụ `hung123`) ở **cả 3 file**, sao cho giống hệt nhau:

- `src/main.cpp`: dòng `const char* TOPIC = ...`
- `web/index.html`: dòng `const TOPIC = ...`
- `python/logger.py`: dòng `TOPIC = ...`

Ví dụ: `tn/nhietdo/hung123/data`

## 4. Chạy hệ thống

### Bước 1: Build và chạy mô phỏng Wokwi
1. Mở thư mục `giam-sat-nhiet-do` bằng VS Code (*File > Open Folder*).
2. Chờ PlatformIO tải thư viện lần đầu, rồi bấm **Build** (biểu tượng dấu tích ở thanh dưới cùng, hoặc chạy `pio run`). Bước này tạo `.pio/build/esp32dev/firmware.bin`.
3. Nhấn `F1`, chọn **Wokwi: Start Simulator**.
4. Nhấp vào cảm biến DHT22 trong mô phỏng và kéo thanh nhiệt độ. Serial Monitor sẽ in dòng JSON mỗi 2 giây. Trên 35 °C thì LED đỏ sáng, ngược lại LED xanh sáng.

### Bước 2: Mở giao diện web
Mở `web/index.html` bằng trình duyệt (nhấp đúp file, hoặc dùng Live Server). Khi thấy "Đã kết nối broker" và số nhiệt độ cập nhật là thành công. Trang có sẵn chế độ sáng/tối theo hệ thống.

### Bước 3: Ghi log bằng Python
```
cd python
python logger.py
```
Dữ liệu được in ra màn hình và ghi vào `python/data/log.csv`. Bấm **Ctrl+C** để dừng, chương trình sẽ ghi các mẫu vào `verilog/temps.hex`.

### Bước 4: Chạy mô phỏng Verilog và đối chiếu
```
cd python
python verify.py
```
Script biên dịch Verilog bằng `iverilog`, chạy `vvp`, rồi so trạng thái của mạch với mô hình Python. Cuối bảng sẽ có dòng `PASS` hoặc `FAIL`. Có thể chạy ngay với dữ liệu mẫu có sẵn, không cần chạy Bước 3 trước.

Xem dạng sóng:
```
cd verilog
gtkwave wave.vcd
```
Trong GTKWave, chọn `tb_temp_monitor > dut` rồi kéo các tín hiệu `state`, `temp_x10`, `led_red`, `t_max`, `t_min` sang khung sóng.

## 5. Nguyên lý của mô-đun Verilog

Nhiệt độ vào tính theo 0.1 °C (352 nghĩa là 35.2 °C). Máy trạng thái có 3 mức:

| Trạng thái | Điều kiện vào | Điều kiện thoát |
|---|---|---|
| Bình thường (LED xanh) | dưới 30.0 °C | |
| Chú ý (LED vàng) | từ 30.0 °C | xuống dưới 29.0 °C |
| Cảnh báo (LED đỏ) | từ 35.0 °C | xuống dưới 34.0 °C |

Ngưỡng thoát thấp hơn ngưỡng vào 1 °C (hysteresis) để tránh đèn nhấp nháy khi nhiệt độ dao động quanh ngưỡng. Mô-đun cũng ghi lại nhiệt độ cao nhất và thấp nhất. Các tham số `WARN`, `ALARM`, `HYST` chỉnh được khi khai báo mô-đun.

Lưu ý: mô-đun Verilog chạy trong mô phỏng, không nạp lên ESP32. Firmware ESP32 chỉ dùng một ngưỡng 35 °C. Nhiệt độ âm bị cắt về 0 khi xuất ra `temps.hex`.

## 6. Xử lý lỗi thường gặp

| Hiện tượng | Nguyên nhân và cách xử lý |
|---|---|
| Web hiện "Chờ dữ liệu" mãi | Topic ở web và `main.cpp` không khớp, hoặc Wokwi chưa chạy. Kiểm tra Serial Monitor có in JSON và "MQTT OK" chưa. |
| Web hiện "Mất kết nối" | Mạng chặn cổng 8884 (WebSocket). Thử mạng khác hoặc điểm phát 4G. |
| Wokwi báo thiếu firmware | Chưa Build. Chạy Build ở Bước 1 rồi Start Simulator lại. |
| Wokwi báo cần license | Chạy *Wokwi: Request a Free License* (mục 2, bước 6). |
| Serial chỉ in dấu chấm | Chưa vào được WiFi mô phỏng. Dừng và chạy lại mô phỏng, kiểm tra Internet. |
| `verify.py` báo chưa cài iverilog | Cài Icarus Verilog và thêm vào PATH, mở lại terminal. |
| `logger.py` lỗi `CallbackAPIVersion` | paho-mqtt cũ. Chạy `pip install -U paho-mqtt`. |
| Trên Windows không có lệnh `python` | Dùng `py` thay cho `python`. |
| `temps.hex` rỗng hoặc verify báo lỗi | Bước 3 chưa nhận được mẫu nào trước khi bấm Ctrl+C. Chạy lại khi Wokwi đang gửi dữ liệu. |
