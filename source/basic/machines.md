(working-with-machines)=

# 使用机器

Renode 允许轻松处理跨多台机器的仿真。

## 创建机器

一开始，仿真是空的，因为没有要运行的计算机。要添加空 KEY，请执行：

```
(monitor) mach create
(machine-0)
```

这将创建第一台计算机，如果您不给它一个自定义名称，它将从 0 开始索引（因此称为 machine-0）。此命令还会将 Monitor 的上下文切换到此新计算机。

再次执行相同的命令将创建另一台名为 `machine-1` 的机器：

```none
(machine-0) mach create
(machine-1)
```

您可以通过提供自定义名称作为参数来创建具有自定义名称的计算机：

```none
(monitor) mach create "my-machine"
(my-machine)
```

要列出所有已创建的计算机及其名称和索引，请键入：

```none
(my-machine) help mach
```

(machine-context)=

## 在机器之间切换

当您想要将 Monitor 的上下文切换到另一种机器类型时：

```none
(machine-1) mach set "machine-0"
(machine-0)
```

除了计算机的名称，您还可以使用其索引：

```none
(machine-1) mach set 0
(machine-0)
```

(loading-platforms)=

## 加载平台

创建机器后，它只包含一个外围设备 - **系统总线** ，简称为 `sysbus`。没有内存或 CPU，因此计算机尚未准备好执行任何代码。

要列出所有外围设备，请执行：

```none
(machine-0) peripherals
Available peripherals:

  sysbus (SystemBus)
```

要加载预定义的平台（在本例中为 *Microsemi MiV*），请键入：

```none
(machine-0) machine LoadPlatformDescription @platforms/cpus/miv.repl
(machine-0) peripherals
Available peripherals:
sysbus (SystemBus)
│
├── clint (CoreLevelInterruptor)
│     <0x44000000, 0x4400FFFF>
│
├── cpu (RiscV)
│     Slot: 0
│
├── ddr (MappedMemory)
│     <0x80000000, 0x83FFFFFF>
│
├── flash (MappedMemory)
│     <0x60000000, 0x6003FFFF>
│
├── gpioInputs (MiV_CoreGPIO)
│     <0x70002000, 0x700020A3>
│
├── gpioOutputs (MiV_CoreGPIO)
│     <0x70005000, 0x700050A3>
│
├── plic (PlatformLevelInterruptController)
│     <0x40000000, 0x43FFFFFF>
│
├── timer0 (MiV_CoreTimer)
│     <0x70003000, 0x7000301B>
│
├── timer1 (MiV_CoreTimer)
│     <0x70004000, 0x7000401B>
│
└── uart (MiV_CoreUART)
      <0x70001000, 0x70001017>
```

`.repl` （Renode 平台） 文件的格式显示在 [描述平台](./describing_platforms.md) 部分。

(accessing-and-manipulating-peripherals)=
## 访问和作外围设备

当您在 Monitor 中的计算机上下文中时，您可以按名称引用外围设备。您可以读取和写入外围设备的属性，并对其执行一些作。可用属性和作的集合取决于外围设备的类型。

例如，要检查内存大小，请执行：

```none
(machine-0) sysbus.ddr Size
```

要在外围设备上调用作，请使用相同的语法，但将 `Size` 替换为作名称，例如 `ZeroAll`：

```none
(machine-0) sysbus.ddr ZeroAll
```

要获取可用属性或作的完整列表，只需输入外围设备的名称：

```none
(machine-0) sysbus.ddr
The following methods are available:
- Void DebugLog (String message)
- Void Dispose ()
[...]
- Void WriteWordUsingDwordBigEndian (Int64 address, UInt16 value)
- Void ZeroAll ()
Usage:
sysbus.ddr MethodName param1 param2 ...
The following properties are available:
- Int32 SegmentCount
    available for 'get'
- Int32 SegmentSize
    available for 'get'
- Int64 Size
    available for 'get'
Usage:
- get: sysbus.ddr PropertyName
- set: sysbus.ddr PropertyName Value
```

`Usage` 部分描述了访问外围设备功能的正确语法。

## 加载二进制文件

创建并配置平台后，您可以将软件上传到它上面。Renode 允许您运行与真实硬件上完全相同的可执行文件，这意味着无需更改二进制文件或重新编译源代码。


在 Renode 中，您可以使用本地二进制文件或通过 HTTPS 加载它们。如果您没有二进制文件，则可以使用此示例 [Zephyr Shell 示例进行 MiV](https://dl.antmicro.com/projects/renode/shell-demo-miv.elf-s_803248-ea4ddb074325b2cc1aae56800d099c7cf56e592a)。

要将本地 `.elf` 文件加载到内存中，请执行：

```none
(machine-0) sysbus LoadELF @my-project.elf
```

要通过 HTTPS 加载二进制文件：

```none
(machine-0) sysbus LoadELF @https://remote-server.com/my-project.elf
```

````{note}
如果加载多个 ELF 文件，Renode 将查找最后一个 ELF 文件的最低加载部分，以查找初始 PC（或 Cortex-M 计算机上的矢量表偏移量）。这意味着您加载的最后一个 ELF 将设置您的 CPU 起点。

要覆盖此设置，您可以使用以下方法手动设置初始值：

```none
sysbus.cpu PC 0xYourValue
```

或者，在 Cortex-M 上：

```none
sysbus.cpu VectorTableOffset 0xYourValue
```
````

Renode 还支持其他可执行格式，如原始二进制、`UImage` 和 `HEX`。要加载它们，请相应地使用 `LoadBinary`、`LoadUImage` 或 `LoadHEX`。

## 清除仿真

如果你想切换到另一个项目，你可以删除整个仿真：：

```none
(machine-0) Clear
```

所有机器、外围设备和加载的二进制文件都将被删除，Renode 将返回到其初始状态。
