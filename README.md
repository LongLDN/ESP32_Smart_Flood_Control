# Hệ thống cảnh báo ngập và điều khiển bơm — ESP32

Dự án đọc cảm biến mưa, cảm biến mực nước và công tắc phao; cảnh báo bằng LED/còi, điều khiển rào chắn bằng servo và bơm qua relay. Ứng dụng Blynk hiển thị số đo và cho phép chuyển chế độ tự động/thủ công. Chương trình dùng **Blynk.Edgent** để cấu hình Wi-Fi và tài khoản Blynk khi chạy lần đầu.

## 1. Phần cứng và kết nối

| Thiết bị | GPIO ESP32 | Ghi chú |
| --- | ---: | --- |
| Cảm biến mưa, ngõ analog AO | 35 | Đọc giá trị analog |
| Cảm biến mực nước, ngõ analog | 34 | Đọc giá trị analog |
| Công tắc phao | 33 | `INPUT_PULLUP`; mã xem HIGH là mức kích hoạt bơm |
| Relay bơm, chân IN | 13 | **Active LOW**: LOW bật, HIGH tắt |
| Servo rào chắn, chân signal | 18 | Góc 0° hoặc 130° |
| LED vàng / đỏ / xanh | 25 / 26 / 27 | Mưa lớn / ngập / bình thường |
| Buzzer | 32 | Âm báo |
| Nút rào chắn / nút bơm | 16 / 17 | `INPUT_PULLUP`; nối nút từ GPIO đến GND |
| Nút BOOT trên board | 0 | Đặt lại cấu hình Edgent |

Nối GND chung cho các module tín hiệu và dùng điện trở hạn dòng cho LED. Đảm bảo ngõ analog vào ESP32 không vượt 3,3 V. Servo và bơm cần nguồn phù hợp riêng; **không lấy nguồn bơm từ chân GPIO**. Kiểm tra mức HIGH/LOW thực tế của phao và loại relay trước khi đấu bơm thật.

## 2. Chuẩn bị phần mềm

1. Cài Arduino IDE và board package **ESP32 by Espressif Systems**.
2. Trong Library Manager, cài **Blynk** (bản hỗ trợ Blynk IoT/Edgent) và **ESP32Servo**. Các thư viện `WiFi.h`, `Preferences.h`, `WebServer.h`, `DNSServer.h`, `Update.h`, `HTTPClient.h` đi cùng ESP32 board package.
3. Tạo tài khoản Blynk IoT và chuẩn bị mạng Wi-Fi **2,4 GHz**.
4. Tạo Blynk Template. Trong `Edgent_ESP32.ino`, `BLYNK_TEMPLATE_ID` hiện là `TMPL6-v9U3j-_`, tên là `IoT102 Project`. Nếu dùng template của bạn, thay **cả hai macro** này bằng ID và tên tương ứng. Không tự thêm `BLYNK_AUTH_TOKEN`: Edgent cấp và lưu token khi cấu hình thiết bị.

## 3. Cách nạp và chạy

1. Tải repository qua **Code → Download ZIP**, hoặc chạy:
   ```bash
   git clone https://github.com/LongLDN/IOT102_Group03_FinalReport_Code.git
   ```
2. Mở `Edgent_ESP32/Edgent_ESP32.ino` bằng Arduino IDE. Giữ các file `.h` trong **cùng thư mục** với sketch.
3. Chọn **Tools → Board → ESP32 Dev Module** nếu dùng board ESP32 DevKit, rồi chọn đúng **Tools → Port**. Nếu dùng board khác, chọn đúng loại board và kiểm tra định nghĩa trong `Settings.h`.
4. Ngắt tải bơm để thử an toàn. Nhấn **Verify** rồi **Upload**. Nếu IDE kẹt ở `Connecting...`, giữ nút BOOT đến khi bắt đầu nạp rồi thả.
5. Mở **Serial Monitor** ở **115200 baud**. Khi chưa có cấu hình, ESP32 tạo access point có tên bắt đầu bằng `Blynk`. Dùng ứng dụng Blynk để thêm thiết bị theo luồng Edgent, chọn template, nhập mạng Wi-Fi. Nếu cấu hình qua trình duyệt, kết nối Wi-Fi AP của ESP32 và mở `http://192.168.4.1/` để nhập thông tin Wi-Fi/token thiết bị.
6. Sau khi ESP32 kết nối Wi-Fi/Blynk Cloud, Serial Monitor in `Rain`, `Water`, `Float`, `Mode`; dashboard hiển thị dữ liệu. Lần khởi động tiếp theo, Edgent dùng cấu hình đã lưu.

Muốn cấu hình lại, giữ nút BOOT/GPIO0 khoảng **10 giây** (theo `Settings.h`/`ResetButton.h`) để xóa thông tin Edgent và tiến hành thêm thiết bị lại. Tránh giữ BOOT lúc vừa bật nguồn khi không cần vào bootloader.

## 4. Datastream và sự kiện Blynk

Tạo Virtual Pin datastream kiểu số (Integer) đúng các chân sau. Các widget hiển thị dùng V0–V2, các widget **Switch** dùng V3–V5.

| Pin | Ý nghĩa | Miền giá trị |
| --- | --- | --- |
| V0 | Mực nước quy đổi | 0–100 (%) |
| V1 | Mức mưa quy đổi | 0–100 (%) |
| V2 | Trạng thái phao | 0/1 |
| V3 | Điều khiển/trạng thái bơm | 0 tắt, 1 bật |
| V4 | Điều khiển/trạng thái rào chắn | 0 góc 0°, 1 góc 130° |
| V5 | Chế độ | 0 manual, 1 auto |

Tạo các Blynk **Events** với mã chính xác `flood`, `heavy_rain`, `pump_on`, `pump_off` để nhận thông báo tương ứng. Nếu template cũ không thuộc quyền tài khoản của bạn, tạo template mới và cập nhật ID/tên ở đầu sketch.

## 5. Nguyên lý hoạt động

- Chế độ ban đầu là **auto**. `waterPercent` được tính từ `analogRead(GPIO34)` bằng `map(raw, 0, 2000, 0, 100)`; `rainPercent` tính từ `4095 - analogRead(GPIO35)` rồi quy đổi 0–100. Đây là thang đo của mô hình, cần hiệu chuẩn theo cảm biến thực tế.
- Khi nước **> 60%**, còi kêu 5 nhịp, ghi event `flood`, LED đỏ nhấp nháy và servo mở rào đến 130° nếu không bị giữ điều khiển thủ công. Cảnh báo ngập được đặt lại khi nước **< 50%**.
- Khi mưa **> 60%**, còi kêu 3 nhịp, ghi event `heavy_rain`; LED vàng nhấp nháy nếu không có cảnh báo ngập ưu tiên. Trạng thái mưa lớn được đặt lại khi **< 50%**. Bình thường LED xanh sáng, rào về 0°.
- Khi phao đọc **HIGH liên tục hơn 3 giây**, auto bật bơm (`pump_on`); khi về LOW, auto tắt bơm (`pump_off`). Relay trong mã bật bằng LOW.
- Nhấn nút GPIO16/17 hoặc điều khiển V3/V4 sẽ chuyển sang **manual** và đảo/đặt trạng thái rào hoặc bơm. Gạt V5 lên 1 để trở lại auto; V5 = 0 chọn manual.

**Giới hạn mã hiện tại:** V0/V1/V2/V5 được gửi và Serial được in ở **mỗi vòng `loop()`**, chưa có giới hạn tần suất; nếu dashboard cập nhật quá dày, nên chuyển việc gửi dữ liệu sang `BlynkTimer`. Hàm còi, servo và xử lý nút dùng `delay()`, có thể làm hệ thống phản hồi chậm trong lúc chờ. Nút vật lý chưa có cơ chế chống dội hoàn chỉnh.

## 6. Vai trò các file

| File | Chức năng |
| --- | --- |
| `Edgent_ESP32.ino` | Chân kết nối, cảm biến, logic auto/manual, Blynk V0–V5 |
| `BlynkEdgent.h`, `BlynkState.h` | Vòng đời kết nối và các trạng thái Edgent |
| `ConfigMode.h`, `ConfigStore.h` | Trang cấu hình Wi-Fi/Blynk, lưu cấu hình |
| `Settings.h`, `ResetButton.h`, `Indicator.h` | Cấu hình board, reset, đèn trạng thái Edgent |
| `OTA.h`, `Console.h` | Cập nhật OTA, console chẩn đoán |

## 7. Lỗi thường gặp

| Hiện tượng | Hướng xử lý |
| --- | --- |
| Thiếu `BlynkSimpleEsp32_SSL.h` | Cài/cập nhật thư viện Blynk hỗ trợ Edgent. |
| Thiếu `ESP32Servo.h` | Cài ESP32Servo trong Library Manager. |
| Không hiện cổng COM | Dùng cáp USB có truyền dữ liệu, kiểm tra driver USB-serial và chọn đúng Port. |
| Upload kẹt `Connecting...` | Nhấn giữ BOOT khi bắt đầu nạp, thả khi upload tiến hành. |
| Không vào Wi-Fi/Blynk | Kiểm tra Wi-Fi 2,4 GHz, mật khẩu, template ID/token và Serial Monitor 115200; cấu hình lại nếu cần. |
| Giá trị mưa/nước không hợp lý | Kiểm tra dây AO/GND, điện áp, chiều tín hiệu và hiệu chỉnh mốc `2000` trong `loop()`. |
| Bơm chạy ngược ý định | Xem `Float: 0/1` trên Serial Monitor; kiểm tra NO/NC của phao và relay active LOW. |
| Đèn trạng thái Edgent không sáng | `USE_ESP32_DEV_MODULE` trong `Settings.h` chưa cấu hình LED Edgent; LED ứng dụng vẫn ở GPIO25/26/27. |

> Chỉ dùng phần cứng với nguồn và cách điện thích hợp. Mô hình này chưa được đánh giá để sử dụng như hệ thống cảnh báo ngập an toàn thực tế.
