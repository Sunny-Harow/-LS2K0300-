# 龙芯 2K0300 交叉编译工具链 (rc1.3-1) 完整目录文档

> **路径**: `H:\loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.3-1`
> **目标**: `x86_64` (Windows/Linux PC) → `loongarch64-linux-gnu` (久久派 LS2K0300)

---

## 顶层目录

```
loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.3-1/
├── bin/                         ← 交叉编译器主目录 (30 个可执行文件)
├── include/                     ← GDB JIT 调试头文件
├── lib/                         ← GCC 内部库
├── libexec/                     ← GCC 编译后端 (cc1, cc1plus 等)
├── loongarch64-linux-gnu/       ← 目标平台文件
├── share/                       ← 文档/GDB配置/man手册
└── versions/                    ← 各组件版本信息
```

---

## 1. `bin/` — 交叉编译工具 (30 个)

| 工具 | 功能说明 |
|---|---|
| `loongarch64-linux-gnu-gcc` | **C 编译器** (主程序, → GCC 8.3.0) |
| `loongarch64-linux-gnu-g++` | **C++ 编译器** |
| `loongarch64-linux-gnu-gfortran` | Fortran 编译器 |
| `loongarch64-linux-gnu-gdb` | **GDB 调试器** |
| `loongarch64-linux-gnu-as` | GNU 汇编器 |
| `loongarch64-linux-gnu-ld` | GNU 链接器 |
| `loongarch64-linux-gnu-ld.bfd` | BFD 链接器 (ld 的实际后端) |
| `loongarch64-linux-gnu-ar` | 静态库归档工具 |
| `loongarch64-linux-gnu-ranlib` | 静态库索引生成 |
| `loongarch64-linux-gnu-nm` | 符号表查看 |
| `loongarch64-linux-gnu-objdump` | 反汇编工具 |
| `loongarch64-linux-gnu-objcopy` | 目标文件格式转换 |
| `loongarch64-linux-gnu-readelf` | ELF 文件分析 |
| `loongarch64-linux-gnu-strip` | 符号/调试信息剥离 |
| `loongarch64-linux-gnu-size` | 段大小查看 |
| `loongarch64-linux-gnu-strings` | 提取可打印字符串 |
| `loongarch64-linux-gnu-addr2line` | 地址 → 源码行号转换 |
| `loongarch64-linux-gnu-cpp` | C 预处理器 |
| `loongarch64-linux-gnu-c++filt` | C++ 符号名反修饰 |
| `loongarch64-linux-gnu-elfedit` | ELF 文件编辑器 |
| `loongarch64-linux-gnu-gcov` | 代码覆盖率工具 |
| `loongarch64-linux-gnu-gcov-dump` | gcov 数据导出 |
| `loongarch64-linux-gnu-gcov-tool` | gcov 离线合并 |
| `loongarch64-linux-gnu-gprof` | 性能分析工具 |
| `loongarch64-linux-gnu-gcc-ar` | GCC LTO 版 ar |
| `loongarch64-linux-gnu-gcc-nm` | GCC LTO 版 nm |
| `loongarch64-linux-gnu-gcc-ranlib` | GCC LTO 版 ranlib |
| `loongarch64-linux-gnu-gdb-add-index` | GDB 索引加速脚本 |
| `loongarch64-linux-gnu-gcc-8.3.0` | GCC 8.3.0 硬链接 |

---

## 2. `include/` — 头文件

```
include/
└── gdb/
    └── jit-reader.h              ← GDB JIT (Just-In-Time) 调试接口
```

---

## 3. `lib/` — GCC 内部库

```
lib/
└── gcc/
    └── loongarch64-linux-gnu/
        └── 8.3.0/                 ← GCC 8.3.0 后端目录
            ├── crtbegin.o         ← C 运行时启动文件
            ├── crtend.o
            ├── libgcc.a           ← GCC 运行时库
            ├── libgcc_eh.a        ← 异常处理支持库
            └── plugin/            ← GCC 插件支持
```

---

## 4. `libexec/` — GCC 编译后端

```
libexec/
└── gcc/
    └── loongarch64-linux-gnu/
        └── 8.3.0/
            ├── cc1                ← C 编译器核心 (预处理→汇编)
            ├── cc1plus            ← C++ 编译器核心
            ├── collect2           ← 链接器包装
            ├── lto1               ← Link-Time Optimization
            └── install-tools/     ← 安装辅助工具
```

---

## 5. `loongarch64-linux-gnu/` — 目标平台文件

### 5.1 `bin/` — 目标平台工具 (10 个)

| 工具 | 功能 |
|---|---|
| `ar` | 静态库管理 |
| `as` | 汇编器 |
| `ld` / `ld.bfd` | 链接器 |
| `nm` | 符号查看 |
| `objcopy` | 目标文件转换 |
| `objdump` | 反汇编 |
| `ranlib` | 库索引 |
| `readelf` | ELF 分析 |
| `strip` | 符号剥离 |

### 5.2 `include/` — 目标平台头文件

```
loongarch64-linux-gnu/include/
├── c++/8.3.0/                 ← C++ 标准库头文件
├── objc/                      ← Objective-C 支持
└── sanitizer/                 ← Address/Thread/UB Sanitizer
```

### 5.3 `lib/` — 链接脚本 (21 个)

```
loongarch64-linux-gnu/lib/
└── ldscripts/
    ├── elf64loongarch.x           ← 默认链接脚本
    ├── elf64loongarch.xbn         ← 标志性加载
    ├── elf64loongarch.xc          ← 组合
    ├── elf64loongarch.xce         ← 组合+嵌入式
    ├── elf64loongarch.xd          ← 数据段
    ├── elf64loongarch.xdc         ← 数据+组合
    ├── elf64loongarch.xdce        ← 数据+组合+嵌入式
    ├── elf64loongarch.xde         ← 数据+嵌入式
    ├── elf64loongarch.xdw         ← 数据只读
    ├── elf64loongarch.xdwe        ← 数据只读+嵌入式
    ├── elf64loongarch.xe          ← 嵌入式
    ├── elf64loongarch.xn          ← 非默认
    ├── elf64loongarch.xr          ← 只读
    ├── elf64loongarch.xs          ← 共享
    ├── elf64loongarch.xsc         ← 共享+组合
    ├── elf64loongarch.xsce        ← 共享+组合+嵌入式
    ├── elf64loongarch.xse         ← 共享+嵌入式
    ├── elf64loongarch.xsw         ← 共享+数据只读
    ├── elf64loongarch.xswe        ← 共享+数据只读+嵌入式
    ├── elf64loongarch.xu          ← 未定义
    └── elf64loongarch.xw          ← 数据只读
```

### 5.4 `sysroot/` — 目标系统根文件镜像

```
loongarch64-linux-gnu/sysroot/
├── etc/
│   └── rpc/                       ← RPC 服务定义
│
├── usr/
│   ├── bin/                       ← 目标系统用户程序
│   ├── include/                   ← 目标系统 C 库头文件
│   │   ├── stdio.h, stdlib.h ...  ← 标准 C 库
│   │   └── linux/...              ← Linux 系统调用
│   ├── lib/                       ← 目标系统 32 位库
│   ├── lib64/                     ← 目标系统 64 位库
│   │   ├── libc.so                ← C 运行时库
│   │   ├── libm.so                ← 数学库
│   │   ├── libpthread.so          ← POSIX 线程库
│   │   └── libdl.so               ← 动态链接库
│   ├── libexec/                   ← 系统级可执行文件
│   ├── sbin/                      ← 系统管理程序
│   └── share/                     ← 架构无关数据
│
└── var/
    └── db/                        ← 系统数据库
```

---

## 6. `share/` — 文档和配置

### 6.1 许可证

```
share/
├── licenses.txt                   ← 所有组件的开源许可证合集
```

### 6.2 GDB 配置

```
share/gdb/
├── syscalls/                      ← 系统调用 XML 定义 (各架构)
│   ├── gdb-syscalls.dtd           ← DTD 文档类型定义
│   ├── aarch64-linux.xml          ← ARM 64位
│   ├── amd64-linux.xml            ← x86 64位
│   ├── arm-linux.xml              ← ARM 32位
│   ├── i386-linux.xml             ← x86 32位
│   ├── mips-n32-linux.xml         ← MIPS N32
│   ├── mips-n64-linux.xml         ← MIPS N64
│   ├── mips-o32-linux.xml         ← MIPS O32
│   ├── ppc-linux.xml              ← PowerPC 32位
│   ├── ppc64-linux.xml            ← PowerPC 64位
│   ├── s390-linux.xml             ← IBM S/390 32位
│   ├── s390x-linux.xml            ← IBM S/390 64位
│   ├── sparc-linux.xml            ← SPARC 32位
│   ├── sparc64-linux.xml          ← SPARC 64位
│   └── freebsd.xml                ← FreeBSD
│
└── system-gdbinit/                ← 系统级 GDB 初始化脚本
    ├── elinos.py                  ← ELinOS 嵌入式系统
    └── wrs-linux.py               ← Wind River Linux
```

### 6.3 Info 手册 (GDB 完整文档, 7个分卷)

```
share/info/
├── dir                            ← Info 目录索引
├── gdb.info-1                     ← GDB 手册 第一部分
├── gdb.info-2                     ← GDB 手册 第二部分
├── gdb.info-3                     ← GDB 手册 第三部分
├── gdb.info-4                     ← GDB 手册 第四部分
├── gdb.info-5                     ← GDB 手册 第五部分
├── gdb.info-6                     ← GDB 手册 第六部分
└── gdb.info-7                     ← GDB 手册 第七部分
```

### 6.4 Man 手册页

| 路径 | 内容 |
|---|---|
| `man/man1/` | **29 个用户命令手册** (gcc, g++, gdb, ld, as, objdump...) |
| `man/man5/` | `gdbinit.5` — GDB 初始化文件格式 |
| `man/man7/` | `gpl.7` / `gfdl.7` / `fsf-funding.7` — 许可证文档 |

#### man1/ 完整列表 (29 个)

| 手册文件 | 对应工具 |
|---|---|
| `loongarch64-linux-gnu-gcc.1` | GCC C 编译器 |
| `loongarch64-linux-gnu-g++.1` | G++ C++ 编译器 |
| `loongarch64-linux-gnu-gdb.1` | GDB 调试器 |
| `loongarch64-linux-gnu-gdbserver.1` | GDB 远程调试服务 |
| `loongarch64-linux-gnu-gfortran.1` | Fortran 编译器 |
| `loongarch64-linux-gnu-as.1` | 汇编器 |
| `loongarch64-linux-gnu-ld.1` | 链接器 |
| `loongarch64-linux-gnu-ar.1` | 静态库工具 |
| `loongarch64-linux-gnu-nm.1` | 符号表查看 |
| `loongarch64-linux-gnu-objcopy.1` | 目标文件转换 |
| `loongarch64-linux-gnu-objdump.1` | 反汇编 |
| `loongarch64-linux-gnu-ranlib.1` | 库索引工具 |
| `loongarch64-linux-gnu-readelf.1` | ELF 分析 |
| `loongarch64-linux-gnu-size.1` | 段大小查看 |
| `loongarch64-linux-gnu-strings.1` | 字符串提取 |
| `loongarch64-linux-gnu-strip.1` | 符号剥离 |
| `loongarch64-linux-gnu-cpp.1` | C 预处理器 |
| `loongarch64-linux-gnu-c++filt.1` | C++ 符号反修饰 |
| `loongarch64-linux-gnu-elfedit.1` | ELF 编辑 |
| `loongarch64-linux-gnu-gcov.1` | 代码覆盖率 |
| `loongarch64-linux-gnu-gcov-dump.1` | gcov 导出 |
| `loongarch64-linux-gnu-gcov-tool.1` | gcov 合并 |
| `loongarch64-linux-gnu-gprof.1` | 性能分析 |
| `loongarch64-linux-gnu-addr2line.1` | 地址转换 |
| `loongarch64-linux-gnu-gdb-add-index.1` | GDB 索引 |
| `loongarch64-linux-gnu-windmc.1` | Windows 消息编译 (交叉用) |
| `loongarch64-linux-gnu-windres.1` | Windows 资源编译 (交叉用) |
| `loongarch64-linux-gnu-dlltool.1` | Windows DLL 工具 (交叉用) |

---

## 7. `versions/` — 版本信息

记录工具链各组件版本:
- GCC: 8.3.0
- Binutils: (ld/as/objcopy...)
- GDB: GDB 调试器
- GLIBC: GNU C Library
- Linux Kernel headers

---

## 快速使用

```bash
# 1. 添加 PATH
export PATH="H:/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.3-1/bin:$PATH"

# 2. 编译 C 程序
loongarch64-linux-gnu-gcc -o program program.c

# 3. 查看编译产物
loongarch64-linux-gnu-file program
# 输出: ELF 64-bit LSB executable, LoongArch ...

# 4. SCP 传到板子运行
scp program root@192.168.2.77:/home/root/
```
