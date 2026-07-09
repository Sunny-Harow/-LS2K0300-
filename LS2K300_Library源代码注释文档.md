# LS2K0300 全自动巡检小车 — 完整源代码注释文档

> **硬件平台**: 龙芯 LS2K0300 久久派 V1.1 | LoongArch64 | LoongOS V2 (Linux 4.19.190+)
> **开发语言**: C++17 | **编译器**: GCC 8.3.0 (loongarch64-linux-gnu-g++)
> **OpenCV**: 4.10.0 预编译 LoongArch64 版
> **文档生成日期**: 2026-07-08

---

# 目录

1. [开发环境与编译体系](#一开发环境与编译体系)
2. [硬件接线全表](#二硬件接线全表)
3. [主功能程序详解](#三主功能程序详解)
   - [auto_nav.cpp — 全功能整合主程序](#31-auto_navcpp--全功能整合主程序)
   - [avoid.cpp — 超声波避障](#32-avoidcpp--超声波避障)
   - [wallfollow.cpp — 左墙跟随 + 视觉里程计](#33-wallfollowcpp--左墙跟随--视觉里程计)
   - [human_follow.cpp — 人体跟踪](#34-human_followcpp--人体跟踪)
4. [安全检测程序详解](#四安全检测程序详解)
   - [fire_test.cpp — 火焰检测](#41-fire_testcpp--火焰检测)
   - [smoke_detect.cpp — 香烟检测](#42-smoke_detectcpp--香烟检测)
5. [语音与通信程序详解](#五语音与通信程序详解)
   - [mp3.cpp — DFPlayer Mini MP3 模块](#51-mp3cpp--dfplayer-mini-mp3-模块)
   - [sms_alert.cpp — 语音触发短信报警](#52-sms_alertcpp--语音触发短信报警)
   - [child_call.cpp — 儿童走失呼叫系统](#53-child_callcpp--儿童走失呼叫系统)
6. [环境交互程序详解](#六环境交互程序详解)
   - [quiet_remind.cpp — 噪音检测与静音提醒](#61-quiet_remindcpp--噪音检测与静音提醒)
7. [定位与导航程序详解](#七定位与导航程序详解)
   - [visual_odo.cpp — 视觉里程计](#71-visual_odocpp--视觉里程计)
8. [基础测试程序详解](#八基础测试程序详解)
9. [核心技术原理深度解析](#九核心技术原理深度解析)
10. [DFPlayer Mini TF 卡音频配置](#十dfplayer-mini-tf-卡音频配置)
11. [软件架构与设计模式](#十一软件架构与设计模式)

---

# 一、开发环境与编译体系

## 1.1 交叉编译工具链

| 项目 | 配置 |
|------|------|
| 宿主机 | Ubuntu 24.04 虚拟机 |
| 工具链路径 | `/opt/ls_2k0300_env/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6/` |
| C++ 编译器 | `loongarch64-linux-gnu-g++` (GCC 8.3.0) |
| 目标架构 | loongarch64-linux-gnu |
| OpenCV 路径 | `/opt/ls_2k0300_env/opencv_4_10_build/` |
| OpenCV 版本 | 4.10.0（预编译 LoongArch64） |
| C++ 标准 | C++17 |
| 优化级别 | `-O2` 或 `-O3 -march=loongarch64 -mtune=loongarch64` |

## 1.2 编译命令

### 无第三方依赖的程序 (motor_test, servo 等)

```bash
export PATH=/opt/ls_2k0300_env/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6/bin:$PATH
loongarch64-linux-gnu-g++ -O2 -o 程序名 源文件.cpp
scp -O 程序名 root@172.20.10.2:/home/root/
```

### 依赖 OpenCV 的程序 (auto_nav, fire_test 等)

```bash
loongarch64-linux-gnu-g++ -O2 -o 程序名 源文件.cpp \
    -I/opt/ls_2k0300_env/opencv_4_10_build/include/opencv4 \
    -L/opt/ls_2k0300_env/opencv_4_10_build/lib \
    -lopencv_core -lopencv_imgcodecs -lopencv_imgproc -lopencv_videoio \
    -lopencv_objdetect -lpthread -std=c++17 \
    -Wl,-rpath-link,/opt/ls_2k0300_env/opencv_4_10_build/lib
```

> **`-Wl,-rpath-link` 的作用**: 交叉编译时，libopencv_core.so 链接时又引用了 libopencv_imgproc.so 的符号。`-rpath-link` 告诉链接器在编译时去这个目录找传递依赖。运行时板子上的 `.so` 已通过系统路径找到，所以不需要 `-rpath`。

## 1.3 CMake 构建

项目也支持 CMake 构建（`build.sh` 脚本 + `CMakeLists.txt` + `cross.cmake`）：

- **cross.cmake**: 设置交叉编译器路径和系统类型
- **CMakeLists.txt**: 自动收集 `../user`、`../code`、`../model` 目录下的所有源文件
- **build.sh**: 一键清理→cmake→make→scp上传

## 1.4 hgfs 共享文件夹的符号链接问题

VMware hgfs 不支持 Linux 符号链接。如果从 Windows 复制 OpenCV 库到 hgfs，`libopencv_core.so → libopencv_core.so.4.10.0` 的软链接会变成 0 字节文件。

**解决方案**: 将 `.tar.xz` 解压到 Ubuntu 本地 ext4 文件系统（如 `~/opencv_4_10_build/`）。

---

# 二、硬件接线全表

## 2.1 电机驱动 (L298N)

| L298N 引脚 | 开发板物理PIN | GPIO号 | 功能 |
|-----------|:----------:|:-----:|------|
| IN1 | PIN 22 | **GPIO60** | 左电机正转控制 |
| IN2 | PIN 26 | **GPIO61** | 左电机反转控制 |
| IN3 | PIN 28 | **GPIO62** | 右电机正转控制 |
| IN4 | PIN 24 | **GPIO63** | 右电机反转控制 |
| ENA | 5V 跳线帽 | — | 全速使能（PWM 通过软件在 IN 脚实现） |
| ENB | 5V 跳线帽 | — | 全速使能 |
| GND | PIN 14 (GND0) | — | **必须共地！** |
| +12V | 电池正极 | — | 5×AA 干电池 7.5V |

### 差速转向控制逻辑

| 动作 | IN1(左正) | IN2(左反) | IN3(右正) | IN4(右反) |
|------|:-------:|:-------:|:-------:|:-------:|
| **前进** | 1 | 0 | 1 | 0 |
| **后退** | 0 | 1 | 0 | 1 |
| **原地左转** | 0 | 1 | 1 | 0 |
| **原地右转** | 1 | 0 | 0 | 1 |
| **停止** | 0 | 0 | 0 | 0 |

## 2.2 传感器与模块接线

| 模块 | 信号引脚 | 开发板 PIN | GPIO/接口 | 备注 |
|------|---------|:--------:|:--------:|------|
| **US-100 超声波** | VCC | PIN 2 | 5V | |
| | GND | PIN 20 | GND | |
| | TRIG | PIN 9 | **GPIO72** | 触发脉冲输出 |
| | ECHO | PIN 12 | **GPIO44** | 回波脉宽输入 |
| **SG90 舵机** | SIG(橙) | PIN 23 | **GPIO67** | 50Hz PWM |
| | VCC(红) | 电池正极 | 7.5V | **独立供电！** |
| | GND(棕) | 电池负极+板子GND | | **必须共地！** |
| **USB 摄像头** | USB 插头 | 板子 USB 口 | `/dev/videoX` | UVC 免驱 |
| **DFPlayer Mini MP3** | VCC | PIN 2 | 5V | |
| | GND | PIN 6 | GND | |
| | RX | UART1_TXD | `/dev/ttyS1` | 9600bps, 单向控制 |
| | TX | — | 不接 | |
| **SU-03T 语音唤醒** | VCC | PIN 2 | 5V | |
| | GND | PIN 6 | GND | |
| | RX/TX | UART2 | `/dev/ttyS2` | 9600bps |
| **合宙 Air780E 4G** | VCC | LM2596→5V | 5V (2A峰值) | |
| | GND | 板子 GND | | 共地 |
| | RX/TX | UART3 | `/dev/ttyS3` | 115200bps |
| | MIC± | 驻极体咪头 | | 通话拾音 |
| | SPK± | 4Ω3W 扬声器 | | 通话放音 |
| **KY-038 声音传感器** | VCC | PIN 2 | 5V | |
| | GND | PIN 6 | GND | |
| | DO | PIN 14 | **GPIO45** | 数字输出(HIGH=有声) |

## 2.3 GPIO 禁区（绝对不可触碰！）

| GPIO 范围 | 占用功能 | 后果 |
|:--------:|---------|------|
| GPIO90-95 | eMMC 接口 | **触碰系统崩溃！** |
| GPIO96-105 | eMMC + SDIO1 (WiFi) | WiFi 失效 |
| GPIO50-51 | 新内核 I2C1 占用 | 冲突 |

---

# 三、主功能程序详解

## 3.1 auto_nav.cpp — 全功能整合主程序

**功能概述**: 这是巡检小车的核心主程序，集成了火焰检测、人腿检测、MP3 分级语音播报、100Hz 软件 PWM 降速、MJPEG 远程监控等功能。

### 源文件结构

```
auto_nav.cpp (314行)
├── 1. GPIO 电机控制 (行27-75)
│   ├── gpio_out()      — sysfs GPIO 导出与初始化
│   ├── gpio()          — GPIO 值写入(fopen/fclose方式)
│   ├── m_fwd()/m_stop()— 基础运动函数
│   ├── speed_thread()  — 100Hz PWM 独立线程
│   ├── pfd_w()         — 持久 fd 写入(lseek+write)
│   ├── du()            — 微秒级忙等(clock_gettime)
│   └── motor_on()/off()— 原子变量同步控制
├── 2. MP3 串口模块 (行77-105)
│   ├── mp3_init()      — 初始化(杀getty+硬件复位0x0C)
│   └── mp3_play()      — 10字节命令帧发送
├── 3. 摄像头 + 检测 (行107-159)
│   ├── cam_thread()    — 摄像头采集线程(自动扫描0-4)
│   ├── detect()        — Haar 级联人腿检测(每5帧1次)
│   └── detect_fire()   — HSV 火焰检测(4通道+闪烁)
├── 4. MJPEG 推流 (行162-198)
│   └── mjpeg_thread()  — HTTP MJPEG 服务器(:8081)
└── 5. 主状态机 (行200-313)
    └── main()          — 火焰优先→人腿→避障状态机
```

### 关键实现细节

#### GPIO 初始化 (gpio_out)

```cpp
// 双保险策略: 先 unexport 清除残留状态 → export 重新导出 → 设置方向
static void gpio_out(int g){
    char p[64]; snprintf(p,64,"/sys/class/gpio/gpio%d/value",g);
    if(access(p,0)==0){  // 已导出 → 先释放
        FILE*f=fopen("/sys/class/gpio/unexport","w");
        if(f){fprintf(f,"%d",g);fclose(f);usleep(50000);}
    }
    FILE*f=fopen("/sys/class/gpio/export","w");  // 重新导出
    if(f){fprintf(f,"%d",g);fclose(f);usleep(100000);}
    snprintf(p,64,"/sys/class/gpio/gpio%d/direction",g);
    f=fopen(p,"w"); if(f){fprintf(f,"out");fclose(f);}
}
```

**设计理由**: sysfs 导出的 GPIO 可能被上次程序残留的状态影响。先 unexport 再 export 确保干净的初始状态。

#### 100Hz 软件 PWM 降速

```cpp
// 持久文件描述符: 避免每次 fopen/fclose 的 ~200µs 开销
static int pfd[4]={-1,-1,-1,-1};

// 微秒级忙等: Linux usleep() 精度仅 ~10ms，无法做 100Hz PWM
static void du(int us){
    long long e=0; struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC,&ts);
    e=(long long)ts.tv_sec*1000000+ts.tv_nsec/1000+us;
    do{clock_gettime(CLOCK_MONOTONIC,&ts);}
    while((long long)ts.tv_sec*1000000+ts.tv_nsec/1000<e);
}

// 独立线程: 避免主循环影响 PWM 时序
static void speed_thread(){
    while(keep_running.load()){
        if(!motor_running.load()){ usleep(10000); continue; }  // 空闲时低CPU
        // ON: 4ms (40% 占空比) = 前进
        pfd_w(pfd[0],1); pfd_w(pfd[1],0); pfd_w(pfd[2],1); pfd_w(pfd[3],0);
        du(4000);
        // OFF: 6ms
        pfd_w(pfd[0],0); pfd_w(pfd[1],0); pfd_w(pfd[2],0); pfd_w(pfd[3],0);
        du(6000);
    }
}
```

**关键技术点**:
1. **持久 fd**: `open()` 一次后保持 `pfd[]`，用 `lseek+write` 替代 `fopen+fprintf+fclose`，从 ~200µs 降到 ~80µs
2. **忙等延时**: `clock_gettime(CLOCK_MONOTONIC)` 精度 ~1µs，比 `usleep()` 的 ~10ms 高 10000 倍
3. **原子变量**: `std::atomic<bool>` 实现线程安全的电机状态切换
4. **40% 占空比**: 4ms ON / 6ms OFF = 有效降速 60%，保证巡检平稳

#### MP3 模块初始化

```cpp
static void mp3_init(){
    system("killall -9 getty 2>/dev/null");  // 释放被 getty 占用的串口
    usleep(300000);
    mp3_fd = open("/dev/ttyS1", O_RDWR|O_NOCTTY);
    // ... 配置 9600 8N1 ...
    usleep(1500000);  // 等 DFPlayer 上电稳定
    // 硬件复位命令 (0x0C): 解决模块卡死问题
    int rs=0xFF+0x06+0x0C+0x00+0x00+0x00, rc=-rs;
    unsigned char rst[10]={0x7E,0xFF,0x06,0x0C,0x00,0x00,0x00,
        (unsigned char)((rc>>8)&0xFF),(unsigned char)(rc&0xFF),0xEF};
    write(mp3_fd,rst,10); usleep(500000);
}
```

**设计理由**: DFPlayer Mini 在反复测试、频繁格卡后容易进入内部错误状态（蓝灯亮但不出声）。硬件复位命令 (0x0C) 是兜底解决方案。

#### 火焰检测

```cpp
static bool detect_fire(){
    if(fcnt%5!=0) return fire_px>800;  // 每5帧检测1次，复用上次结果
    cv::Mat f;
    { std::lock_guard<std::mutex> l(lock); f=shared.clone(); }
    cv::Mat hsv, m1, m2, m3, m4, mask;
    cv::cvtColor(f,hsv,cv::COLOR_BGR2HSV);

    // 四通道火焰颜色检测 (注意 OpenCV 红色跨 H 边界)
    cv::inRange(hsv,cv::Scalar(0,150,150),cv::Scalar(10,255,255),m1);    // 红(前)
    cv::inRange(hsv,cv::Scalar(160,150,150),cv::Scalar(179,255,255),m2);  // 红(后)
    cv::inRange(hsv,cv::Scalar(10,150,150),cv::Scalar(25,255,255),m3);   // 橙
    cv::inRange(hsv,cv::Scalar(25,150,150),cv::Scalar(35,255,255),m4);   // 黄
    mask=m1|m2|m3|m4;  // 四通道 OR 合并

    fire_px=cv::countNonZero(mask);
    // 闪烁判定: 帧间变化率 > 40% = 真火 (排除静态干扰)
    int chg=abs(fire_px-fire_prev); fire_prev=fire_px;
    return fire_px>800 && (fire_prev>0? (float)chg/fire_prev>0.4 : false);
}
```

**火焰检测四通道设计原理**:
- **红色(前)**: H:0-10°，覆盖火焰的深红部分
- **红色(后)**: H:160-179°，OpenCV 中红色跨 H 通道边界（0°和 180°是同一颜色）
- **橙色**: H:10-25°，火焰中层温度颜色
- **黄色**: H:25-35°，火焰外层温度颜色
- **饱和度 S>150**: 排除低饱和度的白色/灰色物体
- **亮度 V>150**: 排除暗色物体
- **闪烁判定 >40%**: 火焰本质特征——帧间像素大幅变化，排除人手/灯光等静态物体

#### 主状态机

```
状态机设计:
┌──────────────────────────────────────────────────┐
│                火焰检测 (最高优先级)                │
│  fire=true → 停车 + mp3_play(3) + fire_active=true │
│  fire_active && 无火3秒 → fire_active=false        │
├──────────────────────────────────────────────────┤
│  !fire_active 时:                                  │
│                                                    │
│  S_NORMAL (正常行驶):                              │
│    motor_on() + 每13s播0001 + detect()             │
│    detect()>0 → motor_off() + mp3_play(2)          │
│              → state=S_STOPPED + stop_time=now      │
│                                                    │
│  S_STOPPED (停车等待):                             │
│    motor_off() + 继续检测                          │
│    detect()==0 && now-stop_time>=3s                │
│              → motor_on() + mp3_play(1)            │
│              → state=S_NORMAL                       │
└──────────────────────────────────────────────────┘
循环速率: usleep(80000) ≈ 12.5Hz
```

#### 摄像头自动扫描

```cpp
// 解决 /dev/videoX 编号随重启变化的经典问题
static void cam_thread(){
    cv::VideoCapture cap;
    int idx=-1;
    for(int i=0;i<5;i++){  // 依次尝试 0-4
        cap.open(i);
        if(cap.isOpened()){ idx=i; break; }
    }
    if(idx<0){ printf("CAM FAIL\n"); keep_running.store(false); run=false; return; }
    // ... 320×240@15fps ...
}
```

---

## 3.2 avoid.cpp — 超声波避障

**功能概述**: 基于 US-100 超声波传感器 + SG90 舵机的全自动避障程序。参考 Arduino Project Hub 和 BlueMazer/ParmarHarsh 开源扫描算法。

### 源文件结构

```
avoid.cpp (168行)
├── GPIO 底层 (行30-52)
│   ├── us_now()        — 微秒时间戳
│   ├── du()            — 忙等延时
│   ├── gpio_setup()    — GPIO 导出+方向
│   ├── gpio_open()     — 打开value文件获取fd
│   ├── fd_w() / fd_r() — 持久fd读写
│   └── pfd_w()         — 电机fd写入
├── PWM 调速 (行56-67)
│   ├── speed_thread()  — 60%占空比向前
│   └── go()            — 原子变量切换方向
├── 舵机控制 (行70-81)
│   ├── servo()         — 转到目标角度+保持ms时长
│   └── spulse()        — 单次位置脉冲(维持角度)
├── 超声波测距 (行83-106)
│   ├── quick_dist()    — 快速测距(舵机不动时,~60ms)
│   └── look()          — 转角度+稳定400ms+测距
└── 主避障循环 (行110-167)
    └── main()          — 前进→快测→遇障→扫两方向→决策转向
```

### 舵机控制 — 关键实现

```cpp
// 转到目标角度并保持 ms 毫秒
static void servo(int deg, int ms){
    if(deg<0)deg=0; if(deg>180)deg=180;
    int pulse=600+(2400-600)*deg/180;  // 600µs(0°) ~ 2400µs(180°)
    long long end=us_now()+ms*1000;
    // 持续发 PWM 直到超时
    while(us_now()<end){
        fd_w(fd_servo,1); du(pulse);       // 高电平 pulse µs
        fd_w(fd_servo,0); du(20000-pulse); // 低电平 20ms-pulse
    }
}

// 单次位置脉冲（维持当前角度，不停在发 PWM）
static void spulse(int deg){
    int pulse=600+(2400-600)*deg/180;
    fd_w(fd_servo,1); du(pulse);
    fd_w(fd_servo,0); du(20000-pulse);
    // 发两次确保舵机收到
    fd_w(fd_servo,1); du(pulse);
    fd_w(fd_servo,0); du(20000-pulse);
}
```

**SG90 设计要点**:
- 脉冲范围: 600-2400µs（对应 0-180°），50Hz (20ms周期)
- **必须独立供电**: 从电池直接取电，不能从板子5V取（堵转电流2.5A）
- **必须共地**: 舵机 GND 和板子 GND 连接

### 超声波测距 — 关键实现

```cpp
// 快速测距 (~60ms一次，舵机不动时使用)
static double quick_dist(){
    // 1. 等 ECHO 回低 (避免上次测量残留)
    long long dl=us_now()+100000;
    while(fd_r(fd_echo)&&us_now()<dl);

    // 2. 发 TRIG 15µs 脉冲
    fd_w(fd_trig,1); du(15); fd_w(fd_trig,0);

    // 3. 等 ECHO 变高
    dl=us_now()+100000;
    while(!fd_r(fd_echo)&&us_now()<dl);
    if(us_now()>=dl) return 99;  // 超时=无障碍

    // 4. 测 ECHO 高电平脉宽
    long long t1=us_now();
    dl=us_now()+100000;
    while(fd_r(fd_echo)&&us_now()<dl);
    if(us_now()>=dl) return 99;

    // 5. 距离 = 脉宽(µs) × 0.01715 (声速补偿系数)
    return (double)(us_now()-t1)*0.01715;
}
```

**US-100 测距原理**:
- 声速 ≈ 343m/s (20°C)，往返距离 = 34300cm/s × (脉宽/2)
- 系数推导: 34300 / 2 / 1000000 = 0.01715 cm/µs
- 超时 100ms ≈ 最大可测约 1715cm（远超450cm的传感器规格）

### 避障决策算法

```
停车-扫描-决策-执行 四步循环:

快测前方 → 距离 > THRESH → 确保前进 + 50ms后重测 (快速通道)
         → 距离 ≤ THRESH → 停车!
              ├── 扫左(0°) 测 dL
              ├── 扫右(90°) 测 dR
              └── 决策:
                    dL > THRESH && dL > dR → 后退0.8s→左转0.8s→前进0.8s
                    dR > THRESH              → 后退0.8s→右转0.8s→前进0.8s
                    都堵                      → 后退1.5s→右转1.2s→前进1.0s
```

**设计亮点**: 平时舵机不动，高频快测前方（~60ms/次）；只有前方遇到障碍时才转动舵机扫两侧。这样既保证了快速响应，又减少了舵机频繁转动。

---

## 3.3 wallfollow.cpp — 左墙跟随 + 视觉里程计

**功能概述**: 保持左侧墙壁 30cm 距离行走，同时运行视觉里程计跟踪行驶轨迹。参考 robotics-hana/Wall-Following 开源方案。

### 源文件结构

```
wallfollow.cpp (202行)
├── GPIO + PWM + 舵机 (行34-68)
├── 超声波测距 get_dist() (行78-90)
├── 三方向扫描 scan_surroundings() (行93-97)
├── 视觉里程计 (行100-147)
│   ├── cam_thread()        — 灰度图采集
│   └── update_odometry()   — FAST角点+LK光流
└── 主循环 (行157-201)
    └── main()              — 停车→扫三方向→决策→执行
```

### 三方向扫描 — 墙跟随核心

```cpp
// 扫左边三个方向: 120°(左后)、90°(正左=墙)、60°(左前)
static void scan_surroundings(double *dRear, double *dWall, double *dFront){
    servo(120, 300); *dRear=get_dist();   // 左后: 测墙壁连续性
    servo(90,  200); *dWall=get_dist();   // 正左: 测墙距
    servo(60,  200); *dFront=get_dist();  // 左前: 测拐角/障碍
}
```

### 比例控制决策

```cpp
if(dFront<20){
    // 左前有障碍 → 右转避开
    printf("左前堵! 右转\n"); go(4); usleep(400000);
}else{
    double err=dWall-WALL_TARGET;  // 误差 = 实际墙距 - 目标30cm
    if(dWall>300){
        // 丢墙(>3m) → 左转找墙
        printf("丢墙 左找\n"); go(3); usleep(300000);
    }else if(err>10){
        // 离墙太远(>40cm) → 左转靠墙
        printf("远%.0f 左靠\n",dWall); go(3); usleep(200000);
    }else if(err<-10){
        // 离墙太近(<20cm) → 右转离墙
        printf("近%.0f 右离\n",dWall); go(4); usleep(200000);
    }else{
        // 20-40cm → 直走
        printf("OK 直走\n"); go(1); usleep(400000);
    }
}
```

### 视觉里程计 — 光流跟踪

```cpp
static void update_odometry(){
    // 1. 获取灰度帧
    cv::Mat gray; { std::lock_guard<std::mutex> l(cam_lock); gray=cam_gray.clone(); }

    // 2. 提取 100 个 FAST 角点
    std::vector<cv::Point2f> new_pts;
    cv::goodFeaturesToTrack(gray,new_pts,100,0.01,10);

    // 3. Lucas-Kanade 金字塔光流跟踪
    if(!prev_pts.empty() && !new_pts.empty()){
        std::vector<cv::Point2f> tracked;
        std::vector<unsigned char> st; std::vector<float> err;
        cv::calcOpticalFlowPyrLK(prev_gray,gray,prev_pts,tracked,st,err);

        // 4. 统计水平位移 → 推算航向角变化
        double sum_dx=0; int cnt=0;
        for(size_t i=0;i<st.size();i++){
            if(!st[i]) continue;                     // 跟踪失败的点跳过
            double dx=tracked[i].x-prev_pts[i].x;
            if(fabs(dx)>50) continue;               // 过滤异常值
            sum_dx+=dx; cnt++;
        }
        if(cnt>5){                                  // 至少5个有效点
            double turn=sum_dx/cnt*0.1;             // 标定系数: 0.1°/像素
            heading+=turn;                           // 累积航向角
            // 推算轨迹坐标
            double fwd=10; double rad=heading*M_PI/180.0;
            pos_x+=fwd*cos(rad); pos_y+=fwd*sin(rad);
        }
    }
    prev_gray=gray.clone(); prev_pts=new_pts;
}
```

**Lucas-Kanade 光流法原理**:
1. **FAST 角点**: `goodFeaturesToTrack()` 在图像中找 100 个"好跟踪"的特征点
2. **金字塔光流**: `calcOpticalFlowPyrLK()` 在图像金字塔的每一层跟踪这些点的位移
3. **方向推算**: 水平像素位移 avg_dx × 标定系数 = 航向角变化量
4. **异常过滤**: |dx|>50 像素的认为是误匹配，丢弃

---

## 3.4 human_follow.cpp — 人体跟踪

**功能概述**: 先通过帧差法找运动区域，再在运动区域内跑 Haar 人腿检测，根据人框位置调整小车方向实现跟踪。

### 核心创新 — 运动预过滤

```cpp
static bool detect_person(int &cx, int &ah){
    // 第1步: 帧差法找运动区域
    cv::Mat mmask;
    if(!prev_gray.empty()){
        cv::Mat diff;
        cv::absdiff(gray,prev_gray,diff);              // 帧间差分
        cv::threshold(diff,mmask,25,255,cv::THRESH_BINARY); // 二值化
        cv::Mat k=cv::getStructuringElement(cv::MORPH_RECT,cv::Size(8,8));
        cv::dilate(mmask,mmask,k);                     // 膨胀连接碎片
    }
    prev_gray=gray.clone();

    // 第2步: 只在运动区域跑 Haar 检测 (大幅减少无效计算)
    std::vector<cv::Rect> found;
    if(!mmask.empty()){
        std::vector<std::vector<cv::Point>> cs;
        cv::findContours(mmask,cs,cv::RETR_EXTERNAL,cv::CHAIN_APPROX_SIMPLE);
        for(auto &c: cs){
            cv::Rect r=cv::boundingRect(c);
            if(r.area()<500) continue;                 // 太小的忽略
            // 在运动区域周围扩展一些余量
            r.x=std::max(0,r.x-30); r.y=std::max(0,r.y-40);
            r.width=std::min(frame.cols-r.x,r.width+60);
            r.height=std::min(frame.rows-r.y,r.height+80);
            // 在 ROI 内检测
            bc.detectMultiScale(frame(r),lf,1.08,2,0,cv::Size(20,30));
            for(auto &x:lf){ x.x+=r.x; x.y+=r.y; found.push_back(x); }
        }
    }
    // 第3步: 运动区域无结果 → 全图搜索兜底
    if(found.empty()) bc.detectMultiScale(frame,found,1.08,2,0,cv::Size(20,30));
    // ...
}
```

### 跟踪控制逻辑

```cpp
// 按人框中心位置调整方向
int off=cx-160;  // 160 = 图像中心(320/2)
if(off<-30)      { go(3); usleep(250000); }  // 人在左边 → 左转
else if(off>30)  { go(4); usleep(250000); }  // 人在右边 → 右转
go(1); // 始终前进

// 5帧检测不到人 → 停车
if(!found){ lost++; if(lost>=5){ go(0); } }
else lost=0;
```

---

# 四、安全检测程序详解

## 4.1 fire_test.cpp — 火焰检测

**功能概述**: 火焰检测独立测试程序，通过 HSV 颜色空间 + 闪烁特征判定火焰，MJPEG 推流到 8082 端口。

### 检测算法

```cpp
static bool detect_fire(cv::Mat &f, int &pixels){
    cv::Mat hsv, mask1, mask2, mask3, mask;
    cv::cvtColor(f, hsv, cv::COLOR_BGR2HSV);

    // 三通道火焰颜色 (本版本未拆分红跨边界)
    cv::inRange(hsv, cv::Scalar(0,  50, 180), cv::Scalar(10, 255, 255), mask1);  // 红
    cv::inRange(hsv, cv::Scalar(10, 50, 180), cv::Scalar(25, 255, 255), mask2);  // 橙
    cv::inRange(hsv, cv::Scalar(25, 50, 180), cv::Scalar(35, 255, 255), mask3);  // 黄
    mask = mask1 | mask2 | mask3;

    pixels = cv::countNonZero(mask);

    // 闪烁判定: 前后帧变化率 > 30%
    int change = abs(pixels - prev_fire);
    prev_fire = pixels;
    float ratio = prev_fire>0 ? (float)change/prev_fire : 0;
    return pixels > 300 && ratio > 0.3;  // 阈值比 auto_nav 更宽松(300 vs 800)
}
```

**与 auto_nav.cpp 中火焰检测的差异**:
| 参数 | fire_test.cpp | auto_nav.cpp |
|------|:-----------:|:----------:|
| 颜色通道数 | 3 (未拆红) | 4 (红拆为0-10+160-179) |
| S 阈值 | >50 | >150 |
| V 阈值 | >180 | >150 |
| 像素阈值 | >300 | >800 |
| 闪烁率阈值 | >30% | >40% |

auto_nav.cpp 的参数更严格，减少误报；fire_test.cpp 更宽松，用于独立测试和调参。

---

## 4.2 smoke_detect.cpp — 香烟检测

**功能概述**: 两阶段检测——先用肤色检测定位手部位置，再在手部附近搜索白色细长圆柱形物体（香烟特征）。

### 两阶段检测架构

```cpp
// 阶段1: 肤色检测找手 (HSV 肤色范围)
static bool detect_hand(cv::Mat &f, cv::Rect &hand_roi){
    cv::Mat hsv, mask;
    cv::cvtColor(f,hsv,cv::COLOR_BGR2HSV);
    // 肤色: H:0-25, S:50-150, V:80-255
    cv::inRange(hsv,cv::Scalar(0,50,80),cv::Scalar(25,150,255),mask);
    // 形态学去噪: 开运算去小噪点 → 膨胀连接碎片
    cv::Mat k=cv::getStructuringElement(cv::MORPH_RECT,cv::Size(5,5));
    cv::morphologyEx(mask,mask,cv::MORPH_OPEN,k);
    cv::dilate(mask,mask,k);
    // 找面积 >800px 的最大轮廓作为手部位置
    std::vector<std::vector<cv::Point>> contours;
    cv::findContours(mask,contours,cv::RETR_EXTERNAL,cv::CHAIN_APPROX_SIMPLE);
    for(auto &c: contours){
        double a=cv::contourArea(c);
        if(a>800){ hand_roi=cv::boundingRect(c); return true; }
    }
    return false;
}

// 阶段2: 在手部ROI内找白色细长圆柱 (香烟特征)
static bool detect_cigarette(cv::Mat &f, cv::Rect hand_roi, std::vector<cv::Rect> &boxes){
    // 扩展搜索区域: 手周围多搜30-50px
    int x=std::max(0,hand_roi.x-30); int y=std::max(0,hand_roi.y-30);
    int w=std::min(f.cols-x,hand_roi.width+60);
    int h=std::min(f.rows-y,hand_roi.height+60);
    cv::Mat roi=f(cv::Rect(x,y,w,h));
    cv::Mat hsv, mask;
    cv::cvtColor(roi,hsv,cv::COLOR_BGR2HSV);
    // 纯白: S<15 (低饱和度) + V>235 (高亮度)
    cv::inRange(hsv,cv::Scalar(0,0,235),cv::Scalar(179,15,255),mask);
    // 找小面积(30-800px)、细长(长宽比>3:1)的轮廓
    std::vector<std::vector<cv::Point>> contours;
    cv::findContours(mask,contours,cv::RETR_EXTERNAL,cv::CHAIN_APPROX_SIMPLE);
    for(auto &c: contours){
        double a=cv::contourArea(c);
        if(a<30||a>800) continue;          // 过滤太大/太小
        cv::RotatedRect rr=cv::minAreaRect(c);
        float rw=rr.size.width, rh=rr.size.height;
        if(rw<3||rh<3) continue;
        float ratio=(rw>rh)?rw/rh:rh/rw;
        if(ratio>3.0){                      // 长宽比>3 = 细长物体
            cv::Rect rb=rr.boundingRect();
            rb.x+=x; rb.y+=y;              // 偏移回原图坐标
            boxes.push_back(rb);
        }
    }
    return !boxes.empty();
}
```

---

# 五、语音与通信程序详解

## 5.1 mp3.cpp — DFPlayer Mini MP3 模块

**功能概述**: DFPlayer Mini 串口 MP3 模块交互式测试程序。通过 UART1 (/dev/ttyS1, 9600bps) 发送 10 字节命令帧，支持键盘选择播放 1-10 号曲目。

### DFPlayer Mini 10 字节命令帧协议

```
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────────┬──────────┬──────┐
│ 0x7E │ 0xFF │ 0x06 │ CMD  │ 0x00 │  DH  │  DL  │ CHECK_H  │ CHECK_L  │ 0xEF │
│ 起始  │ 版本 │ 长度 │ 命令 │ 应答 │数据H │数据L │ 校验高   │ 校验低   │ 结束 │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────────┴──────────┴──────┘

校验和 = - (0xFF + 0x06 + CMD + 0x00 + DH + DL)
        取低16位，CHECK_H = 高8位，CHECK_L = 低8位
```

### 命令码速查

| CMD | 功能 | DH,DL |
|:---:|------|-------|
| 0x03 | 播放指定曲目 | 曲目号(高8位,低8位) |
| 0x04 | 暂停 | 0,0 |
| 0x05 | 继续 | 0,0 |
| 0x06 | 下一曲 | 0,0 |
| 0x07 | 上一曲 | 0,0 |
| 0x09 | 音量设置 | 0,音量(0-30) |
| 0x0C | **硬件复位** | 0,0 |
| 0x0E | 停止播放 | 0,0 |
| 0x0F | 播放指定文件夹曲目 | 文件夹号,曲目号 |

### 命令发送实现

```cpp
static void mp3_cmd(unsigned char cmd, unsigned char dat_h, unsigned char dat_l) {
    int sum = 0xFF + 0x06 + (int)cmd + 0x00 + (int)dat_h + (int)dat_l;
    int chk = -sum;  // 校验和取负
    unsigned char buf[10] = {
        0x7E, 0xFF, 0x06, cmd, 0x00, dat_h, dat_l,
        (unsigned char)((chk>>8)&0xFF),   // CHECK_H
        (unsigned char)(chk&0xFF),         // CHECK_L
        0xEF
    };
    write(uart_fd, buf, 10);
}
```

**初始化序列（关键！）**:
1. `killall -9 getty` — 释放被占用串口
2. `usleep(300000)` — 等串口释放
3. 配置 UART: 9600bps, 8N1
4. `usleep(1500000)` — **等 DFPlayer 上电稳定（最关键的等待）**
5. 发送 `0x0C` 硬件复位 + `usleep(500000)` — **解决模块卡死**
6. 设置音量 25

---

## 5.2 sms_alert.cpp — 语音触发短信报警

**功能概述**: 路人语音唤醒后说"这里着火了"→ SU-03T 发指令码 0x01 → LS2K0300 收到后 → MP3 确认播报 → 4G 模块发短信给管理员。

### 三串口架构

```
SU-03T 语音模块 ──UART2──┐
                          ├── LS2K0300
Air780E 4G模块 ──UART3──┤     │
                          │     ├── 解析指令
DFPlayer MP3 ────UART1───┘     ├── MP3 应答
                               └── AT 命令发短信
```

### AT 指令短信发送流程

```cpp
static int send_sms(const char* phone, const char* msg){
    // 步骤1: 设置文本模式
    gsm_send("AT+CMGF=1");
    gsm_wait("OK",3000);

    // 步骤2: 设置接收号码
    char cmd[64]; snprintf(cmd,sizeof(cmd),"AT+CMGS=\"%s\"",phone);
    gsm_send(cmd);
    gsm_wait(">",5000);  // 等提示符 ">"

    // 步骤3: 发送消息体 + Ctrl+Z(0x1A) 结束
    char full[256];
    snprintf(full,sizeof(full),"%s\r\n\x1A",msg);
    write(gsm_fd,full,strlen(msg)+3);
    gsm_wait("OK",10000);  // 等发送完成
}
```

### 非阻塞语音指令读取

```cpp
static int voice_read_cmd(){
    fd_set fds; FD_ZERO(&fds); FD_SET(voice_fd,&fds);
    struct timeval tv={0,200000};  // 200ms 超时
    if(select(voice_fd+1,&fds,0,0,&tv)>0){
        unsigned char c;
        if(read(voice_fd,&c,1)==1) return c;
    }
    return -1;  // 超时或无数据
}
```

使用 `select()` 而非阻塞 `read()`，保证主循环不被语音模块阻塞——即使没有语音输入，程序也能继续运行其他逻辑。

---

## 5.3 child_call.cpp — 儿童走失呼叫系统

**功能概述**: 走失儿童找到机器人 → 语音唤醒 → 4G 拨号给预存家长号码 → 双向语音通话。

### 通话流程

```
儿童: "小Π小Π,找不到妈妈"
  → SU-03T 识别指令 0x02
  → MP3 播放 "正在帮你联系家长"
  → Air780E: ATD<妈妈号码>;  (分号=语音通话模式)
  → GSM 模块等待接通
    ├── 接通 → MP3 播 "电话已接通" → 双向通话 → 等待挂断
    └── 失败 → MP3 播 "拨号失败,请稍后再试"
  → 检测 NO CARRIER 或用户按 q → ATH 挂断
  → MP3 恢复待机音
```

### 通话状态监听

```cpp
static int wait_hangup(){
    printf("通话中, 按q或等待对方挂断...\n");
    char buf[256]={0}; int t=0;
    while(1){
        // 检查 GSM 模块返回的状态字
        usleep(100000);
        char tmp[32]={0}; int n=read(gsm_fd,tmp,31);
        if(n>0){
            strncat(buf,tmp,sizeof(buf)-t-1); t+=n;
            if(strstr(buf,"NO CARRIER")||strstr(buf,"BUSY"))
                return 1;  // 对方挂断或占线
        }
        // 同时检查键盘: q键主动挂断
        fd_set fds; FD_ZERO(&fds); FD_SET(0,&fds);
        struct timeval tv={0,50000};
        if(select(1,&fds,0,0,&tv)>0){
            char c=getchar(); if(c=='\n')continue;
            if(c=='q'||c=='Q') return 2;  // 主动结束
        }
    }
}
```

**双监听机制**: 同时监听 GSM 模块的 `NO CARRIER`（对方挂断）和键盘输入（主动挂断），确保通话结束能被及时检测到。

---

# 六、环境交互程序详解

## 6.1 quiet_remind.cpp — 噪音检测与静音提醒

**功能概述**: KY-038 声音传感器检测噪音 >45dB → Haar 视觉定位发声者 → 驱动小车靠近 → MP3 播报"请保持安静"。

### 硬件防抖 — 10 采 6 中

```cpp
static int detect_noise(){
    int hits=0;
    for(int i=0;i<10;i++){
        if(is_noisy()) hits++;    // 读 KY-038 DO 脚
        usleep(50000);            // 50ms 采样间隔 = 500ms 总窗口
    }
    return hits>=6;  // 10次中 ≥6次超阈值 → 真噪音（避免瞬时噪声误触发）
}
```

**防抖设计原因**: 图书馆环境中可能有书本掉落、椅子拖动等瞬时声响。10 次采样取 6 次以上才判定为真噪音，有效过滤瞬时冲击噪声。

### 发声者定位与靠近

```cpp
// 找画面中最大的人框中心
static int find_person(){
    cv::Mat f;
    { std::lock_guard<std::mutex> l(lock); f=shared.clone(); }
    std::vector<cv::Rect> found;
    bc.detectMultiScale(f,found,1.08,2,0,cv::Size(20,30));
    if(found.empty()) return -1;
    cv::Rect best=found[0];
    for(auto &r:found) if(r.area()>best.area()) best=r;
    return best.x+best.width/2;  // 返回人框中心 X 坐标
}

// 按偏移量转向: 人在左边→左转, 在右边→右转
int off=px-160;
if(off<-40)      { go(3); usleep(200000); }      // 左转
else if(off>40)  { go(4); usleep(200000); }      // 右转
else printf("正前\n");
mp3_play(7);     // "请保持安静"
go(1); usleep(2000000);  // 前进 2 秒靠近
```

### 冷却机制

```cpp
if(now-last_quiet<15){ usleep(100000); continue; }  // 15秒冷却
```

15 秒内不重复触发，避免对同一噪音源反复提醒。

---

# 七、定位与导航程序详解

## 7.1 visual_odo.cpp — 视觉里程计

**功能概述**: 纯视觉定位测试程序，通过光流跟踪 FAST 角点推算航向角和行驶轨迹。

### 算法流程

```
摄像头采集 320×240 灰度帧
    ↓
goodFeaturesToTrack() 检测 100 个 FAST 角点
    ↓
calcOpticalFlowPyrLK() 金字塔光流跟踪上一帧角点
    ↓
统计成功跟踪点的平均水平和竖直位移
    ↓
avg_dx → 水平位移 × 标定系数 → 航向角变化
avg_dy → 竖直位移 × 前进步长 → 位置更新
    ↓
累积 heading, pos_x, pos_y → 轨迹输出
```

### 关键代码

```cpp
// 检测新特征点
std::vector<cv::Point2f> new_pts;
cv::goodFeaturesToTrack(gray,new_pts,100,0.01,10);

// 光流跟踪
cv::calcOpticalFlowPyrLK(prev_gray,gray,prev_pts,tracked,status,err);

// 统计
for(size_t i=0;i<status.size();i++){
    if(!status[i]) continue;
    double dx=tracked[i].x-prev_pts[i].x;
    double dy=tracked[i].y-prev_pts[i].y;
    if(fabs(dx)>50||fabs(dy)>50) continue;  // 过滤异常
    sum_dx+=dx; sum_dy+=dy; count++;
}

if(count>5){
    double turn_deg=avg_dx*0.05;         // 标定系数 0.05°/像素
    heading+=turn_deg;
    double fwd_cm=avg_dy*0.1;            // 竖直位移→前进距离
    pos_x+=fwd_cm*cos(heading);
    pos_y+=fwd_cm*sin(heading);
}
```

---

# 八、基础测试程序详解

## 8.1 电机与运动测试

### motor_test.cpp — 电机时序测试
最简单的电机功能验证: 前进 2s → 左转 1s → 右转 1s → 后退 2s。不包含 PWM（全速运行）。

### straight.cpp — 纯直线前进
50% PWM 占空比纯直线前进测试，Ctrl+C 停止。最简代码量验证 PWM 降速效果。

### turn_straight.cpp — 转弯专项测试
直走 3s → 右转 1.3s → 持续直走 (30% PWM)。测试转弯角度是否准确。

## 8.2 传感器测试

### sensor_test.cpp — 超声波+舵机联合测试
HC-SR04 + MG996R 360° 舵机联合调试程序。使用 MG996R 360° 连续旋转舵机的速度控制 API：
```cpp
static void servo_stop(){ servo_pulse(1500); }
static void servo_cw(int s){ servo_pulse(1500 - s*5); }   // s:0-100, 越低越快
static void servo_ccw(int s){ servo_pulse(1500 + s*5); }  // s:0-100, 越高越快
```

### sg90_test.cpp — SG90 位置舵机平滑扫描
0°→180°→0° 平滑扫角度测试，每个角度驻留 15 个脉冲（约 0.3 秒）。验证 SG90 位置舵机工作正常。

## 8.3 舵机校准工具

### servo_cal.cpp — MG996R 360° 停止点校准
1300→1700µs，步进 5µs，每步持续 2 秒。操作者观察舵机找到完全停止的脉冲宽度。

### servo_tune.cpp — 左右平衡校准
交替左转/右转，手动调整 `left_us` 和 `right_us` 直到两边转动角度对称。解决山寨 MG996R CW/CCW 速度不对称问题。

### servo_spin.cpp — 最简持续旋转
GPIO67 输出 1100µs 脉冲让舵机持续 CW 转，验证舵机能转的基础测试。

## 8.4 通信测试

### uart2_test.cpp — US-100 UART 模式 + pinmux 配置
通过 `/dev/mem` 内存映射配置 GPIO44/45 的 pinmux 寄存器为 UART2 主功能，然后用 UART 模式读 US-100 (发 0x55 返回距离 mm)。

**pinmux 操作原理**:
```cpp
// 内存映射访问 GPIO pinmux 寄存器
volatile unsigned int *reg=(volatile unsigned int*)mmap(0,4096,
    PROT_READ|PROT_WRITE,MAP_SHARED,fd,0x16000000);
// GPIO44/45 pinmux @ 0x16000498, bits 25:24 和 27:26
unsigned int val=reg[0x498/4];
val &= ~(0xF<<24);      // 清除旧值
val |= (0xF<<24);       // 设为 11=UART2 主功能
reg[0x498/4]=val;
```

### i2c_free.cpp — GPIO50/51 pinmux 释放
将 GPIO50/51 从 I2C1 主功能释放为普通 GPIO。通过写 pinmux 寄存器 `0x1600049c` 的 bits 4-7 为 0。

### mpu_test.cpp — MPU6050 软件 I2C 测试
在 GPIO72(SCL) + GPIO44(SDA) 上实现软件 I2C 协议，扫描 0x68/0x69 地址读取 MPU6050 的 WHO_AM_I 寄存器。

**软件 I2C 实现**:
```cpp
#define T 500  // 时钟周期 500µs ≈ 1kHz I2C
static void i2c_start(){ sda_w(1); scl(1); du(T); sda_w(0); du(T); scl(0); }
static void i2c_stop() { sda_w(0); scl(1); du(T); sda_w(1); du(T); }
static int i2c_write(unsigned char b){
    for(int i=7;i>=0;i--){ sda_w(b>>i&1); du(T/5); scl(1); du(T); scl(0); }
    // 读 ACK
    sda_w(1); sda_in(); scl(1); du(T); int ack=!sda_r(); scl(0); du(T);
    return ack;
}
```

## 8.5 MJPEG 监控测试

### mjpeg.cpp — 视频流 + 人体检测 + MP3 联动
最早的视频监控程序，MJPEG 推流到 8080 端口，Haar 人腿检测，检测到人自动切换 MP3 播放（有人→0002，无人→0001）。

**状态变化触发** (而非持续触发):
```cpp
int want_play = (g_human_count > 0) ? 2 : 1;
if (want_play != g_last_play && now - g_last_cmd > 3) {
    mp3_play(want_play);  // 只在状态变化时发一次命令
    g_last_play = want_play;
    g_last_cmd = now;
}
```

---

# 九、Seekfree 官方库文件详解

LS2K300_Library 项目包含两个主要部分：**用户项目代码**（project/user/，前三~八章已详述）和 **Seekfree 逐飞科技官方开源库**。

官方库位于 `Seekfree_LS2K0300_Opensource_Library/libraries/`，分四层：
- **zf_driver/** — 底层驱动（GPIO/PWM/ADC/定时器/编码器/网络）
- **zf_device/** — 设备驱动（IMU/激光测距/显示屏/摄像头）
- **zf_common/** — 通用工具（类型定义/FIFO/字体/字符串转换）
- **zf_components/** — 应用组件（PC 调试助手协议/机器学习推理）

## 9.1 zf_driver — 底层驱动库（11 个模块）

所有驱动类均使用 RAII 模式（构造获取资源，析构释放），禁止拷贝（`= delete`）。

### 9.1.1 zf_driver_adc — ADC 模数转换

**文件**: `zf_driver_adc.hpp` + `zf_driver_adc.cpp`

基于 Linux sysfs 的 ADC 读取接口，使用原始文件描述符（无 C 标准 IO 缓冲）。

```
类: zf_driver_adc
├── 构造: zf_driver_adc(adc_path, mode)
├── convert() → uint16    — 读取 ADC 原始值
└── get_scale() → float   — 读取校准系数

路径宏:
  ADC_CH0_PATH ~ ADC_CH7_PATH → /sys/bus/iio/devices/iio:device0/in_voltageN_raw
  ADC_SCALE_PATH             → in_voltage_scale

关键实现: lseek(fd,0,SEEK_SET) + read() + atoi()
          每次读取前必须 lseek 回文件开头（避免读到旧值）
```

### 9.1.2 zf_driver_delay — 延时宏

**文件**: `zf_driver_delay.hpp` + `zf_driver_delay.cpp`

最简模块，封装 Linux `usleep()` 系统调用：
```cpp
#define system_delay_ms(time)  (usleep(time*1000))
#define system_delay_us(time)  (usleep(time))
```

### 9.1.3 zf_driver_encoder — 编码器

**文件**: `zf_driver_encoder.hpp` + `zf_driver_encoder.cpp`

继承 `zf_driver_file_buffer`。读写设备节点获取电机转速/位置。
```
get_count() → int16     — 读取累计脉冲数
clear_count() → void    — 写 0 清零计数

路径宏: ZF_ENCODER_QUAD_1/2 (正交编码, 默认), ZF_ENCODER_DIR_1/2 (方向编码, 需改DTS)
```

### 9.1.4 zf_driver_file_buffer — 二进制文件 IO 基类

**文件**: `zf_driver_file_buffer.hpp` + `zf_driver_file_buffer.cpp`

encoder/gpio/pwm 的基类。提供 POSIX fd 级别的读写：
```
read_buff(buf, len) → int8   — 读满 len 字节才返回 0（全或无语义）
write_buff(buf, len) → int8  — 写满 len 字节才返回 0

关键: 构造函数按 flags 决定是否 O_CREAT|O_TRUNC
      只读模式不创建文件，读写/只写模式自动创建
```

### 9.1.5 zf_driver_file_string — 文本文件 IO

**文件**: `zf_driver_file_string.hpp` + `zf_driver_file_string.cpp`

使用 C `FILE*` 的文本读写，适用于 sysfs 文本接口：
```
read_string(str)  → int8   — fscanf(fp, "%s", str) 读一个空白分隔字符串
write_string(str) → int8   — fputs(str, fp) 写字符串
rewind_file()     → void   — rewind(fp) 重置文件位置
```

### 9.1.6 zf_driver_gpio — GPIO 控制

**文件**: `zf_driver_gpio.hpp` + `zf_driver_gpio.cpp`

继承 `zf_driver_file_buffer`。ASCII 编码的 GPIO 电平控制：
```
set_level(level) → void   — 写 '0'(0x30) 或 '1'(0x31)
get_level() → uint8       — 读 1 字节返回 ASCII 值

路径宏: ZF_GPIO_BEEP, ZF_GPIO_HALL_DETECTION, ZF_GPIO_KEY_1~4,
        ZF_GPIO_MOTOR_1/2
```

### 9.1.7 zf_driver_pit — 硬实时周期定时器

**文件**: `zf_driver_pit.hpp` + `zf_driver_pit.cpp`

基于 `timerfd` + `epoll` + `pthread` + `SCHED_FIFO` 优先级 99 的硬实时定时器。**这是整个库最核心的定时基础**。

```
init_ms(period_ms, callback) → int
  └── 1. timerfd_create(CLOCK_MONOTONIC)
      2. timerfd_settime() 配置周期
      3. epoll_create1() + epoll_ctl(EPOLLIN|EPOLLET) 边缘触发
      4. pthread_create() 运行 pit_timer_thread
      5. pthread_setschedparam(SCHED_FIFO, 99)

pit_timer_thread():
  while(!exit):
    epoll_wait(-1) 阻塞等待（0% CPU 空闲）
    → read(timerfd) 消费事件
    → 调用用户回调

stop(): 设置退出标志 → join 线程 → 关闭 fd
```

### 9.1.8 zf_driver_pit_fd — C++ 风格定时器

**文件**: `zf_driver_pit_fd.hpp` + `zf_driver_pit_fd.cpp`

使用 `std::function` 回调 + `std::thread` 的 C++ 封装：
```
类: timer_fd
├── 构造: timer_fd(interval_ms, std::function<void()>)
├── start() → void    — 创建 timerfd + 线程
├── stop()  → void    — 停止并 join

与 zf_driver_pit 的差异:
  - 使用阻塞 read() 而非 epoll（更简单）
  - 优先级 10（低于 pit 的 99）
  - 处理定时器溢出（missed ticks 补偿回调）
```

### 9.1.9 zf_driver_pwm — PWM 输出

**文件**: `zf_driver_pwm.hpp` + `zf_driver_pwm.cpp`

继承 `zf_driver_file_buffer`。二进制读写 PWM 设备节点：
```
get_dev_info(pwm_info) → void  — 读取频率/最大占空比/时钟等
set_duty(duty) → void          — 写 uint16 占空比值

路径宏: ZF_PWM_ESC_1(电调), ZF_PWM_MOTOR_1/2, ZF_PWM_SERVO_1
```

### 9.1.10 zf_driver_tcp_client — TCP 客户端

**文件**: `zf_driver_tcp_client.hpp` + `zf_driver_tcp_client.cpp`

非阻塞 TCP + 重试机制的网络客户端：
```
init(ip, port) → int8           — socket+connect+设为非阻塞
send_data(buf, len) → uint32    — 循环 send, EAGAIN时等待重试(最大100次×10ms)
read_data(buf, len) → uint32    — 非阻塞 recv, EAGAIN时返回0

重试策略: max_retry×retry_interval = 100×10ms = 最长 1 秒阻塞
```

### 9.1.11 zf_driver_udp — UDP 客户端

**文件**: `zf_driver_udp.hpp` + `zf_driver_udp.cpp`

非阻塞 UDP 通信（无连接）：
```
init(ip, port) → int8           — socket+存储目标地址（不 connect）
send_data(buf, len) → uint32    — sendto() 直接发包
read_data(buf, len) → uint32    — recvfrom(MSG_DONTWAIT) 非阻塞接收
                                   EAGAIN时静默返回0（避免日志污染）
```

---

## 9.2 zf_device — 设备驱动库（5 个模块）

### 9.2.1 zf_device_dl1x — DL1X 激光测距传感器

**文件**: `zf_device_dl1x.hpp` + `zf_device_dl1x.cpp`

继承 `zf_driver_file_string`。通过 Linux IIO 子系统读取距离数据。

```
类: zf_device_dl1x
├── init() → dl1x_device_type_enum  — 自动检测 DL1A (1) vs DL1B (2)
├── get_distance() → int16           — 读取原始距离值
└── get_dev_type() → enum            — 获取设备型号

sysfs 路径:
  DL1X_EVENT_PATH     = /sys/bus/iio/devices/iio:device2/events/in_voltage_change_en
  DL1X_DISTANCE_PATH  = /sys/bus/iio/devices/iio:device2/in_distance_raw

初始化流程: 打开 event 路径 → write('1') 触发硬件初始化
          → read 回读检测型号 → 打开 distance 路径
```

### 9.2.2 zf_device_imu — IMU 惯性测量单元

**文件**: `zf_device_imu.hpp` + `zf_device_imu.cpp`

继承 `zf_driver_file_string`。支持自动识别 4 种传感器型号。

```
类: zf_device_imu
├── init() → imu_device_type_enum
│   └── 自动识别: IMU660RA/IMU660RB/IMU660RC (6轴) vs IMU963RA (9轴)
├── get_acc_x/y/z()  → int16  — 加速度计 (3 轴)
├── get_gyro_x/y/z() → int16  — 陀螺仪 (3 轴)
└── get_mag_x/y/z()  → int16  — 磁力计 (3 轴，仅 IMU963RA；其他返回0)

sysfs 路径 (9 个):
  /sys/bus/iio/devices/iio:device1/in_accel_x/y/z_raw
  /sys/bus/iio/devices/iio:device1/in_anglvel_x/y/z_raw
  /sys/bus/iio/devices/iio:device1/in_magn_x/y/z_raw  (仅9轴)

关键: 磁力计 fd 仅在检测到 IMU963RA 时才打开
```

### 9.2.3 zf_device_ips200_fb — IPS200 显示屏 (2.0寸, 240×320)

**文件**: `zf_device_ips200_fb.hpp` + `zf_device_ips200_fb.cpp`

独立的帧缓冲(framebuffer)驱动类，通过 `mmap` 映射 `/dev/fb0` 直接写像素。

```
类: zf_device_ips200
├── init(path, is_reload_driver)
│   └── rmmod fb_st7789v → insmod → open(/dev/fb0) → ioctl 读分辨率 → mmap → clear
├── 基本绘图:
│   ├── draw_point(x, y, color)       — 单像素写入 screen_base[y*w+x]
│   ├── draw_line(x1,y1, x2,y2, color) — 点斜式画线
│   ├── clear()                        — 填充默认背景色
│   └── full(color)                    — 填充指定颜色
├── 文字渲染:
│   ├── show_char(x, y, ch)            — 8×16 ASCII 字符 (ascii_font_8x16)
│   ├── show_string(x, y, str)         — 字符串
│   ├── show_int/uint/float(x,y,val,n) — 格式化数值显示
│   └── show_wave(x,y,*data,w,max,...) — 波形图绘制
├── 图像显示:
│   ├── show_gray_image(...)           — 8位灰度图 (支持二值化阈值)
│   ├── show_rgb565_image(...)         — 16位 RGB565 (支持字节序交换)
│   ├── displayimage_gray(*p,w,h)      — 便捷灰度显示
│   └── displayimage_rgb565(*p,w,h)    — 便捷 RGB565 显示
└── 内核模块: fb_st7789v.ko
```

### 9.2.4 zf_device_tft180_fb — TFT180 显示屏 (1.8寸, 128×160)

**文件**: `zf_device_tft180_fb.hpp` + `zf_device_tft180_fb.cpp`

API 与 IPS200 几乎完全相同，差异仅在：
- 分辨率: 128×160（vs 240×320）
- 内核模块: `fb_st7735r.ko`（vs `fb_st7789v.ko`）

**注意**: `full()` 函数内的循环边界硬编码 240×320 是已知 bug（从 IPS200 复制时未修改），但 `init()` 中的清屏使用了正确的 `this->width` 和 `this->height`。

### 9.2.5 zf_device_uvc — USB 摄像头 (UVC)

**文件**: `zf_device_uvc.hpp` + `zf_device_uvc.cpp`

基于 OpenCV `cv::VideoCapture` 的摄像头驱动，支持格式转换和曝光控制。

```
类: zf_device_uvc
├── init(path="/dev/video0")
│   └── cap.open() → 配置 FOURCC(YUY2或MJPG) → 设分辨率+FPS → 开自动曝光
├── wait_image_refresh() → int8       — 阻塞读一帧 (cap >> frame)
├── get_gray_image_ptr() → uint8_t*   — BGR→灰度转换后返回指针
├── get_rgb_image_ptr() → uint16_t*   — BGR→BGR565 转换后返回指针
├── get_frame_mjpg() → cv::Mat        — 返回当前帧副本
├── is_camera_opened() → bool
├── set_auto_exposure(mode) → void    — 模式: 1=手动, 3=自动 (龙芯特有值)
├── set_exposure_value(val) → int8    — 手动曝光值设置
└── get_auto_exposure_mode() / get_current_exposure()

编译开关:
  UVC_USE_MJPG = 0  — YUY2 格式; =1 — MJPG 格式

默认参数: 160×120 @ 180fps
```

---

## 9.3 zf_common — 通用工具库（4 个模块）

### 9.3.1 zf_common_typedef — 统一类型定义

**文件**: `zf_common_typedef.hpp`

整个库的基石。包含：
- **标准 C 头文件**: stdio, stdlib, string, stdint, stdbool, stdarg, unistd, errno
- **Linux 系统头文件**: fcntl, sys/mman, sys/ioctl, pthread, sched, sys/prctl, signal, sys/timerfd, sys/epoll, linux/fb, sys/socket, netinet/in, arpa/inet
- **C++ 标准库**: iostream, functional, atomic, chrono, thread, cstring, cstdlib, cerrno
- **类型别名**: `uint8`(uchar), `uint16`(ushort), `uint32`(uint), `int8`(schar), `int16`(sshort), `int32`(int) 及对应的 `vuint8`/`vint8` 等 volatile 版本
- **弱符号宏**: `#define ZF_WEAK __attribute__((weak))` 允许用户覆盖库函数

### 9.3.2 zf_common_fifo — 环形 FIFO 缓冲区

**文件**: `zf_common_fifo.hpp` + `zf_common_fifo.cpp`

适用于中断安全场景的泛型环形缓冲区（支持 8/16/32 位数据类型）。

```
数据结构:
  fifo_struct {
    execution: uint8      — 操作标志位 (防止中断嵌套破坏)
    type: data_type_enum  — FIFO_DATA_8BIT/16BIT/32BIT
    *buffer: void*        — 用户提供的存储空间
    head: uint32          — 写指针 (指向空位)
    end:  uint32          — 读指针 (指向数据)
    size: uint32          — 剩余空闲容量
    max:  uint32          — 总容量
  }

API:
  fifo_init(fifo, type, buffer, size)  — 绑定缓冲区并初始化
  fifo_write_element(fifo, data)       — 写入单个元素
  fifo_write_buffer(fifo, data, len)   — 批量写入 (处理回绕)
  fifo_read_element(fifo, &data, flag) — 读取单个 (可选消费)
  fifo_read_buffer(fifo, data, &len)   — 批量读取 (处理回绕)
  fifo_read_tail_buffer(...)           — 读最近写入的数据
  fifo_clear(fifo)                     — memset 清零整缓冲区
  fifo_used(fifo) → uint32             — 已存储元素数

读取模式:
  FIFO_READ_AND_CLEAN — 读后从缓冲区移除
  FIFO_READ_ONLY      — 仅查看不消费

执行标志位 (防止 ISR 重入):
  FIFO_IDLE(0x00) | FIFO_RESET(0x01) | FIFO_CLEAR(0x02)
  | FIFO_WRITE(0x04) | FIFO_READ(0x08)
```

### 9.3.3 zf_common_font — 字体与颜色

**文件**: `zf_common_font.hpp` + `zf_common_font.cpp`

包含完整的字体位图数据和 RGB565 颜色定义。

```
颜色枚举 (rgb565_color_enum):
  RGB565_WHITE(0xFFFF), RGB565_BLACK(0x0000), RGB565_RED(0xF800),
  RGB565_GREEN(0x07E0), RGB565_BLUE(0x001F), RGB565_YELLOW(0xFFE0),
  RGB565_CYAN(0x07FF), RGB565_MAGENTA(0xF81F), RGB565_GRAY(0x8430),
  RGB565_BROWN(0xBC40), RGB565_PINK(0xFE19) 等

字体位图:
  ascii_font_8x16[95][16]    — 8×16 点阵 ASCII (空格到~，每字符16字节)
  ascii_font_6x8[91][6]      — 6×8 点阵 ASCII
  chinese_test[4][16]         — 4个中文("逐飞科技") 16×16行优先
  oled_16x16_chinese[4][16]   — 同上但列优先+反序 (适配OLED)

图像:
  gImage_seekfree_logo[38400] — 240×80 RGB565 公司 logo
```

### 9.3.4 zf_common_function — 工具函数

**文件**: `zf_common_function.hpp` + `zf_common_function.cpp`

数学工具 + 自定义轻量 `sprintf`。

```
数值裁剪宏:
  func_abs(x)           — 绝对值
  func_limit(x, y)      — 对称限制: [-y, +y]
  func_limit_ab(x, a, b)— 非对称限制: [a, b]

数学函数:
  func_get_greatest_common_divisor(a,b) — GCD（更相减损术，无除法）
  func_soft_delay(t)                    — 忙等延时

字符串转换（均自实现，不依赖 libc）:
  func_str_to_int/int_to_str   — 有符号整数 (范围 ±32767)
  func_str_to_uint/uint_to_str — 无符号整数 (范围 0-65535)
  func_str_to_float/float_to_str — 浮点数 (6位小数)
  func_str_to_double/double_to_str — 双精度 (9位小数)
  func_str_to_hex/hex_to_str   — 十六进制 (支持0x前缀)

zf_sprintf(buf, format, ...):
  自定义轻量级 sprintf 实现。
  支持格式: %c %d/%i %u %o %x/%X %s %p %f/%F %%
  不支持: 宽度/精度修饰符, %a
```

### 9.3.5 zf_common_headfile — 统一头文件

**文件**: `zf_common_headfile.hpp`

包含所有库头文件的便捷入口。按层级组织：
```
zf_common → zf_driver → zf_device → zf_components → ncnn → OpenCV → TFLM
```

---

## 9.4 zf_components — 应用组件（2 个模块）

### 9.4.1 seekfree_assistant — PC 调试助手协议

**文件**: `seekfree_assistant.hpp` + `seekfree_assistant.cpp`

与 Seekfree Assistant PC 工具通信的二进制协议栈。支持三大功能：

**功能 1: 虚拟示波器**（最多 8 通道 float 数据流）
```
seekfree_assistant_oscilloscope_send()
└── 构建包头(0xAA|通道数) → 发 float[] 数据 → 附加校验和
```

**功能 2: 摄像头图像传输**（支持灰度/RGB565/二值）
```
seekfree_assistant_camera_information_config()  — 配置图像参数
seekfree_assistant_camera_boundary_config()     — 配置车道线叠加数据
seekfree_assistant_camera_send()
├── 发送图像帧头(类型+宽高+长度)
├── 发送原始像素数据 (灰度:W*H / RGB565:W*H*2 / 二值:W*H/8)
└── 可选发送边界/车道线坐标 (X/Y/XY 三种模式)
```

**功能 3: 参数在线调优**（PC→MCU 反向通道）
```
seekfree_assistant_data_analysis()
└── 从 FIFO 解析 0x55 帧头 → 校验和验证 → 提取 float 参数
    → 存入 seekfree_assistant_parameter[] → 置位 update_flag[]
    (仅当 SEEKFREE_ASSISTANT_SET_PARAMETR_ENABLE=1 时编译)
```

### 9.4.2 seekfree_assistant_interface — 通信传输抽象

**文件**: `seekfree_assistant_interface.hpp` + `seekfree_assistant_interface.cpp`

将协议层与具体通信方式解耦：
```
seekfree_assistant_interface_init(send_callback, recv_callback)
└── 注册用户提供的发送/接收函数指针

支持的传输方式:
  SEEKFREE_ASSISTANT_DEBUG_UART    — 调试串口
  SEEKFREE_ASSISTANT_WIRELESS_UART — 无线串口
  SEEKFREE_ASSISTANT_WIFI_SPI      — 高速 WiFi SPI
  SEEKFREE_ASSISTANT_CUSTOM        — 用户自定义回调 (最灵活)

默认弱实现:
  seekfree_assistant_transfer() → 返回 len (空操作，用户覆盖)
  seekfree_assistant_receive()  → 返回 0   (无数据，用户覆盖)
```

---

# 十、Example 示例程序详解

示例程序位于 `LS2K300_Library/Example/`，分为核心板演示和主板演示两大类。

## 10.1 主板示例 (Motherboard_Demo) — 9 大类别

### E1 主板基础外设

| 示例 | 文件 | 硬件 | 功能 |
|------|------|------|------|
| E01_01 | `button_switch_buzzer_demo` | 4 按键 (GPIO77-80) | GPIO 输入读取 |
| E01_02 | `beep_demo` | 蜂鸣器 (GPIO26) | GPIO 输出 + 间歇鸣叫 |
| E01_03 | `adc_battery_voltage_demo` | ADC_CH7 | 电池电压检测 (11:1分压) |
| E01_04 | `led_blink_demo` | 板载S_LED (GPIO83) | 原生 sysfs GPIO 控制 + atexit 清理 |

### E2 编码器

| 示例 | 硬件 | 功能 |
|------|------|------|
| E02_01 | 方向编码器 (GPIO28/29, 34/35) | PIT 10ms 定时读取→打印 RPM |
| E02_02 | 正交编码器 (同引脚) | 同上但用 quadrature 模式 (默认) |

### E3 电机控制

| 示例 | 硬件 | 功能 |
|------|------|------|
| E03_01 | DRV8701E + 双直流电机 | 三角波占空比 0-30% 循环调速 |
| E03_02 | 舵机 (GPIO86, 50-300Hz) | 75°↔105° 往复扫描 |
| E03_03 | 无刷电调 (GPIO89, 50Hz) | 0-30% 油门循环 |

### E4 IMU 惯性传感器

| 示例 | 硬件 | 功能 |
|------|------|------|
| E04_01 | IMU963RA/660RA/660RB | 自动识别→10Hz 输出加速度/陀螺仪/磁力计数据 |

### E5 显示

| 示例 | 硬件 | 功能 |
|------|------|------|
| E05_01 | IPS200 (240×320, ST7789V) | 全功能演示: logo+文字+波形+画线+纯色填充 |
| E05_02 | TFT180 (128×160, ST7735R) | 同上 + 分辨率适配 |
| E05_03 | IPS200 + UVC 摄像头 | 实时摄像头预览 (灰度/RGB565 交替) |
| E05_04 | UVC 摄像头 | FPS 测量 (std::chrono) |

### E6 无线通信

| 示例 | 硬件 | 功能 |
|------|------|------|
| E06_01 | WiFi + UDP | UDP 客户端 echo 回环 |
| E06_02 | WiFi + TCP | TCP 客户端 echo 回环 |
| E06_03 | WiFi + TCP | **虚拟示波器**: 4 通道递增波形发 PC |
| E06_04 | WiFi + TCP + UVC | **灰度图像传输**: 带车道线叠加 |
| E06_05 | WiFi + TCP + UVC | **RGB565 图像传输**: 字节序交换+车道线 |

### E7 测距

| 示例 | 硬件 | 功能 |
|------|------|------|
| E07_01 | DL1A/DL1B ToF (I2C4) | 自动识别型号→100ms 周期测距打印象 |

### E8 摄像头

| 示例 | 功能 |
|------|------|
| E08_01~05 | 与 E6_04~05 和 E5_03~04 相同 (摄像头专项分组) |

### E9 TensorFlow Lite Micro

| 示例 | 硬件 | 功能 |
|------|------|------|
| E09_01 | CPU 推理 | 静态 JPEG 分类 (40×40→3 类) |
| E09_02 | CPU + UVC | 实时摄像头推理 (40×40→3 类, 每10帧输出) |

**TFLM 配置**: 20 个 Op (Conv2D, DepthwiseConv2D, MaxPool2D, FC, Relu6, Softmax 等) + 128KB tensor arena。

---

## 10.2 核心板示例 (Coreboard_Demo)

| 示例 | 功能 | 关键技术 |
|------|------|----------|
| E07_pit | PIT 定时器基础 | 10ms 回调打印 + SIGINT 信号处理 + atexit 清理 |
| E08_flash | 文件存储 | zf_driver_file_string 读写 int/float 参数 |

---

# 十二、核心技术原理深度解析

## 9.1 sysfs GPIO 性能特征

| 操作 | 耗时 | 说明 |
|------|:---:|------|
| fopen+fprintf+fclose | ~200µs | 最慢，每次打开/关闭文件 |
| open+lseek+write (持久fd) | ~80µs | 快 2.5 倍 |
| open+lseek+read (持久fd) | ~15µs | 读取略快 |
| usleep() | 精度 ~10ms | 受 HZ=100 限制 |
| clock_gettime 忙等 | 精度 ~1µs | 适用微秒级时序 |

**教训**: sysfs GPIO 的 ~15µs 读取延迟意味着 HC-SR04 在 2cm 处（ECHO 脉宽仅 150µs）无法稳定测量——150µs 内只能采样约 10 次。这是 Linux sysfs 架构的物理限制，代码无法弥补。

## 9.2 L298N BJT H 桥压降

```
电池 7.5V → L298N → 电机
                  ↓
        上桥 Vbe ≈ 0.7V (驱动饱和)
        下桥 Vce ≈ 1.0-1.3V
        ────────────────
        总压降 ≈ 1.7-2.0V
        电机实得 ≈ 5.5V

电压对照表:
┌──────────┬──────────┬──────────┬──────────┐
│ 电池电压  │ 电机实得  │ 占额定6V │ 状态     │
├──────────┼──────────┼──────────┼──────────┤
│ 6.0V     │ ~4.0V    │ 67%      │ 动力不足 │
│ 7.4V     │ ~5.4V    │ 90%      │ ✅推荐   │
│ 8.4V(满) │ ~6.4V    │ 107%     │ ✅安全   │
│ 12.0V    │ ~10.0V   │ 167%     │ ❌烧电机  │
└──────────┴──────────┴──────────┴──────────┘
```

## 9.3 100Hz 软件 PWM 原理

```
持久 fd + clock_gettime 忙等的三要素:
1. 独立线程: 不与主循环共享 CPU，保证 PWM 时序
2. 原子变量: std::atomic<bool> 线程安全切换状态
3. 忙等延时: du() 精度 ~1µs，usleep() 精度 ~10ms

100Hz = 10ms 周期:
  ┌─────┐          ┌─────┐
  │ ON  │   OFF    │ ON  │   OFF
──┘4ms  └──6ms────┘4ms  └──6ms───
  ←──────── 10ms ────────→

40% 占空比 → 电机有效电压 = 电池电压 × 40%
           → 速度降为全速的 ~60%
```

## 9.4 DFPlayer Mini 初始化时序

```
上电 → 1.5s 等待 → 硬件复位 0x0C → 0.5s 等待 → 可用
                                         ↑
                                   最关键的一步！
                         解决蓝灯亮但不播放的问题
```

## 9.5 火焰检测 HSV 颜色空间

```
HSV 颜色模型:
  H (Hue):        0°=红  60°=黄  120°=绿  180°=青/红
  S (Saturation): 0=灰  255=纯色
  V (Value):      0=黑  255=最亮

OpenCV 编码:
  H: 0-179 (180=360°/2)
  S: 0-255
  V: 0-255

火焰特征:
  ┌─────────┬────────────┬──────────┬──────────┐
  │ 颜色    │ H 范围     │ S 范围   │ V 范围   │
  ├─────────┼────────────┼──────────┼──────────┤
  │ 深红    │ 0-10       │ >150     │ >150     │
  │ 红(补)  │ 160-179    │ >150     │ >150     │
  │ 橙      │ 10-25      │ >150     │ >150     │
  │ 黄      │ 25-35      │ >150     │ >150     │
  └─────────┴────────────┴──────────┴──────────┘

关键区分点: 闪烁特征
  真火: 帧间像素变化率 > 40% (火焰本质上是动态的)
  干扰: 人手、灯光、红色物体 — 变化率 < 20%
```

## 9.6 Haar 级联 vs HOG

| | HOG (Histogram of Oriented Gradients) | Haar 级联 |
|------|:--:|:--:|
| 检测原理 | 梯度方向直方图 + SVM | 矩形特征 + AdaBoost 级联 |
| CPU 耗时/帧 | >500ms | <50ms |
| LS2K0300 可用 | ❌ 太慢 | ✅ 实时 |
| 模型大小 | ~1MB | 395KB |
| 适用场景 | GPU/NPU 平台 | MCU/Linux 低算力平台 |

**关键教训**: 在算力受限的嵌入式平台上，**确定性检测**（Haar 级联、颜色阈值）远优于**概率性感知**（梯度密度、纹理分析）。

---

# 十三、DFPlayer Mini TF 卡音频配置

TF 卡需包含以下 MP3 文件。**注意**: DFPlayer Mini 按文件拷贝到 TF 卡的先后顺序播放，不是按文件名！必须逐个拷贝确保顺序正确。

| 文件名 | 用途 | 触发场景 |
|--------|------|----------|
| 0001.mp3 | 正常巡检背景音 | 每 13 秒循环播放 |
| 0002.mp3 | 人腿检测报警 | 检测到读者滞留 |
| 0003.mp3 | 火焰报警 | 视觉检测到火焰 |
| 0004.mp3 | 香烟提醒 | 检测到香烟 |
| 0005.mp3 | 拨号失败提示 | 4G 拨号失败 |
| 0006.mp3 | 电话接通提示 | 4G 通话已接通 |
| 0007.mp3 | 静音提醒 | 噪音 >45dB |

> 格式化 TF 卡时必须选 FAT32，分区表类型选 MBR。

---

# 十四、软件架构与设计模式

## 11.1 多线程架构

```
┌─────────────────────────────────────────────────┐
│                    main() 主线程                  │
│  火焰检测 → 人腿检测 → 状态机 → 决策 → 执行       │
│  loop: usleep(80000) ~12.5Hz                     │
├─────────────────────────────────────────────────┤
│              cam_thread 摄像头线程                │
│  OpenCV VideoCapture → shared(彩色)               │
│                      → shared_gray(灰度)          │
│  mutex 保护, 独立于主循环                         │
├─────────────────────────────────────────────────┤
│           speed_thread PWM 调速线程               │
│  100Hz (10ms周期) 忙等循环                        │
│  读 atomic<int> motor_state 决定输出              │
│  motor_state=0 时空闲 usleep(10000) 降低 CPU      │
├─────────────────────────────────────────────────┤
│          mjpeg_thread 远程监控线程 (可选)          │
│  HTTP MJPEG 服务器, 端口 8081                     │
│  JPEG 质量 35%, ~15fps                           │
└─────────────────────────────────────────────────┘

线程间通信:
  shared / shared_gray  ←→  std::mutex lock
  motor_running          ←→  std::atomic<bool>
  motor_state            ←→  std::atomic<int>
```

## 11.2 设计模式总结

### 持久文件描述符模式
```cpp
// 初始化: open() 一次
pfd[i]=open("/sys/class/gpio/gpio60/value",O_RDWR);
// 使用: lseek+write 替代 fopen+fprintf+fclose
lseek(pfd[i],0,SEEK_SET); write(pfd[i],&c,1);
```
**优势**: 从 ~200µs 降到 ~80µs，2.5×加速。对 100Hz PWM 来说这是刚需。

### 状态变化触发模式 (MP3 播报)
```cpp
int want_play = (detected) ? 2 : 1;
if (want_play != last_play && now - last_cmd > 3) {
    mp3_play(want_play);
    last_play = want_play;  // 只在状态变化时发一次命令
}
```
**优势**: 避免每帧都发串口命令阻塞 UART 和 DFPlayer。

### 快速通道 + 慢速通道模式 (避障)
```
正常行驶: 快测前方 ~60ms/次 (舵机不动)
遇障:     停车 → 舵机扫两方向 ~1s (完整扫描)
```
**优势**: 90% 时间走快速通道，只有 10% 时间触发完整扫描。

### 优先级状态机
```
火焰检测 (最高优先级) → 停车+报警
  ↓ 无火
人腿检测 → 停车+播报+等待3秒
  ↓ 无腿
正常巡检 → 超声波避障 + 视觉里程计 + 背景音
```

## 11.3 GPIO sysfs 公用函数库

所有程序使用的通用 GPIO 操作模式:

```cpp
// 导出+设方向 (带状态清理)
static void gpio_out(int g){
    // 已导出? → unexport
    // export → 等100ms
    // 设 direction=out
}

// fopen/fclose 方式写入 (简单场景)
static void gpio(int g, int v){
    FILE*f=fopen("/sys/class/gpio/gpio%d/value"...);
    fprintf(f,"%d",v); fclose(f);
}

// 持久 fd 方式写入 (高频 PWM 场景)
static void pfd_w(int fd, int v){
    char c=v?'1':'0';
    lseek(fd,0,SEEK_SET); write(fd,&c,1);
}
```

---

> **文档结束**
>
> 本文档覆盖了 LS2K0300 全自动巡检小车项目的全部 25 个源程序文件，包含：
> - 详细的逐行代码逻辑注释
> - 硬件接线全表与 GPIO 映射
> - 核心算法原理深度解析（火焰检测 HSV、LK 光流、软件 PWM）
> - 通信协议说明（DFPlayer 10 字节帧、AT 指令短信）
> - 设计模式与架构总结
> - 编译命令速查
>
> **项目 BOM 成本**: < 250 元（含语音唤醒 SU-03T + 4G 通信 Air780E）
> **续航时间**: > 4 小时（5×AA 干电池 7.5V）
> **整车重量**: < 1kg
