# LS2K0300 久久派 — 完整项目仓库结构

> **平台**: 龙芯 LS2K0300 (LoongArch64)
> **开发板**: 久久派 V1.1
> **来源**: [逐飞科技 SEEKFREE 开源库](https://seekfree.taobao.com/)

---

## 📁 顶层目录总览

```
LS2K0300_Library-master/
├── LS2K300_Library/          ← 【核心】开源库 + 例程代码
├── buildroot-2405/           ← Buildroot 2024.05 构建系统 (龙芯 LoongArch 适配)
├── linux-4.19-202506/        ← Linux 4.19 内核源码 (龙芯 LS2K0300 移植版)
├── resource/                 ← 资源图片
├── tools/                    ← 开发工具文档
├── MP3模块-------/           ← MP3 模块资料
├── 【原理图】.../             ← 硬件原理图、丝印图、尺寸图、位号图
├── 【封装】开源封装/           ← PCB 封装库
├── 【文档】说明书 芯片手册等/   ← 芯片数据手册、开发文档
├── 【软件】交叉编译工具 上位机等/ ← 上位机软件、调试工具
├── 久久派升级/                ← 固件升级相关资料
├── 久久派官方资源下载/         ← 官方资源
├── 龙芯2K30x opencv库/       ← OpenCV 4.10 交叉编译版本
└── *.md / *.pdf              ← 项目开发文档 (13+ 份)
```

---

## 🔧 1. LS2K300_Library/ — 核心代码库

### 1.1 目录结构

```
LS2K300_Library/
├── Example/                                  ← 示例代码
│   ├── Coreboard_Demo/                       ← 核心板裸板示例
│   │   ├── E07_pit_demo/                     ←   定时器中断例程
│   │   └── E08_flash_demo/                   ←   Flash 读写例程
│   └── Motherboard_Demo/                     ← 主板外设示例 (9 大类)
│       ├── E1_motherboard/                   ←   E1: 主板基础外设
│       │   ├── E01_01_button_switch_buzzer/  ←     按键 + 蜂鸣器
│       │   ├── E01_02_beep_demo/             ←     蜂鸣器 PWM
│       │   └── E01_03_led_blink_demo/        ←     LED 点灯程序 ★
│       ├── E2_encoder/                       ←   E2: 编码器
│       │   ├── E02_01_encoder_dir_demo/
│       │   └── E02_02_encoder_quad_demo/
│       ├── E3_motor/                         ←   E3: 电机控制
│       │   ├── E03_01_drv8701e_double_motor/
│       │   ├── E03_02_servo_control_demo/
│       │   └── E03_03_brushless_electric_regulating/
│       ├── E4_imu/                           ←   E4: IMU 惯性测量
│       │   └── E04_01_imu_demo/
│       ├── E5_display/                       ←   E5: 显示屏
│       │   ├── E05_01_ips200_display_demo/
│       │   ├── E05_02_tft180_display_demo/
│       │   ├── E05_03_uvc_ips200_display/
│       │   └── E05_04_uvc_fps_show_demo/
│       ├── E6_wireless/                      ←   E6: 网络通信
│       │   ├── E06_01_udp_demo/
│       │   ├── E06_02_tcp_client_demo/
│       │   └── E06_03~05_tcp_oscilloscope_uvc/
│       ├── E7_gnss_range/                    ←   E7: GNSS + 测距
│       │   └── E07_01_tof_dl1x_demo/
│       ├── E8_camera/                        ←   E8: 摄像头
│       │   ├── E08_01~02_tcp_uvc_gray_rgb/
│       │   └── E08_03~05_uvc_display_fps/
│       └── E9_tflite/                        ←   E9: AI 推理 (TensorFlow Lite)
│           ├── E09_01_tflite_picture_classic/
│           └── E09_02_tflite_camera_classic/
│
├── Seekfree_LS2K0300_Opensource_Library/     ← 开源库主代码
│   ├── project/                              ←   项目模板
│   │   ├── user/                             ←     用户代码 (main.cpp)
│   │   ├── code/                             ←     业务逻辑代码
│   │   ├── model/                            ←     TFLite 模型文件
│   │   └── out/                              ←     编译输出
│   └── libraries/                            ←   库文件 (5 层架构)
│       ├── zf_common/                        ←     公共层: typedef, fifo, font
│       ├── zf_driver/                        ←     驱动层: GPIO, PWM, PIT, ADC, ENET...
│       ├── zf_device/                        ←     设备层: IMU, 屏幕, 摄像头, TOF...
│       ├── zf_components/                    ←     组件层: TFLM, ncnn, 调试助手
│       └── doc/                              ←     许可证文档
│
├── build_all.sh                              ← 全量编译脚本
├── delete_temp_file.sh                       ← 临时文件清理
└── .gitignore
```

### 1.2 开源库五层架构

```
应用层 (Example/user main.cpp)
    ↓
设备层 (zf_device/)    — IMU, IPS200屏, TFT180屏, UVC摄像头, DL1X激光测距
    ↓
组件层 (zf_components/) — TensorFlow Lite Micro, ncnn, 逐飞调试助手
    ↓
驱动层 (zf_driver/)    — GPIO, PWM, PIT, ADC, Encoder, Delay, UDP, TCP
    ↓
公共层 (zf_common/)    — 类型定义, FIFO, 字库, 通用函数
```

#### 驱动层文件清单

| 文件 | 功能 |
|---|---|
| `zf_driver_gpio.cpp/hpp` | GPIO 读写 (sysfs /dev 设备文件) |
| `zf_driver_pwm.cpp/hpp` | PWM 输出控制 |
| `zf_driver_pit.cpp/hpp` | 定时器中断 |
| `zf_driver_adc.cpp/hpp` | ADC 模拟采集 |
| `zf_driver_encoder.cpp/hpp` | 编码器读取 |
| `zf_driver_delay.cpp/hpp` | us/ms 延时 (system_delay_ms) |
| `zf_driver_udp.cpp/hpp` | UDP 网络通信 |
| `zf_driver_tcp_client.cpp/hpp` | TCP 客户端 |
| `zf_driver_file_buffer.cpp/hpp` | 文件设备基类 |
| `zf_driver_file_string.cpp/hpp` | 文件字符串操作 |

#### 设备层文件清单

| 文件 | 功能 |
|---|---|
| `zf_device_imu.cpp/hpp` | IMU 惯性测量单元 |
| `zf_device_ips200_fb.cpp/hpp` | IPS 200×200 屏 framebuffer |
| `zf_device_tft180_fb.cpp/hpp` | TFT 180×180 屏 framebuffer |
| `zf_device_uvc.cpp/hpp` | USB Video Class 摄像头 |
| `zf_device_dl1x.cpp/hpp` | DL1X 激光测距模块 |

### 1.3 每个 Demo 的结构

```
E01_03_led_blink_demo/        ← 以 LED 点灯为例
├── user/
│   ├── main.cpp              ← 用户主程序
│   ├── CMakeLists.txt        ← CMake 构建配置
│   ├── cross.cmake           ← 龙芯 LoongArch 交叉编译工具链
│   └── build.sh              ← 一键编译 + SCP 上传脚本
├── code/                     ← 业务代码目录
├── model/                    ← AI 模型文件目录
└── out/                      ← 编译输出目录
```

---

## 🐧 2. linux-4.19-202506/ — Linux 内核

龙芯 LS2K0300 移植版 Linux 4.19 内核，包含：

- **arch/loongarch/** — LoongArch 架构支持
- **drivers/** 中新增 LS2K0300 外设驱动：
  - GPIO (sysfs 接口 → `/dev/zf_gpio_*`)
  - PWM, ADC, PIT, UART, SPI, I2C
  - USB Video Class (UVC)
  - 网络驱动 (GMAC)
- **设备树 (DTS)**: `seekfree_2k0300_coreboard.dts` 定义板级 GPIO 引脚映射

---

## 📦 3. buildroot-2405/ — Buildroot 构建系统

龙芯 LoongArch 架构适配的 Buildroot 2024.05：

| 配置项 | 说明 |
|---|---|
| `config_systemd_2.0` | 带 systemd 的完整系统配置 |
| `config_mini_16MB` | 极简 16MB Flash 配置 |
| `config_prt_scanner_*` | 打印机扫描仪配置 |
| `seekfree_overlay/` | 逐飞科技定制 rootfs 叠加层 |
| `board/.../Config.in.loongarch` | LoongArch 架构配置 |

---

## 📚 4. 项目开发文档 (13+ 份)

| 文件 | 说明 |
|---|---|
| `README.md` / `README.en.md` | 项目说明 (中/英) |
| `LS2K0300_统一代码库.md` | 统一代码库总览 |
| `LS2K300_Library源代码注释文档.md` | 库代码注释文档 |
| `LS2K0300久久派_V1.1_板卡使用手册_v1.2_20240705.pdf` | 板卡使用手册 |
| `2K0300引脚复用配置与中断分配.pdf` | 引脚复用与中断分配表 |
| `项目申报书_完整版.md` / `项目申报书_修订版.md` | 项目申报书 |
| `项目开发记录.md` | 开发记录 |
| `传感器集成板原理图.md` / `传感器集成板_PCB参数表.md` | 传感器扩展板资料 |
| `技术调试与复盘日志.md` / `技术调试与复盘日志_完整版.md` | 技术调试日志 |
| `全对话技术复盘日志_完整版.md` | 全对话复盘 |
| `2026-06-25_晚间调试记录.md` | 某次调试记录 |

---

## 🛠️ 5. 硬件资料

### 【原理图】原理图 丝印图 尺寸图 位号图/
- 核心板 + 主板原理图
- PCB 丝印层、尺寸图、元件位号图

### 【封装】开源封装/
- PCB 封装库 (KiCad/AD 等)

### 【文档】说明书 芯片手册等/
- LS2K0300 芯片数据手册
- 各外设模块规格书

### 【软件】交叉编译工具 上位机等/
- 上位机调试工具
- 串口/UDP 调试助手

---

## 🚀 6. GPIO 引脚速查 (主板)

| 功能 | 设备节点 | GPIO | 备注 |
|---|---|---|---|
| 蜂鸣器 | `/dev/zf_gpio_beep` | GPIO26 | 主板默认输出 |
| 按键 1 | `/dev/zf_gpio_key_1` | GPIO77 | |
| 按键 2 | `/dev/zf_gpio_key_2` | GPIO78 | |
| 按键 3 | `/dev/zf_gpio_key_3` | GPIO79 | |
| 按键 4 | `/dev/zf_gpio_key_4` | GPIO80 | |
| 电机 1 | `/dev/zf_gpio_motor_1` | — | |
| 电机 2 | `/dev/zf_gpio_motor_2` | — | |
| 霍尔检测 | `/dev/zf_gpio_hall_detection` | — | |
| **板载 S_LED** | `/sys/class/gpio/gpio83/` | **GPIO83** | ★ 久久派板载 LED |

---

## 📌 7. 交叉编译环境要求

| 组件 | 版本/路径 |
|---|---|
| 交叉工具链 | GCC 8.3, `/opt/ls_2k0300_env/loongson-gnu-toolchain-8.3-*/` |
| OpenCV | 4.10, `/opt/ls_2k0300_env/opencv_4_10_build/` |
| CMake | ≥ 3.5 |
| 目标架构 | `-march=loongarch64 -mtune=loongarch64` |
| 编译优化 | `-O3 -fPIC -ffunction-sections -fdata-sections` |
