# Magmortar(HiFive Unmatched) Debug Notes

PS: 本文中 larvesta, reshiram, magmortar 均为 Unmatched.

## Issue

目前 Magmortar 的问题：

1. 运行一段时间之后会 panic。内核无报错，接串口无崩溃日志。
2. 无法使用 SPI 模式运行 bootloader

## Attempt

### 崩溃问题的尝试

#### #1 固件魔改出了问题

最初的对问题的猜测是：魔改过后的固件和板子不兼容导致崩溃。但新的 larvesta 和 reshiram 使用的固件和 magmortar 是相同的，
这两台机子在上线后持续稳定工作，因此大概和固件的修改没什么关系。

#### #2 温控问题

magmortar 之前的工作环境在机柜内，相对室内温度较高。我们的怀疑是温度限制值太低。因此尝试修改了 magmortar 的温度限制。

给 u-boot 的 spl.c 文件打上如下的 patch，首先将温度限制调低至 0x35 (53 摄氏度):

```diff
+#define TMP451_REMOTE_THERM_LIMIT_INIT_VALUE	0x35
```

使用热风机加热到大约五十度左右，magmortar 这块板子确实又炸了。

但后续将温控问题排除了，重新修正温控限制值到 `0x96`（150 摄氏度）之后，在空调环境下，CPU 负载运行一段时间依旧会出现崩溃现象。

#### #3 Kernel 问题

目前新刷入 SD Card 的固件和系统是无法正常启动的，在 bootloader 运行 `run bootcmd_mmc0` 命令从 SD Card 启动系统会在 `Starting Kernel`
提示信息出来之后卡死。

尝试：

- 更换 SD Card 刷入固件，问题依旧重现。
- 将该 SD Card 插入其他 Unmatched 板子，Kernel 依旧起不来，说明 SD Card 和 magmortar 本身出现的问题无关。

目前怀疑崩溃问题与 kernel 本身有关，仍需后续的调试和调查。

但抛去 Kernel 起不来的问题，在起 Kernel 卡死的时候，其他的板子只是 `Starting Kernel` 然后卡死，而 magmortar 会多汇报一句
`L2Cache: Data Error`.

验证方案（来自 @sequencer)

> 硬件问题我想过唯二的两个情况：
> 1. MMC的坏块在 trap_handler 中导致中断无法处理，原来的卡死问题如果和温度无关的话我怀疑就到这里了。
> 2. L2 缓存有坏块，然后在kernel 在init L2 Cache的时候Data Error是不是指的这个问题
> L2缓存坏块的验证方案是： 用sideband对L2的SRAM执行load/store 或者JTAG用SBA去access对应的地址

### 无法从 SPI 启动

需要使用 JTAG debug bootloader。仍需后续学习使用。

## 相关 Reference

- 我对原有 bootloader 的修改：<https://github.com/XYenChi/bootloader/pulls?q=is%3Apr+sort%3Aupdated-desc+author%3Aavimitin+>
- SPI Boot: <https://github.com/u-boot/u-boot/commit/6a863894ad53b2d0e6c6d47ad105850053757fec>

## 如何构建一个复现环境

1. 准备一个板子，或者克隆一份我修改之后的 qemu-system 镜像构建脚本并跟随 README 的指引构建镜像:

```bash
git clone https://github.com/Avimitin/archriscv-scriptlet-for-bootloader.git -b u-boot
```

2. 将 bootloader 脚本克隆到板子或者 qemu-system 环境里：

```bash
git clone https://github.com/XYenchi/bootloader
cd bootloader && git submodule update --init
```

3. 将 SD Card 插入，找到 SD Card 硬盘位置并使用 dd 烧入

```bash
dd if=image.raw of=/dev/sdb conv=sync bs=4096 status=progress
```

4. 连接电脑和串口，使用 picocom 连接板子的 TTY

```bash
sudo picocom -b 115200 /dev/ttyUSB1
```

5. 启动电源
