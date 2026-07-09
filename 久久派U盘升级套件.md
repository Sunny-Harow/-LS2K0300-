# 久久派（龙芯 LS2K0300）U盘升级套件

> **用途**: 通过 U 盘烧录/恢复久久派板卡系统  
> **目标设备**: EMMC (`/dev/mmcblk0`)

---

## 📁 目录结构

```
久久派升级/
├── PMON/
│   └── ls2k300-dtb.bin          ← PMON 固件 + 设备树 blob
│
├── 操作手册/
│   └── 久久派（龙芯板卡）U盘升级操作手册.pdf
│
├── 硬盘工具/
│   └── DiskGenius-Pro-v5.5.0.1488-x64-Chs-v1.2.exe
│
└── 系统U盘/                      ← ★ 拷到 U 盘根目录
    ├── boot.cfg                 ← PMON 启动配置
    ├── vmlinuz                  ← Linux 内核镜像
    ├── update.cpio.gz           ← initramfs (包含升级脚本)
    ├── update.sh                ← 系统烧录 Shell 脚本
    └── update/
        └── LQ_LS2K0300_V2.tar.gz ← 系统根文件系统镜像
```

---

## 🚀 升级流程

```
插入U盘 → 上电 → PMON加载vmlinuz+initramfs → 执行update.sh → 烧录到EMMC → 重启
```

### 1. boot.cfg — PMON 启动配置

```ini
default 0
timeout 2
showmenu 0

title 2k300 CC Update
    kernel (usb0,0)/vmlinuz
    initrd (usb0,0)/update.cpio.gz
    args console=ttyS0,115200
```

- `default 0` — 默认启动第一个菜单项
- `timeout 2` — 2 秒超时自动启动
- `kernel (usb0,0)/vmlinuz` — 从 U 盘第一个分区加载内核
- `initrd (usb0,0)/update.cpio.gz` — 从 U 盘加载 initramfs
- `args console=ttyS0,115200` — 串口控制台输出

### 2. update.sh — 系统烧录脚本

```
┌──────────────────────────────────────┐
│ 1. 检查 EMMC 设备 (/dev/mmcblk0)      │
│ 2. 卸载已有分区                       │
│ 3. 清除分区表 (dd + fdisk)            │
│ 4. 创建新分区 (/dev/mmcblk0p1)        │
│ 5. 格式化 ext4                        │
│ 6. 挂载分区到 /update                  │
│ 7. 解压 LQ_LS2K0300_V2.tar.gz 到 EMMC │
│ 8. 验证关键文件 (sbin/init, etc...)    │
│ 9. 同步 + 卸载 + 重新挂载验证          │
│ 10. 完成，提示重启                     │
└──────────────────────────────────────┘
```

### 3. update.sh 完整代码

<details>
<summary>点击展开完整代码</summary>

```sh
#!/bin/sh

echo "=========================================="
echo "    LoongOS 2K300 系统烧录脚本"
echo "=========================================="

# 检查EMMC设备是否存在
if [ ! -e "/dev/mmcblk0" ]; then
    echo "[ERROR] EMMC 驱动加载失败，未找到 /dev/mmcblk0"
    exit 1
fi

echo "[INFO] 发现EMMC设备: /dev/mmcblk0"

# 卸载已有分区（如果存在）
if [ -e "/dev/mmcblk0p1" ]; then
    echo "[INFO] 卸载现有分区..."
    umount /dev/mmcblk0p1 2>/dev/null
fi

# 清除EMMC分区表并创建新分区
echo "[INFO] 清除EMMC分区表..."
dd if=/dev/zero of=/dev/mmcblk0 bs=512 count=1 2>/dev/null
sync

echo "[INFO] 创建新分区..."
echo -e "n\np\n1\n\n\nw\n" | fdisk /dev/mmcblk0

# 等待分区创建完成
sleep 1

if [ ! -e "/dev/mmcblk0p1" ]; then
    echo "[ERROR] EMMC 分区创建失败"
    exit 2
fi

echo "[INFO] 分区创建成功: /dev/mmcblk0p1"

# 格式化分区为ext4
echo "[INFO] 格式化EMMC分区 (ext4)..."
echo -e "y" | mkfs.ext4 /dev/mmcblk0p1

if [ $? -ne 0 ]; then
    echo "[ERROR] 格式化失败"
    exit 3
fi

echo "[INFO] 格式化完成"

# 挂载EMMC分区
updir=/update
mkdir -p $updir

echo "[INFO] 挂载EMMC分区到 $updir ..."
mount /dev/mmcblk0p1 $updir

if [ $? -ne 0 ]; then
    echo "[ERROR] 挂载失败"
    exit 4
fi

# 检查系统镜像文件（U盘挂载在 /mnt/update/）
IMG_PATH="/mnt/update/LQ_LS2K0300_V2.tar.gz"
if [ ! -e "$IMG_PATH" ]; then
    echo "[ERROR] 未找到系统镜像: $IMG_PATH"
    umount $updir
    exit 5
fi

echo "[INFO] 找到系统镜像: $IMG_PATH"

# 检查镜像完整性
echo "[INFO] 检查镜像完整性..."
tar -tzf "$IMG_PATH" > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "[ERROR] 镜像文件损坏或格式错误"
    umount $updir
    exit 6
fi

echo "[INFO] 镜像检查通过"

# 解压系统镜像到EMMC
echo "[INFO] 正在写入系统，请稍候..."
tar -xzpf "$IMG_PATH" -C $updir

if [ $? -ne 0 ]; then
    echo "[ERROR] 系统写入失败"
    umount $updir
    exit 6
fi

echo "[INFO] 系统写入完成"

# 同步数据到磁盘
echo "[INFO] 同步数据到磁盘..."
sync
sleep 3
sync

# 检查关键文件
echo "[INFO] 检查系统文件结构..."
for dir in proc sys dev boot etc bin sbin lib usr var tmp home; do
    if [ -d "$updir/$dir" ]; then
        echo "  [OK] /$dir"
    else
        echo "  [ERROR] /$dir 不存在"
    fi
done

if [ -e "$updir/sbin/init" ]; then
    echo "  [OK] /sbin/init"
else
    echo "  [ERROR] /sbin/init 不存在"
fi

# 卸载分区
echo "[INFO] 卸载分区..."
sync
sleep 2
sync
umount $updir

# 重新挂载验证
echo "[INFO] 重新挂载验证数据..."
mount /dev/mmcblk0p1 $updir

if [ -e "$updir/sbin/init" ] && [ -e "$updir/bin" ] && [ -e "$updir/etc" ]; then
    echo "[OK] 数据验证通过"
else
    echo "[ERROR] 数据验证失败"
    umount $updir
    exit 9
fi

umount $updir

echo ""
echo "=========================================="
echo "    系统烧录完成！"
echo "=========================================="
echo "[INFO] 请重启设备从EMMC启动"
```
</details>

---

## 📋 错误码说明

| 退出码 | 含义 |
|---|---|
| 0 | 烧录成功 |
| 1 | EMMC 设备未找到 |
| 2 | 分区创建失败 |
| 3 | 格式化 ext4 失败 |
| 4 | 挂载 EMMC 分区失败 |
| 5 | U 盘系统镜像未找到 |
| 6 | 镜像损坏或写入失败 |
| 7 | 无法卸载分区 |
| 8 | 重新挂载失败 |
| 9 | 数据验证失败 |

---

## 🔧 制作升级 U 盘

1. **格式化 U 盘**: FAT32 格式 (可用 DiskGenius)
2. **复制文件**: 将 `系统U盘/` 下所有文件复制到 U 盘根目录
3. **插入久久派**: 将 U 盘插入板子 USB 口
4. **上电启动**: PMON 自动从 U 盘启动并开始烧录
5. **等待完成**: 看到 "系统烧录完成" 后重启
6. **移除 U 盘**: 从 EMMC 正常启动

---

## ⚠️ 注意事项

- 升级会**清空 EMMC 全部数据**，请提前备份
- 升级过程中**切勿断电**
- U 盘必须是 FAT32 格式，文件放在根目录
