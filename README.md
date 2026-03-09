# 🚁 Hệ thống Giám sát Giao thông Thông minh bằng Quadcopter (IoT & AI Integration)

Dự án này là một hệ thống toàn diện (End-to-End) kết hợp giữa Internet of Things (IoT), viễn thông vô tuyến (LoRa), và Trí tuệ Nhân tạo (AI). Hệ thống cho phép thu thập telemetry từ Quadcopter, truyền phát video thời gian thực, xử lý phân tích lưu lượng giao thông và trực quan hóa dữ liệu đa nền tảng (Desktop & Mobile).



---

## 🏗 Kiến trúc Hệ thống & Repository

Hệ thống được thiết kế theo kiến trúc phân tán (Distributed Architecture), bao gồm 6 module chính. Bạn có thể truy cập mã nguồn của từng thành phần qua các link dưới đây:

* 🛸 **1. UAV Telemetry Node:** https://github.com/phxbac909/ESP32-SAT/tree/Communication
* 📷 **2. UAV Video Node:** 
* 📟 **3. Ground Station Gateway:** https://github.com/phxbac909/ESP32-GS/tree/Communication
* 🖥 **4. JavaFX Ground Control Station:** https://github.com/phxbac909/SatelliteClientDesktop
* 🧠 **5. AI Vision Service (DJL/YOLO):** https://github.com/phxbac909/CameraAI
* ☁️ **6. Spring Boot Backend Server:** https://github.com/phxbac909/SattelliteServer
* 📱 **7. iOS Client App:** [[Link GitHub: Điền link tại đây]](https://github.com/phxbac909/client-iOS)

---

## 📡 Chi tiết Cụm phần cứng (Hardware Nodes)

### 1. Cụm thiết bị bay (UAV Edge Nodes)
Gắn trực tiếp trên Quadcopter, làm nhiệm vụ thu thập dữ liệu và hình ảnh.

* **Telemetry Node (ESP32):**
  * **Cảm biến:** * `MPU6050`: Thu thập gia tốc và con quay hồi chuyển để tính toán tư thế (Pitch, Roll, Yaw).
    * `BMP280`: Đo áp suất khí quyển để ước lượng độ cao tương đối.
    * `GPS Neo-6M`: Thu thập tọa độ kinh độ, vĩ độ.
  * **Truyền tin:** Đóng gói dữ liệu thành chuỗi JSON hoặc Byte array, gửi qua module **LoRa RA-01H (433MHz/915MHz)** để tối ưu khoảng cách truyền ngoài tầm nhìn (NLOS).

* **Video Streaming Node (ESP32-CAM):**
  * Sử dụng camera OV2640.
  * Nén ảnh dưới định dạng JPEG và stream liên tục về trạm mặt đất thông qua giao thức **WiFi UDP** để giảm tối đa độ trễ (Low Latency).



### 2. Trạm thu mặt đất (Ground Station Gateway)
* **Phần cứng:** 1x ESP32 + 1x LoRa RA-01H.
* **Chức năng:** Đóng vai trò là cầu nối (Bridge). Lắng nghe các gói tin LoRa từ Quadcopter, kiểm tra tính toàn vẹn (Checksum/CRC) và chuyển tiếp toàn bộ dữ liệu vào máy tính (PC) thông qua giao tiếp **Serial (UART)** với baudrate cao (VD: 115200 hoặc 921600).

---

## 💻 Chi tiết Cụm phần mềm (Software Nodes)

### 3. Phần mềm điều khiển mặt đất (JavaFX GCS)
Đây là trung tâm điều phối dữ liệu tại mặt đất, chạy trên PC.
* **Serial Communication:** Dùng thư viện `JSerialComm` hoặc `RxTx` để đọc luồng dữ liệu từ Ground Station.
* **Giao diện trực quan (UI/UX):**
  * **3D Attitude:** Sử dụng `JavaFX 3D` (SubScene, PhongMaterial) để tạo một mô hình 3D mô phỏng thời gian thực độ nghiêng của Drone.
  * **Telemetry Charts:** Vẽ biểu đồ biến thiên độ cao và tốc độ theo thời gian thực.
  * **GPS Mapping:** Tích hợp WebEngine hoặc thư viện bản đồ (như Leaflet/Google Maps API) để vẽ quỹ đạo bay.
* **Xử lý Video & Điều phối:** Nhận luồng UDP từ ESP32-CAM, hiển thị lên UI, đồng thời forward các frame ảnh (gửi qua HTTP/gRPC) sang AI Service.



### 4. Dịch vụ phân tích hình ảnh (AI Vision Service)
* **Công nghệ:** Java, **Deep Java Library (DJL)**.
* **Model AI:** Khai báo và load pre-trained model **YOLO** (YOLOv5 hoặc YOLOv8).
* **Luồng xử lý:**
  1. Nhận frame ảnh từ JavaFX.
  2. Inference (Suy luận) để phát hiện ô tô, xe máy, người đi bộ...
  3. Áp dụng thuật toán Tracking (như DeepSORT hoặc ByteTrack) để gán ID cho từng phương tiện, đảm bảo không đếm trùng.
  4. Trả về kết quả (Bounding boxes, tổng số xe, mật độ giao thông hiện tại) cho phần mềm JavaFX.



### 5. Máy chủ dữ liệu (Spring Boot Backend)
* **Công nghệ:** Spring Boot, Spring Data JPA, Hibernate, WebSockets.
* **Chức năng:**
  * Cung cấp **RESTful API** để JavaFX đẩy dữ liệu Telemetry và kết quả AI lên hệ thống.
  * Lưu trữ dữ liệu lịch sử bay vào cơ sở dữ liệu (PostgreSQL/MySQL/MongoDB).
  * Khởi tạo **WebSocket Server** (STOMP protocol) để broadcast (phát sóng) luồng dữ liệu thời gian thực tới tất cả các Client đang kết nối.

### 6. Ứng dụng giám sát di động (iOS Client)
* **Công nghệ:** Swift, SwiftUI, SceneKit (cho 3D), MapKit (cho bản đồ).
* **Chức năng:**
  * Kết nối tới Spring Boot Server thông qua **WebSocket** để nhận data stream với độ trễ tính bằng mili-giây.
  * Dùng **SceneKit** render lại tư thế bay 3D của Quadcopter trên điện thoại.
  * Dùng **MapKit** cắm pin vị trí GPS hiện tại.
  * Gọi các API REST để truy xuất lịch sử chuyến bay và biểu đồ thống kê mật độ giao thông.

---

## 🔄 Luồng dữ liệu (Data Flow)

1. **UAV ➔ Ground Station:** Dữ liệu cảm biến đi qua sóng LoRa; Video đi qua sóng WiFi (UDP).
2. **Ground Station ➔ JavaFX PC:** Dữ liệu LoRa được đẩy qua cáp USB (Serial).
3. **JavaFX PC ➔ AI Service:** Frame hình ảnh được trích xuất và gửi đi phân tích. 
4. **AI Service ➔ JavaFX PC:** Trả về Bounding box và Mật độ giao thông.
5. **JavaFX PC ➔ Spring Boot:** Gộp Telemetry + AI Data, gửi POST request lên Server.
6. **Spring Boot ➔ iOS Client:** Push dữ liệu mới nhất qua WebSocket để cập nhật giao diện điện thoại.

---

---
*Dự án được xây dựng với mục tiêu chứng minh khả năng tính toán biên (Edge Computing) và giám sát diện rộng bằng thiết bị bay không người lái kết hợp AI.*
