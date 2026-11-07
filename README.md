# BTL_AI_va_VXL — Hệ thống điều khiển động cơ bằng cử chỉ kết hợp nhận dạng khuôn mặt

> **IP Camera (RTSP) + Jetson Nano B01 + STM32 Bluepill**
> Bài tập lớn môn **Trí tuệ nhân tạo (AI)** và **Vi xử lý (VXL)**

| | |
|---|---|
| **Ngành** | Hệ thống Nhúng thông minh & IoT |
| **Đơn vị** | Trường Điện – Điện tử, Đại học Bách khoa Hà Nội (HUST) |
| **Repository** | `BTL_AI_va_VXL` |

---

## Mục lục

1. [Giới thiệu](#1-giới-thiệu)
2. [Kiến trúc và quy trình hệ thống](#2-kiến-trúc-và-quy-trình-hệ-thống)
3. [Bảng ánh xạ Class – Byte SPI – Tác vụ động cơ](#3-bảng-ánh-xạ-class--byte-spi--tác-vụ-động-cơ)
4. [Cấu trúc thư mục dự án](#4-cấu-trúc-thư-mục-dự-án)
5. [Phần cứng và sơ đồ đấu nối](#5-phần-cứng-và-sơ-đồ-đấu-nối)
6. [Cài đặt khối AI trên Jetson Nano](#6-cài-đặt-khối-ai-trên-jetson-nano)
7. [Chuẩn bị dữ liệu và huấn luyện](#7-chuẩn-bị-dữ-liệu-và-huấn-luyện)
8. [Cài đặt khối vi xử lý STM32 Bluepill](#8-cài-đặt-khối-vi-xử-lý-stm32-bluepill)
9. [Chạy hệ thống](#9-chạy-hệ-thống)
10. [Tham số cấu hình](#10-tham-số-cấu-hình)
11. [Xử lý sự cố](#11-xử-lý-sự-cố)
12. [Giới hạn đã biết và hướng phát triển](#12-giới-hạn-đã-biết-và-hướng-phát-triển)

---

## 1. Giới thiệu

Hệ thống cho phép **điều khiển động cơ DC bằng cử chỉ tay trong không trung**, nhưng chỉ khi người đứng trước camera là **thành viên được đăng ký (Team Members)**.

- **Khối AI (Jetson Nano B01)** nhận luồng video từ IP Camera qua giao thức RTSP, thực hiện:
  1. **Nhận diện khuôn mặt** (Face Recognition) và **chống giả mạo** (Face Anti-spoofing, kiểm tra sống bằng chớp mắt). Đúng thành viên mới được phép điều khiển; nếu là người lạ thì **phát cảnh báo và chặn điều khiển**.
  2. **Nhận dạng cử chỉ tay** (Hand Gesture Recognition), phân loại 4 ký tự: **P, A, Z, O**.
  3. Khi nhận dạng thành công, Jetson Nano truyền **2 byte** qua chuẩn **SPI** về STM32 Bluepill.
- **Khối vi xử lý (STM32 Bluepill)** nhận 2 byte từ SPI bằng ngắt, giải mã và phát **tín hiệu PWM** điều khiển động cơ DC thông qua cầu H.

## 2. Kiến trúc và quy trình hệ thống

```
┌───────────┐  RTSP / H.264   ┌──────────────────────────────────────┐   SPI (2 byte)   ┌──────────────────┐   PWM + DIR   ┌────────┐   ┌──────────┐
│ IP Camera │ ──────────────► │          Jetson Nano B01             │ ───────────────► │  STM32 Bluepill  │ ────────────► │ Cầu H  │──►│ Động cơ  │
└───────────┘   (Ethernet)    │  Master SPI  /dev/spidev0.0          │  Master → Slave  │  SPI1 Slave +    │  TIM2_CH1     │ L298N  │   │ DC       │
                              └──────────────────────────────────────┘                  │  ngắt Rx         │  PA0, PB12/13 └────────┘   └──────────┘
                                                                                         └──────────────────┘
```

**Luồng xử lý trên Jetson Nano cho mỗi khung hình:**

```
Khung hình RTSP
      │
      ▼
Phát hiện + nhận diện khuôn mặt ──► có khuôn mặt LẠ? ──► CÓ ──► Cảnh báo (log, ảnh chụp, còi) + CHẶN điều khiển
      │ không                                                      (không chạy nhận dạng cử chỉ, không gửi SPI)
      ▼
Liveness (chớp mắt trong 15 giây gần nhất?) ──► KHÔNG ──► Chờ chớp mắt, vẫn CHẶN
      │ có
      ▼
Quyền điều khiển = MỞ KHÓA
      │
      ▼
MediaPipe Hands (21 điểm mốc) ──► Bộ phân loại P / A / Z / O / NONE
      │
      ▼
Lọc ổn định (≥ 5 khung liên tiếp, độ tin cậy ≥ 80 %) ──► Gửi 2 byte qua SPI ──► STM32
```

**Luồng xử lý trên STM32 Bluepill:**

```
SPI1 Slave nhận 2 byte (ngắt) ──► HAL_SPI_RxCpltCallback: ghép thành mã 16 bit, đẩy vào hàng đợi vòng, re-arm ngắt
                                                │
                                                ▼
                      Vòng lặp chính: lấy mã ──► giải mã ──► đổi chiều (có thời gian nghỉ) / tăng-giảm Duty Cycle TIM2_CH1
```

## 3. Bảng ánh xạ Class – Byte SPI – Tác vụ động cơ

| Class | Mã 16 bit | Byte 1 | Byte 2 | Tác vụ động cơ | Tác động lên PWM / chân điều khiển |
|:-----:|:---------:|:------:|:------:|----------------|------------------------------------|
| **P** | `0xBC9C` | `0xBC` | `0x9C` | Quay **thuận** | `IN1 = 1`, `IN2 = 0`. Nếu Duty đang bằng 0 thì đặt Duty mặc định 40 % |
| **A** | `0xC38C` | `0xC3` | `0x8C` | Quay **ngược** | `IN1 = 0`, `IN2 = 1`. Nếu Duty đang bằng 0 thì đặt Duty mặc định 40 % |
| **Z** | `0x8DDD` | `0x8D` | `0xDD` | **Tăng tốc** | Tăng Duty Cycle PWM thêm 10 %, tối đa 100 % |
| **O** | `0x42A7` | `0x42` | `0xA7` | **Giảm tốc** | Giảm Duty Cycle PWM đi 10 %, tối thiểu 0 % (động cơ dừng) |

Quy ước truyền: **Byte 1 được gửi trước**, MSB first, SPI mode 0 (CPOL = 0, CPHA = 0), 8 bit/từ, mỗi lệnh là một giao dịch gồm đúng 2 byte với CS giữ ở mức thấp.

Thông số PWM: TIM2, tần số **10 kHz** (PSC = 71, ARR = 99), độ phân giải **1 %** nên giá trị thanh ghi CCR1 chính bằng phần trăm Duty Cycle.

Hành vi bổ sung của firmware:

- Khi **đảo chiều** (P ↔ A), firmware cắt PWM, nhả cả hai chân chiều, chờ 100 ms rồi mới đổi chiều để bảo vệ cầu H.
- Lệnh Z hoặc O khi **chưa có chiều quay** (chưa từng nhận P hoặc A) bị bỏ qua.
- Khung có mã không thuộc bảng trên bị bỏ qua.
- Khi giữ cử chỉ **Z** hoặc **O**, Jetson gửi lặp lại mỗi 0,6 giây để tăng/giảm tốc liên tục; **P** và **A** chỉ gửi một lần khi đổi cử chỉ.

## 4. Cấu trúc thư mục dự án

```
BTL_AI_va_VXL/
├── README.md
├── .gitignore
├── jetson_nano/                          # Khối AI chạy trên Jetson Nano B01
│   ├── main.py                           # Chương trình chính: RTSP → khuôn mặt → cử chỉ → SPI
│   ├── config.py                         # Toàn bộ tham số (RTSP, ngưỡng AI, SPI, ánh xạ byte)
│   ├── rtsp_stream.py                    # Đọc RTSP bằng GStreamer (NVDEC), thread nền, tự kết nối lại
│   ├── face_auth.py                      # Nhận diện khuôn mặt + liveness chớp mắt (EAR)
│   ├── gesture_recognizer.py             # MediaPipe Hands + RandomForest + lọc ổn định
│   ├── spi_transport.py                  # Truyền SPI bằng spidev + bộ điều phối lệnh
│   ├── alert.py                          # Cảnh báo người lạ (log, ảnh chụp, còi buzzer)
│   ├── requirements.txt
│   ├── tools/
│   │   ├── enroll_faces.py               # Chụp ảnh đăng ký khuôn mặt thành viên
│   │   ├── build_face_database.py        # Trích đặc trưng → models/face_db.pkl
│   │   ├── collect_gesture_data.py       # Thu thập landmark cử chỉ P, A, Z, O, NONE → CSV
│   │   ├── train_gesture_classifier.py   # Huấn luyện → models/gesture_clf.joblib
│   │   └── spi_send_test.py              # Gửi thử SPI / kiểm tra loopback, không cần camera
│   ├── dataset/                          # (tự tạo khi chạy, không đẩy lên GitHub)
│   │   ├── faces/<ten_thanh_vien>/*.jpg
│   │   └── gestures/landmarks.csv
│   ├── models/                           # (tự tạo) face_db.pkl, gesture_clf.joblib
│   └── logs/intruders/                   # (tự tạo) ảnh chụp người lạ
└── stm32_bluepill/                       # Firmware STM32F103C8T6 (STM32CubeIDE + HAL)
    └── Core/
        ├── Inc/
        │   └── main.h
        └── Src/
            ├── main.c                    # SPI Rx callback, giải mã 2 byte, điều khiển PWM TIM2
            ├── stm32f1xx_hal_msp.c       # Khởi tạo chân SPI1 / TIM2 / NVIC
            └── stm32f1xx_it.c            # SysTick, SPI1_IRQHandler
```

## 5. Phần cứng và sơ đồ đấu nối

### 5.1. Danh sách linh kiện

| Thiết bị | Ghi chú |
|----------|---------|
| Jetson Nano Developer Kit **B01** | Thẻ nhớ microSD ≥ 64 GB (khuyến nghị UHS-I), JetPack 4.6.x |
| Nguồn Jetson 5 V / 4 A (jack DC barrel) | Cắm jumper **J48** để dùng nguồn barrel khi chạy AI nặng |
| IP Camera hỗ trợ RTSP (H.264) | Nối cùng mạng LAN với Jetson Nano |
| STM32 Bluepill (STM32F103C8T6) | Thạch anh ngoài 8 MHz |
| Mạch nạp ST-Link V2 | Nạp qua SWD |
| Mạch cầu H (L298N hoặc tương đương) | Có chân ENA để nhận PWM |
| Động cơ DC + nguồn riêng cho động cơ | Không cấp nguồn động cơ từ Bluepill / Jetson |
| (Tuỳ chọn) còi buzzer | Báo động người lạ, nối chân 29 của Jetson |

### 5.2. Đấu nối SPI: Jetson Nano (Master) ↔ STM32 Bluepill (Slave)

Cả hai bên đều dùng mức logic **3,3 V**, có thể nối trực tiếp.

| Jetson Nano (header 40 chân) | Tín hiệu | STM32 Bluepill |
|------------------------------|----------|----------------|
| Chân **19** | SPI1_MOSI | **PA7** (SPI1_MOSI) |
| Chân **23** | SPI1_SCK | **PA5** (SPI1_SCK) |
| Chân **24** | SPI1_CS0 | **PA4** (SPI1_NSS) |
| Chân **6** (hoặc 9, 14, 20, 25) | GND | **GND** (bắt buộc nối chung mass) |
| Chân 21 | SPI1_MISO | Không nối (firmware chỉ nhận, chế độ Receive Only) |

> Không nối chân 5 V của Jetson vào Bluepill. Cấp nguồn cho Bluepill qua ST-Link hoặc cổng USB riêng.

### 5.3. Đấu nối STM32 Bluepill ↔ cầu H ↔ động cơ

| STM32 Bluepill | Cầu H (L298N) | Ghi chú |
|----------------|---------------|---------|
| **PA0** (TIM2_CH1, PWM 10 kHz) | **ENA** | Tháo jumper ENA trên module L298N |
| **PB12** | **IN1** | Quay thuận khi IN1 = 1, IN2 = 0 |
| **PB13** | **IN2** | Quay ngược khi IN1 = 0, IN2 = 1 |
| **GND** | **GND** | Nối chung mass với nguồn động cơ |
| PC13 | LED trên board | Nhấp nháy khi nhận lệnh hợp lệ |

Nguồn động cơ (ví dụ 9–12 V) cấp vào chân **+12V** của L298N, động cơ nối vào **OUT1 / OUT2**.

## 6. Cài đặt khối AI trên Jetson Nano

### 6.1. Nạp hệ điều hành

1. Tải JetPack 4.6.x SD Card Image cho Jetson Nano từ trang NVIDIA, ghi vào thẻ nhớ bằng balenaEtcher.
2. Khởi động, hoàn tất cài đặt ban đầu, kết nối mạng.
3. Cập nhật hệ thống:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-dev python3-opencv cmake build-essential \
                    libopenblas-dev liblapack-dev libjpeg-dev git nano
```

4. Bật chế độ hiệu năng tối đa:

```bash
sudo nvpmodel -m 0
sudo jetson_clocks
```

### 6.2. Tạo SWAP 4 GB (bắt buộc trước khi biên dịch dlib)

Jetson Nano chỉ có 4 GB RAM; biên dịch `dlib` và chạy đồng thời nhận diện khuôn mặt, MediaPipe sẽ dễ hết bộ nhớ nếu không có SWAP.

```bash
# 1) Tắt zram mặc định của NVIDIA (cần khởi động lại để có hiệu lực hoàn toàn)
sudo systemctl disable nvzramconfig

# 2) Tạo tập tin SWAP 4 GB trên thẻ nhớ
sudo fallocate -l 4G /mnt/4GB.swap
sudo chmod 600 /mnt/4GB.swap
sudo mkswap /mnt/4GB.swap
sudo swapon /mnt/4GB.swap

# 3) Tự động gắn SWAP mỗi lần khởi động
echo "/mnt/4GB.swap none swap sw 0 0" | sudo tee -a /etc/fstab

# 4) Khởi động lại rồi kiểm tra
sudo reboot
```

Sau khi khởi động lại, kiểm tra:

```bash
free -h
swapon --show
```

Dòng `Swap:` phải hiển thị khoảng **4,0G** và `/mnt/4GB.swap` xuất hiện trong danh sách.

> Mẹo: nếu thẻ nhớ chậm, biên dịch dlib có thể mất hơn một giờ. Nên rút bớt các ứng dụng nền (tắt giao diện desktop bằng `sudo systemctl set-default multi-user.target` rồi khởi động lại, sau này đổi lại `graphical.target`).

### 6.3. Bật và cấu hình SPI

Mặc định SPI trên header 40 chân của Jetson Nano **chưa được bật**.

1. Chạy công cụ cấu hình chân:

```bash
sudo /opt/nvidia/jetson-io/jetson-io.py
```

2. Chọn **Configure Jetson 40pin Header** → **Configure header pins manually** → tích chọn **spi1** (chân 19, 21, 23, 24, 26) → **Back** → **Save pin changes** → **Save and reboot to reconfigure pins**.
3. Sau khi khởi động lại, kiểm tra thiết bị SPI:

```bash
ls -l /dev/spidev*
# Kết quả mong đợi: /dev/spidev0.0  /dev/spidev0.1
```

4. Cấp quyền truy cập SPI cho người dùng thường (không cần chạy `sudo`):

```bash
sudo groupadd -f -r spi
sudo usermod -a -G spi $USER
echo 'SUBSYSTEM=="spidev", GROUP="spi", MODE="0660"' | sudo tee /etc/udev/rules.d/99-spidev.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
# Đăng xuất / đăng nhập lại (hoặc reboot) để nhóm mới có hiệu lực
```

5. Kiểm tra phần cứng SPI bằng **loopback**: dùng một dây nối tạm **chân 19 (MOSI) với chân 21 (MISO)**, tháo STM32 ra, rồi chạy (sau khi đã cài thư viện ở mục 6.4):

```bash
cd jetson_nano
python3 tools/spi_send_test.py --loopback
```

Kết quả đúng là `LOOPBACK: THANH CONG`.

### 6.4. Cài thư viện Python

```bash
git clone https://github.com/<ten-tai-khoan>/BTL_AI_va_VXL.git
cd BTL_AI_va_VXL/jetson_nano

python3 -m pip install --upgrade pip
pip3 install -r requirements.txt        # dlib biên dịch từ mã nguồn, mất khá lâu
```

Kiểm tra OpenCV có GStreamer (để giải mã RTSP bằng phần cứng):

```bash
python3 -c "import cv2; print(cv2.getBuildInformation())" | grep -i gstreamer
# Phải thấy: GStreamer: YES
```

### 6.5. Cài đặt MediaPipe

MediaPipe **không có bản wheel chính thức** cho kiến trúc aarch64 chạy Python 3.6 (JetPack 4.x). Có ba hướng, chọn một:

1. Dùng bản wheel aarch64 do cộng đồng biên dịch sẵn phù hợp Python 3.6 của JetPack 4.6 (tự kiểm tra nguồn tin cậy và phiên bản trước khi cài).
2. Tự biên dịch MediaPipe từ mã nguồn bằng Bazel trên Jetson Nano.
3. Dùng hệ điều hành cộng đồng có Python mới hơn (ví dụ Ubuntu 20.04 cho Jetson Nano) rồi cài `mediapipe` bằng `pip`.

Lớp `HandTracker` trong `gesture_recognizer.py` đã tách biệt phần trích landmark; nếu không cài được MediaPipe, có thể thay bằng một backend landmark bàn tay khác mà không phải sửa phần còn lại của dự án.

Kiểm tra sau khi cài:

```bash
python3 -c "import mediapipe as mp; print(mp.__version__)"
```

### 6.6. Cấu hình IP Camera

Đặt URL RTSP qua biến môi trường (không ghi mật khẩu vào mã nguồn):

```bash
export RTSP_URL="rtsp://admin:matkhau@192.168.1.64:554/Streaming/Channels/101"
export RTSP_CODEC="h264"        # đổi thành h265 nếu camera dùng H.265
```

Đường dẫn RTSP khác nhau giữa các hãng camera; xem tài liệu của camera. Có thể kiểm tra nhanh luồng bằng:

```bash
gst-launch-1.0 rtspsrc location="$RTSP_URL" latency=200 ! rtph264depay ! h264parse ! nvv4l2decoder ! nvvidconv ! xvimagesink
```

Nếu luồng xem được ở bước này mà chương trình không mở được, đặt `RTSP_USE_GSTREAMER=0` để dùng FFMPEG.

## 7. Chuẩn bị dữ liệu và huấn luyện

Thực hiện trong thư mục `jetson_nano/`. Các thao tác thu thập cần cửa sổ hiển thị (màn hình HDMI hoặc VNC); có thể thu thập cử chỉ trên máy tính có webcam bằng `--source 0` vì đặc trưng landmark không phụ thuộc camera.

### 7.1. Đăng ký khuôn mặt Team Members

Lặp lại cho từng thành viên (tên viết liền, không dấu):

```bash
python3 tools/enroll_faces.py --name PhanHoaiAnh --count 25
python3 tools/enroll_faces.py --name ThanhVien2 --count 25
python3 tools/build_face_database.py
```

Khi chụp, xoay nhẹ mặt, đổi biểu cảm, thay đổi ánh sáng. Ảnh nào có 0 hoặc nhiều hơn 1 khuôn mặt sẽ bị bỏ qua.

### 7.2. Thu thập dữ liệu cử chỉ

```bash
python3 tools/collect_gesture_data.py
```

| Phím | Chức năng |
|:----:|-----------|
| `p`, `a`, `z`, `o` | Chọn lớp P, A, Z, O |
| `n` | Chọn lớp **NONE** (tay thả lỏng, tư thế ngẫu nhiên; giúp giảm kích hoạt nhầm) |
| `SPACE` | Bật / tắt ghi liên tục (khoảng 15 mẫu/giây) |
| `q` | Thoát |

Mỗi lớp nên có **300–600 mẫu**, thay đổi khoảng cách, góc xoay, ánh sáng, dùng cả hai tay và nhiều người.

> Mỗi ký tự được xử lý như một **tư thế bàn tay tĩnh**. Hãy định nghĩa rõ hình dạng bàn tay cho P, A, Z, O trước khi thu thập, và chọn các hình dạng đủ khác biệt nhau.

### 7.3. Huấn luyện bộ phân loại

```bash
python3 tools/train_gesture_classifier.py
```

Công cụ in báo cáo độ chính xác, ma trận nhầm lẫn trên tập kiểm tra 20 % và lưu `models/gesture_clf.joblib`. Huấn luyện bằng đúng phiên bản scikit-learn đang chạy trên Jetson Nano.

## 8. Cài đặt khối vi xử lý STM32 Bluepill

### 8.1. Tạo dự án bằng STM32CubeIDE / CubeMX

1. **File → New → STM32 Project**, chọn chip **STM32F103C8Tx**, đặt tên dự án (ví dụ `stm32_bluepill`).
2. Cấu hình trong tab **Pinout & Configuration**:

| Mục | Giá trị |
|-----|---------|
| System Core → **SYS** → Debug | **Serial Wire** |
| System Core → **RCC** → High Speed Clock (HSE) | **Crystal/Ceramic Resonator** |
| **Clock Configuration** | HSE 8 MHz, PLL ×9, **HCLK = 72 MHz**, APB1 prescaler /2 |
| Connectivity → **SPI1** | Mode **Receive Only Slave**, Hardware NSS Signal **Hardware Input**, Data Size **8 Bits**, First Bit **MSB First**, CPOL **Low**, CPHA **1 Edge**, CRC **Disabled** |
| SPI1 → **NVIC Settings** | Bật **SPI1 global interrupt** |
| Timers → **TIM2** | Clock Source **Internal Clock**, Channel1 **PWM Generation CH1**, Prescaler **71**, Counter Period **99**, Pulse **0** |
| **GPIO** | **PC13** Output (nhãn `STATUS_LED`), **PB12** Output (`MOTOR_IN1`), **PB13** Output (`MOTOR_IN2`) |

3. Bấm **Generate Code**.
4. **Thay thế** các tập tin do CubeMX sinh ra bằng các tập tin trong repository này (cấu hình trên tương đương với mã đã viết sẵn):

| Tập tin trong repository | Chép đè lên |
|--------------------------|-------------|
| `stm32_bluepill/Core/Inc/main.h` | `Core/Inc/main.h` |
| `stm32_bluepill/Core/Src/main.c` | `Core/Src/main.c` |
| `stm32_bluepill/Core/Src/stm32f1xx_hal_msp.c` | `Core/Src/stm32f1xx_hal_msp.c` |
| `stm32_bluepill/Core/Src/stm32f1xx_it.c` | `Core/Src/stm32f1xx_it.c` |

5. Kiểm tra trong `Core/Inc/stm32f1xx_hal_conf.h` rằng `HAL_SPI_MODULE_ENABLED` và `HAL_TIM_MODULE_ENABLED` đang được bật (CubeMX tự bật khi SPI1 và TIM2 được cấu hình).
6. Biên dịch: **Project → Build Project**. Không được có lỗi.

### 8.2. Nạp chương trình bằng ST-Link V2

| ST-Link V2 | Bluepill (đầu SWD) |
|------------|--------------------|
| SWDIO | DIO |
| SWCLK | DCLK |
| GND | GND |
| 3,3 V | 3V3 |

1. Đặt jumper **BOOT0 = 0** và **BOOT1 = 0** (chạy từ Flash).
2. Cắm ST-Link vào máy tính, trong STM32CubeIDE chọn **Run → Run** (hoặc **Debug**) để nạp.
3. Sau khi nạp, nhấn nút **Reset** trên Bluepill. LED PC13 chớp nhanh liên tục nghĩa là firmware rơi vào `Error_Handler` (khởi tạo thất bại).

### 8.3. Kiểm tra firmware độc lập với AI

Đấu dây SPI theo mục 5.2 và chạy trên Jetson Nano:

```bash
cd jetson_nano
python3 tools/spi_send_test.py P            # động cơ quay thuận, Duty 40 %
python3 tools/spi_send_test.py Z Z          # tăng tốc hai nấc: 60 %
python3 tools/spi_send_test.py O            # giảm tốc một nấc: 50 %
python3 tools/spi_send_test.py A            # đảo chiều (dừng 100 ms rồi quay ngược)
python3 tools/spi_send_test.py O O O O O    # giảm về 0 %, động cơ dừng
python3 tools/spi_send_test.py --raw BC 9C  # gửi byte tuỳ ý để kiểm tra giải mã
```

Mỗi lệnh hợp lệ làm LED PC13 đổi trạng thái một lần. Có thể đo chân PA0 bằng máy hiện sóng/logic analyzer để kiểm tra tần số 10 kHz và Duty Cycle.

## 9. Chạy hệ thống

```bash
cd jetson_nano
export RTSP_URL="rtsp://admin:matkhau@192.168.1.64:554/Streaming/Channels/101"

python3 main.py                 # có cửa sổ hiển thị
python3 main.py --headless      # chạy qua SSH, không có cửa sổ
python3 main.py --dry-run       # không dùng SPI thật (thử trên PC)
python3 main.py --source 0      # dùng webcam thay IP Camera
```

Quy trình sử dụng:

1. Đứng trước camera. Trạng thái **“HAY CHOP MAT (LIVENESS)”** hiện ra; chớp mắt một lần.
2. Khi trạng thái chuyển sang **“DA XAC THUC”** và điều khiển **“MO KHOA”**, đưa tay làm cử chỉ P, A, Z hoặc O.
3. Nếu có khuôn mặt lạ trong khung hình, hệ thống hiện viền đỏ **“CANH BAO NGUOI LA!”**, ghi log, lưu ảnh vào `logs/intruders/`, kêu còi (nếu có) và **không gửi bất kỳ lệnh SPI nào**.

Chạy tự động khi khởi động (tuỳ chọn): tạo dịch vụ `systemd` gọi `python3 main.py --headless` với biến môi trường `RTSP_URL` được khai báo trong `Environment=`.

## 10. Tham số cấu hình

Toàn bộ nằm trong `jetson_nano/config.py`:

| Tham số | Mặc định | Ý nghĩa |
|---------|:--------:|---------|
| `FACE_MATCH_TOLERANCE` | 0,45 | Ngưỡng khoảng cách nhận diện; nhỏ hơn thì chặt hơn, ít nhận nhầm nhưng dễ từ chối thành viên thật |
| `FACE_PROCESS_EVERY_N_FRAMES` | 2 | Xử lý khuôn mặt mỗi N khung hình để giảm tải CPU |
| `EAR_THRESHOLD` | 0,21 | Ngưỡng Eye Aspect Ratio để coi là mắt nhắm |
| `LIVENESS_VALID_SECONDS` | 15 | Thời gian tin cậy sau mỗi lần chớp mắt |
| `AUTHORIZATION_HOLD_SECONDS` | 2 | Giữ quyền khi mất khuôn mặt thoáng qua |
| `GESTURE_MIN_CONFIDENCE` | 0,80 | Độ tin cậy tối thiểu của bộ phân loại |
| `GESTURE_STABLE_FRAMES` | 5 | Số khung liên tiếp phải cùng một cử chỉ |
| `GESTURE_REPEAT_INTERVAL_S` | 0,6 | Chu kỳ gửi lặp khi giữ cử chỉ Z hoặc O |
| `SPI_SPEED_HZ` | 500000 | Tốc độ SPI; nếu mất khung, hạ xuống 100000 |
| `ALARM_TRIGGER_SECONDS` | 1,0 | Người lạ xuất hiện liên tục bao lâu thì báo động |
| `BUZZER_PIN` | `None` | Số chân BOARD của còi (ví dụ 29) |

Biến môi trường: `RTSP_URL`, `RTSP_CODEC`, `RTSP_USE_GSTREAMER`, `SPI_DRY_RUN`.

## 11. Xử lý sự cố

| Hiện tượng | Nguyên nhân thường gặp | Cách xử lý |
|------------|------------------------|------------|
| `Khong mo duoc nguon video` | Sai URL, sai tài khoản camera, khác mạng | Kiểm tra bằng `gst-launch-1.0` (mục 6.6) hoặc VLC trên máy khác |
| Hình ảnh trễ nhiều giây | Bộ đệm tích luỹ | Giữ `drop=true max-buffers=1`; giảm `RTSP_LATENCY_MS`; giảm độ phân giải luồng phụ của camera |
| `Killed` hoặc treo khi cài dlib | Thiếu RAM | Kiểm tra SWAP 4 GB bằng `free -h`; tắt giao diện desktop khi biên dịch |
| `Permission denied: /dev/spidev0.0` | Chưa có quyền | Làm mục 6.3 bước 4, hoặc tạm thời `sudo chmod 666 /dev/spidev*` |
| Không thấy `/dev/spidev*` | Chưa bật SPI | Chạy lại `jetson-io.py` (mục 6.3) |
| Loopback thất bại | Dây sai, SPI chưa bật | Nối đúng chân 19–21, kiểm tra `ls /dev/spidev*` |
| STM32 không phản ứng | Thiếu GND chung, sai chân CS/SCK, SPI chưa ở mode 0 | Kiểm tra mass chung, đo CS/SCK bằng logic analyzer, giảm `SPI_SPEED_HZ` |
| STM32 nhận sai mã sau khi cắm/rút dây | Lệch khung 2 byte | Nhấn Reset Bluepill; nên cố định dây và giữ CS sạch |
| Động cơ không quay dù PWM có | Chưa tháo jumper ENA, thiếu nguồn động cơ, sai IN1/IN2 | Kiểm tra mục 5.3 |
| Luôn bị coi là “NGUOI LA” | CSDL ít ảnh / điều kiện ánh sáng khác | Chụp thêm ảnh ở nhiều điều kiện; tăng nhẹ `FACE_MATCH_TOLERANCE` (không quá 0,5) |
| Mãi ở “HAY CHOP MAT” | EAR không vượt ngưỡng, FPS khuôn mặt thấp | Chỉnh `EAR_THRESHOLD`, giảm `FACE_PROCESS_EVERY_N_FRAMES` xuống 1, chớp mắt rõ ràng |
| Cử chỉ bị kích hoạt khi không làm cử chỉ | Thiếu lớp NONE | Thu thêm dữ liệu lớp NONE rồi huấn luyện lại |
| FPS thấp (< 8) | CPU quá tải | Tăng `FACE_PROCESS_EVERY_N_FRAMES`, giảm `FRAME_WIDTH/HEIGHT`, bật `nvpmodel -m 0` |

## 12. Giới hạn đã biết và hướng phát triển

**Giới hạn của phiên bản hiện tại:**

- **Chống giả mạo bằng chớp mắt** chống được ảnh in và ảnh tĩnh nhưng **không chống được video phát lại có chớp mắt** hoặc mặt nạ 3D. Với ứng dụng cần an toàn cao, nên nâng cấp bằng mô hình Anti-spoofing học sâu (ví dụ MiniFASNet) hoặc camera chiều sâu.
- Cử chỉ được nhận dạng là **tư thế tĩnh**; ký tự mang tính quỹ đạo (vẽ chữ Z trong không khí) cần mô hình chuỗi thời gian.
- Giao thức **không có lệnh dừng riêng** và **không có xác nhận (ACK)** từ STM32. Để dừng động cơ, dùng cử chỉ O nhiều lần cho đến khi Duty về 0 %. Nếu Jetson ngừng hoạt động, động cơ vẫn giữ trạng thái cuối; nên thêm watchdog thời gian chờ lệnh trên STM32 khi triển khai thực tế.
- Khi mất quyền (người lạ xuất hiện), hệ thống **chặn lệnh mới** nhưng không tự động dừng động cơ đang quay.
- Mã chưa được kiểm chứng trên mọi biến thể camera; cần hiệu chỉnh ngưỡng (`config.py`) theo điều kiện thực tế.

**Hướng phát triển:** thêm lệnh dừng khẩn và ACK hai chiều qua SPI, thêm CRC/byte đồng bộ khung, điều khiển vòng kín tốc độ bằng encoder (PID), tăng tốc suy luận bằng TensorRT, thêm giao diện web giám sát.

---

**Tác giả:** Phan Hoài Anh — Hệ thống Nhúng thông minh & IoT, Trường Điện – Điện tử, Đại học Bách khoa Hà Nội.
Dự án phục vụ mục đích học tập. Hãy thêm tập tin `LICENSE` phù hợp trước khi công khai repository.
