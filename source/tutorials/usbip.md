# USB/IP 支持

Renode 提供内置的 USB/IP 服务器，并允许您将虚拟 USB 设备导出到外部世界。

导出的设备可以连接到主机，从那一刻起，与它们的交互与与真实硬件的交互相同。

这允许混合设置，其中系统的一部分（即虚拟 USB 设备及其环境）在 Renode 中模拟，其余部分来自现实世界。

## USB/IP 协议

USB/IP 协议允许通过 IP 网络在服务器（输出方）和客户端（进口方）之间共享 USB 设备。

该协议[在 Linux 中得到了很好的支持](https://github.com/torvalds/linux/tree/master/tools/usb/usbip) ，并且也有[适用于 Windows 的相应项目](https://github.com/cezanne/usbip-win) ，但后者尚未经过测试。

Renode 目前仅作为服务器工作 - 它只能支持导出设备，但_无法_将物理 USB 设备连接到仿真。

## 创建 USB/IP 服务器

要在 Renode 中创建 USB/IP 服务器实例，请在监视器中键入以下内容：

```
(monitor) emulation CreateUSBIPServer
```

因此，将在仿真中创建 `host.usb` 设备并启动服务器。

现在您可以从主机连接到它。首先，您需要导入 `vhci_hcd` 内核模块：

```sh
$ sudo modprobe vhci_hcd
```

现在，您可以列出导出的设备：

```sh
$ sudo usbip list -r 127.0.0.1
usbip: info: no exportable devices found on 127.0.0.1
```

上面的输出通知 Renode 目前没有导出任何设备。

## 导出设备

让我们导出一个简单的 USB 设备 - 例如鼠标：

```none
(monitor) host.usb AttachUSBMouse
```

:::{note}
请注意，有一些辅助方法允许直接从 `host.usb` 轻松连接简单的 USB 设备，如鼠标、键盘或笔式驱动器。有关更高级的方案，请参阅  [foboot_usbip](#real-life-scenario-foboot).
:::

再次列出导出的设备：

```sh
$ sudo usbip list -r 127.0.0.1
Exportable USB devices
======================
- 127.0.0.1
        1-0: unknown vendor : unknown product (0000:0000)
        : /renode/virtual/1-0
        : (Defined at Interface level) (00/00/00)
        :  0 - Human Interface Device / Boot Interface Subclass / Mouse (03/01/02)
```

## 将导出的设备附加到主机

下一步是将导出的虚拟 USB 设备附加到主机：

```sh
$ sudo usbip attach -r 127.0.0.1 -b 1-0
```

请注意，`-b` 参数必须与 `usb list` 命令返回的设备 ID 匹配。

新的 USB 鼠标现在应该在主机中可见。通过阅读系统日志来确认它：

```sh
$ sudo dmesg
...
[1310770.549767] usb 7-1: New USB device found, idVendor=0000, idProduct=0000, bcdDevice= 0.00
[1310770.549771] usb 7-1: New USB device strings: Mfr=0, Product=0, SerialNumber=0
[1310770.560676] input: HID 0000:0000 as /devices/platform/vhci_hcd.0/usb7/7-1/7-1:1.0/0003:0000:0000.002D/input/input66
[1310770.560888] hid-generic 0003:0000:0000.002D: input,hidraw4: USB HID v0.00 Mouse [HID 0000:0000] on usb-vhci_hcd.0-1/input0
```

或者使用 `lsusb`：

```sh
$ lsusb -v -d 0000:0000

Bus 007 Device 091: ID 0000:0000
Device Descriptor:
bLength                18
bDescriptorType         1
bcdUSB               2.00
bDeviceClass            0
bDeviceSubClass         0
bDeviceProtocol         0
bMaxPacketSize0        64
idVendor           0x0000
idProduct          0x0000
bcdDevice            0.00
iManufacturer           0
iProduct                0
iSerial                 0
bNumConfigurations      1
Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength       0x0022
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0
    bmAttributes         0x00
    (Missing must-be-set bit!)
    (Bus Powered)
    MaxPower                0mA
    Interface Descriptor:
    bLength                 9
    bDescriptorType         4
    bInterfaceNumber        0
    bAlternateSetting       0
    bNumEndpoints           1
    bInterfaceClass         3 Human Interface Device
    bInterfaceSubClass      1 Boot Interface Subclass
    bInterfaceProtocol      2 Mouse
    iInterface              0
        HID Device Descriptor:
        bLength                 9
        bDescriptorType        33
        bcdHID               0.00
        bCountryCode            0 Not supported
        bNumDescriptors         1
        bDescriptorType        34 Report
        wDescriptorLength      46
        Report Descriptors:
        ** UNAVAILABLE **
    Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
        Transfer Type            Interrupt
        Synch Type               None
        Usage Type               Data
        wMaxPacketSize     0x0004  1x 4 bytes
        bInterval              10
```

现在您可以从 Renode 控制鼠标。类型：

```none
(monitor) host.usb MoveMouse 100 100
```

并观察光标在屏幕上移动。

## 现实生活场景：Foboot

在本节中，我们将展示如何运行 Foboot 的模拟 [：Fomu 的 Bootloader](https://github.com/im-tomu/foboot)。

Foboot 在 Fomu 平台上运行，该平台使用 ValentyUSB 内核实现软件驱动的 USB 设备，其中整个逻辑（包括生成 USB 描述符）由 CPU 执行。

### 创建 Fomu 平台

Renode 附带了 Fomu 平台的定义。要创建新的虚拟 Fomu 实例，请在监视器中键入：

```none
(monitor) mach create
(machine-0) machine LoadPlatformDescription @platforms/cpus/fomu.repl
```

加载 Foboot 软件：

```none
(machine-0) sysbus LoadELF @https://antmicro.com/projects/renode/fomu--foboot.elf-s_112080-70b1181d470646a31ebef7300fc8e6dc5447e282
```

创建 USB/IP 服务器并导出 Fomu：

```none
(machine-0) emulation CreateUSBIPServer
(machine-0) host.usb Register sysbus.valenty
```

启动仿真：

```none
(machine-0) start
```

### 在您的主机上使用它

在主机上导入 Fomu：

```sh
$ sudo usbip attach -r 127.0.0.1 -b 1-0
```

使用 `dfu-util` 上传软件：

```sh
$ wget https://antmicro.com/projects/renode/fomu--test_binary_flash.bin-s_1016-4a3c37baf69aeb401f834521b0ac4bc6d157ecdf -O fomu--test_binary_flash.bin
$ sudo dfu-util -D fomu--test_binary_flash.bin
dfu-util 0.9

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2016 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

dfu-util: Invalid DFU suffix signature
dfu-util: A valid DFU suffix will be required in a future dfu-util release!!!
Opening DFU capable USB device...
ID 1209:5bf0
Run-time device DFU version 0101
Claiming USB DFU Interface...
Setting Alternate Setting #0 ...
Determining device status: state = dfuIDLE, status = 0
dfuIDLE, continuing
DFU mode device DFU version 0101
Device returned transfer size 1024
Copying data from PC to DFU device
Download    [=========================] 100%         1016 bytes
Download done.
state(7) = dfuMANIFEST, status(0) = No error condition is present
state(8) = dfuMANIFEST-WAIT-RESET, status(0) = No error condition is present
Done!
```

断开 `dfu-util` 连接以重新启动到上传的软件：

```sh
$ sudo dfu-util -e
```

由于 Fomu 没有很多我们可以观察到的接口，因此上传的二进制文件非常简单。查看日志，您将看到连续值重复写入 0x40000000。

要更详细地分析加载的二进制文件，您可以使用 Renode 的 [GDB 调试功能](../debugging/gdb.md)或广泛的[日志记录支持](../basic/logger.md) 。

