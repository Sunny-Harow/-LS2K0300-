# 龙芯 2K0300 (久久派) 开发工具集

> 适用于 LS2K0300 / 久久派 嵌入式 Linux 开发，包括交叉编译工具链、OpenCV 库、环境搭建软件及调试工具。

---

## 📁 目录结构

```
久久pai/
├── 龙芯2K30x 交叉编译工具/
│   └── loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6.tar.xz
│
├── 龙芯2K30x opencv库/
│   └── opencv_4_10_build.tar.xz
│
├── 龙芯环境搭建软件/
│   ├── VMware-workstation-full-17.0.2-21581411.exe
│   ├── MobaXterm_Portable_v25.1.zip
│   └── Ubuntu 24/
│       └── ubuntu-24.04.2-desktop-amd64.iso
│
├── MobaXterm/
│   └── MobaXterm_Personal_24.2.exe
│
├── Xshell5/
│   ├── Xshell_5.0.1055.exe
│   └── Xshell5密钥.txt
│
├── 龙邱UDP_串口调试助手_LoongLiteTool.exe
├── FileZilla_3.67.1_win64-setup.exe
├── CH341SER.EXE
├── DiskGenius-Pro-v5.5.0.1488-x64-Chs-v1.2.exe
└── DG-Pro-v5.5.0.1488-x64-Chs-v1.2.zip
```

---

## 🔧 1. 交叉编译工具链

| 属性 | 说明 |
|---|---|
| **文件** | `loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6.tar.xz` |
| **大小** | 34.36 MB |
| **编译器** | GCC 8.3 |
| **宿主平台** | x86_64 (你的 PC，Linux 环境) |
| **目标平台** | loongarch64-linux-gnu (龙芯 2K0300) |

### 安装方法

```bash
# 解压到 /opt 目录（需在 Linux/WSL 中执行）
sudo mkdir -p /opt/ls_2k0300_env
sudo tar -xvf loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6.tar.xz -C /opt/ls_2k0300_env/

# 添加环境变量
echo 'export PATH=/opt/ls_2k0300_env/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 验证
loongarch64-linux-gnu-gcc --version
```

### 解压后目录结构

```
loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.6/
├── bin/
│   ├── loongarch64-linux-gnu-gcc        ← C 编译器
│   ├── loongarch64-linux-gnu-g++        ← C++ 编译器
│   ├── loongarch64-linux-gnu-ld         ← 链接器
│   ├── loongarch64-linux-gnu-ar         ← 静态库归档
│   ├── loongarch64-linux-gnu-strip      ← 符号剥离
│   └── ...
├── loongarch64-linux-gnu/               ← 目标平台的库和头文件
├── include/
├── lib/
├── libexec/
└── share/
```

---

## 📷 2. OpenCV 库（交叉编译版）

| 属性 | 说明 |
|---|---|
| **文件** | `opencv_4_10_build.tar.xz` |
| **版本** | OpenCV 4.10 |
| **用途** | 编译好的 loongarch64 版本，交叉编译时链接使用 |

### 安装方法

```bash
sudo mkdir -p /opt/ls_2k0300_env
sudo tar -xvf opencv_4_10_build.tar.xz -C /opt/ls_2k0300_env/
# 完成后路径: /opt/ls_2k0300_env/opencv_4_10_build/
```

---

## 💻 3. 开发环境搭建软件

### 3.1 虚拟机 & Ubuntu

| 文件 | 说明 |
|---|---|
| `VMware-workstation-full-17.0.2-21581411.exe` | VMware Workstation 17 虚拟机软件 |
| `ubuntu-24.04.2-desktop-amd64.iso` | Ubuntu 24.04 桌面版 ISO 镜像 |

> ⚠️ Ubuntu ISO 以 `.7z` 分卷压缩存储，需用压缩软件打开 `.001` 文件提取。

### 3.2 终端工具

| 文件 | 说明 |
|---|---|
| `MobaXterm_Portable_v25.1.zip` | MobaXterm 便携版 v25.1（SSH/串口/SCP 多功能终端） |
| `MobaXterm_Personal_24.2.exe` | MobaXterm 安装版 v24.2 |
| `Xshell_5.0.1055.exe` | Xshell 5 终端软件 |

---

## 🛠️ 4. 调试与辅助工具

| 文件 | 大小 | 说明 |
|---|---|---|
| `龙邱UDP_串口调试助手_LoongLiteTool.exe` | 116.9 MB | UDP + 串口调试工具（龙邱出品） |
| `FileZilla_3.67.1_win64-setup.exe` | 11.83 MB | FTP/SFTP 文件传输工具 |
| `CH341SER.EXE` | 0.27 MB | CH340/CH341 USB 转串口驱动 |
| `DiskGenius-Pro-v5.5.0.1488-x64-Chs-v1.2.exe` | 31.33 MB | 磁盘分区/镜像烧录工具 |

---

## 🚀 快速开始：两种开发方式

### 方式一：板载直接编译（推荐，最简单）

```
你的 PC (Windows)                    久久派板子 (LoongArch Linux)
┌─────────────────┐                  ┌──────────────────────────┐
│ MobaXterm / SSH │  ─── 网络 ───→  │ gcc 直接编译 C/C++ 程序   │
│ 编辑代码         │                  │ gcc -o main main.cpp      │
│ SCP 上传代码     │                  │ sudo ./main               │
└─────────────────┘                  └──────────────────────────┘
```

- **优点**：不需要 Ubuntu 虚拟机，不需要交叉工具链
- **适合**：小程序、调试、测试

### 方式二：交叉编译（大型项目）

```
┌────────────────────────────────────────┐     SCP      ┌──────────────────┐
│ VMware Ubuntu 24.04                    │  ─────────→  │ 久久派板子        │
│ + 龙芯交叉工具链 (GCC 8.3)             │              │                  │
│ + OpenCV 4.10 (loongarch 版)          │              │ ./可执行文件      │
│ + CMake + make                        │              │                  │
└────────────────────────────────────────┘              └──────────────────┘
```

- **优点**：编译速度快，支持 CMake 管理的大项目
- **适合**：大型工程、需要链接 OpenCV/TFLM 等库的项目

---

## 📌 注意事项

1. **交叉工具链必须在 Linux x86_64 环境**下运行（Ubuntu 虚拟机 或 WSL2），不能在 Windows 直接使用
2. 工具链解压后约 200~300 MB，确保磁盘空间充足
3. 板子自带 gcc 编译器，简单程序直接在板子上编译即可，不需要交叉编译
4. CH341SER 驱动安装后，USB 转 TTL 串口线才能正常使用
5. 龙邱调试工具仅支持 Windows
