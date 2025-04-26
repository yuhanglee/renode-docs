# Renode、Fomu 和 EtherBone 网桥示例

本教程介绍如何运行混合仿真，其中平台的一部分在 Renode 中运行，另一部分在 FPGA 硬件上运行。

## 系统架构

系统架构由三个主要部分组成：

* [FOMU](https://github.com/im-tomu/fomu-hardware) - 基于 Lattice iCE40UP5K 的电路板，带 RGB LED，通过 USB 连接到主机;
* EtherBone 桥接器，在 TCP 和 USB 之间转换 wishbone 数据包;
* Renode 模拟 [LiteX](https://github.com/enjoy-digital/litex) 平台，运行控制 RGB LED 的 Zephyr OS。

## 先决条件

### 模拟

对于仿真部分，您的系统中需要有最新版本的 Renode。

您可以为作系统安装[预构建的软件包](https://github.com/renode/renode/releases) ，也可以从从[源构建 Renode](https://renode.readthedocs.io/en/latest/advanced/building_from_sources.html) 中指定的源构建副本。

### 硬件

对于硬件部分，您需要有 [FOMU](https://github.com/im-tomu/fomu-hardware) 板。

[FOMU 需要用 Foboot](https://github.com/im-tomu/fomu-hardware) bitstream 进行闪存，该 [Foboot](https://github.com/im-tomu/foboot) bitstream 包含带有 USB-Wishbone 桥接的 [ValentyUSB](https://github.com/mithro/valentyusb) IP core。制造的 FOMU 预装了此 bitstream，可以立即使用。

如果您已经组装了自己的板子副本或更改了原始的 bitsteam，请记得再次加载 Foboot。为方便起见，bitstream 的预构建版本[由 Antmicro 托管](https://antmicro.com/projects/renode/foboot-bitstream.bin-s_104250-fc5f419372eb9a3a0baa5556483163bcfccb7d33) 。

### 网桥

为了将 Renode 连接到 [FOMU](https://github.com/im-tomu/fomu-hardware)，您需要下载 [LiteX](https://github.com/enjoy-digital/litex) 附带的 EtherBone 到 USB 桥接器。

克隆存储库并初始化环境：

```bash
git clone https://github.com/enjoy-digital/litex
cd litex
./litex_setup.py init  # this will clone the dependencies
export PYTHONPATH=`pwd`:`pwd`/litex:`pwd`/migen
```

`litex_server.py` 还需要在系统中安装 `pyusb` 包：

```bash
pip3 install pyusb
```

## 验证设备

将 [FOMU](https://github.com/im-tomu/fomu-hardware) 插入 USB 端口，并通过检查 `dmesg` 日志来验证它是否已被识别：

```text
[65038.250957] usb 2-1: new full-speed USB device number 16 using xhci_hcd
[65038.409283] usb 2-1: New USB device found, idVendor=1209, idProduct=5bf0, bcdDevice= 1.01
[65038.409286] usb 2-1: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[65038.409287] usb 2-1: Product: Fomu DFU Bootloader v1.7.2-3-g9013054
[65038.409288] usb 2-1: Manufacturer: Foosn
```

注意：产品的版本可能有所不同，但应该可以在 v.1.7.2 及更高版本中正常工作。

如果未检测到设备，请参阅以下部分。

## 加载 bitstream （可选）

如果您的设备在 1.7.2 或更高版本中被检测到 DFU 引导加载程序，您可以跳过此步骤。

为了将 bitstream 上传到未被识别为 DFU Bootloader 的器件，您需要一个外部编程板。

:::{note}

为方便起见，您可以使用 [Fomu Programmer](https://github.com/antmicro/fomu-programmer) - Antmicro 的 [FOMU](https://github.com/im-tomu/fomu-hardware) 开放硬件编程板。

:::

下载并制作 [iceprog](https://github.com/cliffordwolf/icestorm/tree/master/iceprog) - 用于 Lattice iCE40 的开源编程软件：

:::{note}

构建 `iceprog` 需要在系统中提供 `ftdi` 库头文件。

:::

```bash
git clone https://github.com/cliffordwolf/icestorm
cd icestorm/iceprog
make
```

下载预构建的 bitstream：

```bash
wget https://antmicro.com/projects/renode/foboot-bitstream.bin-s_104250-fc5f419372eb9a3a0baa5556483163bcfccb7d33 -O foboot-bitstream.bin
```

:::{note}

您也可以按照 [Foboot](https://github.com/im-tomu/foboot) 页面上的说明自己构建 bitstream。

:::

将板子连接到编程器并将 bitstream 加载到 FPGA：

```bash
sudo iceprog foboot-bitstream.bin
```

## 运行演示

从 [LiteX](https://github.com/enjoy-digital/litex) 存储库启动 EtherBone 网桥

```bash
cd litex
sudo python3 litex/tools/litex_server.py --usb --usb-vid 0x1209 --usb-pid 0x5bf0
```

使用 Renode 附带的脚本在模拟中运行 Zephyr OS 映像：

```text
(monitor) start @scripts/complex/fomu/renode_etherbone_fomu.resc
```

现在，您可以使用特殊命令从 Zephyr 的 shell 控制硬件 LED：

```bash
uart:~$ led_toggle
uart:~$ led_breathe
```

`led_toggle`
    切换绿色 LED

`led_breathe`
    使蓝色 LED 闪烁并产生淡入/淡出效果

