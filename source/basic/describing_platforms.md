# 描述平台

Renode 使用基于文本的格式来描述平台。平台描述文件通常具有 `.repl` 扩展名，但这不是必需的。

格式及其语法的广泛描述可在 [平台描述格式](../advanced/platform_description_format.md) 部分找到。下面我们介绍基本用法和最常见的场景。

## 定义外围设备

要添加外围设备，您需要知道其类型，选择其名称和注册点。大多数外围设备将在 `sysbus` 上注册 - 一个始终可用的外围设备，不必明确定义。

类型名称必须指示外围模型的类。这必须是带有命名空间的全名，但可以省略默认命名空间 `Antmicro.Renode.Peripherals`。

例如，要创建一个类型 `Antmicro.Renode.Peripherals.UART.MiV_CoreUART` 为 的 UART 对象，该对象在 `0x80000000` 处连接到系统总线，请使用：

```none
uart0: UART.MiV_CoreUART @ sysbus 0x80000000
```

一些外围设备，如前面提到的 UART，需要构建参数。REPL 格式允许您设置外围模型的构造函数参数和属性。它们位于声明下方，有四个缩进空格：

```none
uart0: UART.MiV_CoreUART @ sysbus 0x80000000
    clockFrequency: 66000000
```

构造函数参数以小写字母开头，属性以大写字母开头。

## 连接外围设备

在上面的示例中， `uart0` peripheral 连接到特定地址的系统总线。但是，也可以将外设连接到其他总线，如 I2C 或 SPI，连接到 GPIO 控制器等。

例如，要在 `0x80` 处将温度传感器连接到名为 `i2c0` 的 `I2C` 控制器，请键入：

```none
sensor: Sensors.SI70xx @ i2c0 0x80
```

外设也可以通过 GPIO 或中断连接。Renode 以类似方式处理这些信号，并允许您使用 `->` 运算符创建连接。

要将计时器连接到 `plic` 中断控制器上的第 31 个中断，请运行：

```none
timer: Timers.MiV_CoreTimer @ sysbus 0x1000000
    -> plic @ 31
```

## 包含文件

您可以使用 `using` 关键字在平台中包含现有的 REPL 文件。

将在监视器内部 PATH 上的每个目录中查找该路径，该 `PATH` 可通过 `path` 命令进行编辑。默认情况下，`PATH` 包含 Renode 安装目录，因此可以像这样包含与 Renode 一起分发的平台文件：

```
using "platforms/cpus/miv.repl"
```

您还可以提供绝对路径：

```
using "/tmp/platform.repl"
```

或者相对于包含 `using` 语句的 REPL 文件的路径：

```
using "./platform.repl"
using "../other/platform.repl"
```

:::{note}

在 Windows 上，`/` 在所有这些情况下都可以用作路径分隔符。如果你想使用 Windows 样式的 `\` 分隔符，你需要对它们进行转义：

```
using "D:\\platforms\\platform1.repl"
```
:::
