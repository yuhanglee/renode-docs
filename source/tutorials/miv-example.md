# Microsemi Mi-V 示例

在本教程中，您将学习如何使用 RISC-V CPU 设置 Microsemi 的单个 Mi-V 板的全功能仿真。它涵盖的主题范围从安装 Renode，到基本命令，以及与 [Microsemi 的 SoftConsole IDE 集成](#softconsole-integration)的过程。

虽然本教程侧重于单个 Mi-V 平台，但 Renode 支持在同一仿真中运行多个设备 - 将这个场景扩展到多个互连的节点应该相当简单。

## 安装

Renode 框架托管[在 GitHub 上](https://github.com/renode/renode) 。您可以按照 [README](https://github.com/renode/renode/blob/master/README.md) 文件中描述的过程下载并安装它，但最简单的开始方法是从 [https://github.com/renode/renode/releases](https://github.com/renode/renode/releases) 下载二进制版本。预编译包以 deb 和 rpm 包（适用于 Linux）、dmg 包（适用于 macOS）和 zip 存档（适用于 Windows）的形式提供。

## 启动 Renode

与 Renode 交互的众多方式之一是通过其命令行界面。

安装 Renode 后，您应该能够从命令行运行 `renode` 命令。或者，查找 Renode.exe 二进制文件。启动 Renode 后，您将看到两个窗口 - 一个用于 Renode 的智能记录器，另一个用于称为“监视器”的 CLI。

在此窗口中，您将能够创建和控制整个仿真环境。

## 脚本

虽然您可以交互式地键入所有命令，但最好将它们分组到可重用的 Renode 脚本中，这些脚本通常具有“.resc”扩展名。它们的目的是加载二进制文件、设置启动条件、准备环境、将机器连接到网络等。

在本文中，我们将使用 Renode 包中提供的脚本，名为 {script}`scripts/single-node/miv.resc <single-node/miv.resc>` 有关您将运行的特定脚本的详细信息，请检查 Renode 安装目录中的文件（例如，对于 Linux，它是“/opt/renode/scripts”）。

## 加载我们的设置

提供的脚本将创建一台计算机并加载一个基于 LiteOS 的示例应用程序。

要运行脚本，请使用 `include` 命令（或简称 `i`），并指定要加载的脚本的路径，前面加上 `@` 符号，如下所示：

```none
include @scripts/single-node/miv.resc
```

加载脚本后，您将看到一个新的终端 - 一个由 `showAnalyzer` 命令打开的 UART 窗口。

在提供的脚本中，我们使用的是在线托管的预编译二进制文件，但您可以通过在加载脚本之前设置 `$bin` 变量来提供自己的二进制文件：

```none
$bin=@path/to/application.elf
```

仿真现在已加载，但尚未启动。您可以使用 `start` 和 `pause` - 以及其他命令来控制它，如下一节所述。

## 简单的命令

### 开始和暂停

要控制模拟是否正在运行，请使用 `start` 和 `pause`。

### 机器

在提供的脚本中，我们使用 `mach create` 命令来创建机器。这将切换 Monitor 中的上下文。所有后续命令都针对当前计算机执行。

要更改机器，请使用 `mach set` 命令。使用机器的数字或名称，例如 `mach set 1` 或 `mach set “MI-V”。`

可以使用 `mach` 命令列出所有计算机。要清除当前选择，请使用 `mach clear`。

### 访问外围设备

所有外围设备都可以在 Monitor 中访问，并且它们的大部分方法和属性都向用户公开。要列出所有可用的外围设备，请使用 `peripherals` 命令。

### 外围方法和属性

要访问外围设备，您必须提供其完全限定名称。所有外设都注册在 `sysbus` 中，因此请使用 `sysbus.uart` 访问 UART，或使用 `sysbus.gpioOutputs.led0` 访问其中一个 LED。

大多数提供的演示中都使用了 `using sysbus` 命令，允许您删除 `sysbus.` 前缀。

键入外围设备的名称将为您提供可用方法、字段和属性的列表。该列表是自动生成的，因此大多数可访问成员都不是为最终用户设计的。

该列表显示了每种成员类型的正确 Monitor 语法的示例。

### 其他命令

要查找有关内置 Monitor 命令的信息，请键入 `help` 并参阅文档。运行 `help builtin_command_name` 将打印出给定命令的帮助。

## 调试和检查

Renode 为您提供了多种验证应用程序行为的方法。由于对环境具有完全控制权，您可以以对模拟应用程序 100% 透明的方式添加日志记录、事件钩子、交互式代码调试等。

在这里，我们将仅介绍一些可用的调试选项。

### 函数名称日志记录

当应用程序卡住或行为异常时，检查函数调用的跟踪始终是一个好主意。要在选定的机器中启用函数名称的日志记录，请运行 `cpu LogFunctionNames true` （和 `false` 分别禁用它）。

由于记录的数据量可能太大而无法使用，因此您可以将记录的函数筛选为以指定前缀开头的函数。例如 `cpu LogFunctionNames true "UART_ LOS_"` ，将仅记录以 “UART_” 或 “LOS_” 前缀开头的函数。

### 外设访问日志记录

如果您的驱动程序行为不正确，最好调查与它所控制的设备的通信。要启用 CPU 和 UART 外设之间每次交互的日志记录，请运行 `sysbus LogPeripheralAccess uart` 。

此功能仅适用于直接在 system bus 上注册的外围设备。

### GDB

一种流行的调试工具 GDB 可用于分析在 Renode 中运行的应用程序。它使用与 OpenOCD 相同的远程协议，因此可以轻松地与大多数基于 GDB 的 IDE 集成，例如 SoftConsole 或 Eclipse。要在 Renode 中启动 GDB 存根，请运行`计算机 StartGdbServer 3333`（其中 3333 是示例端口号）并通过调用 `（gdb） target remote ：3333` 从 GDB 进行连接。要启动仿真，您必须同时运行 `start` in Renode 和 `continue` in GDB。

您可以使用 GDB 的大部分常规功能：断点、观察点、步进、读/写变量等。您还可以使用 GDB 中的 `monitor` 命令将命令直接发送到 Renode CLI（以避免在两个控制台窗口之间切换）。

## SoftConsole  集成

Renode 的主要目标之一是轻松与用于开发人员日常工作的工具集成。此类工具的一个很好的示例是 [SoftConsole，它是 Microsemi 的一个基于 Eclipse 的 IDE](https://www.microsemi.com/product-directory/design-tools/4879-softconsole)。

SoftConsole 提供了调试功能，这些功能在连接到硬件时可以正常使用。通过更改项目设置，您可以将其连接到 Renode。

首先在 renode 中运行 GDB 服务器：

```none
(monitor) include @scripts/single-node/miv.resc
(MI-V) machine StartGdbServer 3333 true
```

请注意 `true` 参数 - 它强制 Renode 在 GDB 客户端连接后立即自动启动。

现在，您需要在 SoftConsole 中配置调试配置。

在 Project Explorer 中，右键单击您的项目名称，选择 `Debug As` 和 `Debug Configurations...`。

![image](miv/softconsole-debug.png)

这将打开一个窗口，您需要在其中打开 `Debugger` 选项卡。在那里，取消选中 `Start OpenOCD locally` 复选框，因为 Renode 的作用与 OpenOCD 通常相同。

![image](miv/softconsole-openocd.png)

您必须验证 `Remote Target` 部分中的远程端口号是否与 `StartGdbServer` 命令中提供的端口号相同。

完成这些更改后，您现在可以单击 `Debug`。SoftConsole 将连接到 Renode，仿真将自动启动。

默认情况下，您将在被命中的`主`函数的开头观察到一个断点。

在 SoftConsole 中，您可以添加自己的断点、检查和更改变量以及单步执行代码，同时仍然能够通过 Monitor 以通常的方式与 Renode 进行交互。

![image](miv/softconsole-breakpoint.png)
