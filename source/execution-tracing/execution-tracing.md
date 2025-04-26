# Renode 中的执行跟踪

Renode 完全了解所有模拟组件的内部状态，并且可以使用此信息来跟踪未修改的二进制文件的执行，而无需对模拟进行任何行为更改。

```{note}
某些日志记录功能可能会显著影响模拟的性能，但不会影响执行本身。
```

```{note}
下面列出的命令假定您加载了一个平台、一个名为 `cpu` 的 CPU 节点，并且执行了 `using sysbus` 命令。有关更多详细信息，请参阅有关 [Renode Script 语法](renode-script-syntax)和[访问和作外围设备](accessing-and-manipulating-peripherals)的文档章节。
```

## 记录已执行的函数名称

Renode 可以记录 guest 应用程序当前执行的函数的名称。这要求您使用 `sysbus LoadELF @path/to/elf` 命令来[加载您的应用程序](https://renode.readthedocs.io/en/latest/basic/machines.html#loading-binaries) ，或者 `sysbus LoadSymbolsFrom @path/to/elf` 如果您更喜欢执行二进制或十六进制文件。可以使用以下方法启用函数名称日志记录：

```none
cpu LogFunctionNames true
```

您可以使用以下方法禁用函数名称日志记录：

```none
cpu LogFunctionNames false
```

```{note}
您还可以在此命令的末尾添加另一个 `true`，以从后续代码块中删除重复的函数名称并获得更好的整体性能。
```

要根据前缀筛选函数名称，请在函数末尾添加前缀作为字符串：

```none
cpu LogFunctionNames true "uart"
```

要使用多个前缀和以这两个前缀中的任何一个开头的 log 函数，只需用空格分隔它们即可：

```none
cpu LogFunctionNames true "uart irq"
```

```{note}
Renode 将尝试解缠 C++ 函数名称。它还将使用存储在 ELF 文件中的名称。
```

如果您想了解有关在 Renode 中记录已执行函数名称的更多信息，请访问[使用 logger 一章](../basic/logger.md)

## 记录外围设备访问

虽然有关已执行函数的信息可以让您全面了解执行情况，但了解其他上下文并了解应用程序为何采用特定路径通常是有益的。

为了提供这个额外的上下文，Renode 可以记录对外围设备的访问。此功能允许您查看程序如何使用或不使用 SoC 的特定部分。

您可以使用以下方法启用此功能：

```none
sysbus LogPeripheralAccess <peripheral-name> true
```

```{note}
在大多数情况下，您的外围设备名称需要一个前缀 `sysbus`，例如 `sysbus.uart0`。如果您在脚本中使用 `using sysbus` 命令，则可以省略此前缀。
```

日志包含：

- 外围设备的名称
- 程序计数器的当前值
- 访问的类型和宽度
- 访问的偏移量（相对于外设的基址）和此偏移量映射到的寄存器的名称
- 加载到 register 或从 register 返回的值

您还可以记录对连接到系统总线的所有外围设备的访问：

```none
sysbus LogAllPeripheralsAccess true
```

您可以通过提供 `false` 而不是 `true` 作为最后一个参数来禁用这两个外围设备访问日志记录命令：

```none
sysbus LogAllPeripheralsAccess false
```

如果您想了解有关在 Renode 中记录外围设备访问的更多信息，请访问[使用 logger 一章](../basic/logger.md)

## 执行跟踪

在 Renode 中，您可以查看 CPU 在任何给定时间执行的作，而无需更改代码或使用专用硬件。要启用执行跟踪，请使用：

```none
cpu CreateExecutionTracing "tracer_name" @path-to-file <mode>
```

此外，您还可以使用跟踪器来跟踪内存访问。为此，请键入：

```none
tracer_name TrackMemoryAccesses
```

同样，要跟踪 RISC-V 架构的向量配置，请使用：

```none
tracer_name TrackVectorConfiguration
```

`mode` 可以是以下值之一：
- `PC` - 此模式保存所有程序计数器值。例：

```
0x20400000
0x20400004
0x20400008
0x2040000c
0x20401bc4
0x20401bc8
...
```
- `Opcode` - 此模式保存所有已执行的作码。例：

```
0x0297
0x1028293
0x30529073
0x3B90106F
0x5FC00297
0x88C28293
...
```
- `PCAndOpcode` - 此模式保存程序计数器值和执行的相应作码。例：

```
0x20400000: 0x0297
0x20400004: 0x1028293
0x20400008: 0x30529073
0x2040000c: 0x3B90106F
0x20401bc4: 0x5FC00297
0x20401bc8: 0x88C28293
...
```
- `反汇编` - 除了程序计数器的值和相应的作码外，此模式还使用内置的基于 LLVM 的反汇编器将作码转换为具有所有使用参数的人类可读指令名称。输出还包括此条目所属的元件的名称。例：

```
0x20400000:   00000297  auipc t0, 0           [vinit (entry)]
0x20400004:   01028293  addi t0, t0, 16       [vinit+0x4 (guessed)]
0x20400008:   30529073  csrw mtvec, t0        [vinit+0x8 (guessed)]
0x2040000c:   3b90106f  j 7096                [vinit+0xC (guessed)]
0x20401bc4:   5fc00297  auipc t0, 392192      [__start (entry)]
0x20401bc8:   88c28293  addi t0, t0, -1908    [__start+0x4 (guessed)]
...
```

您可以将 `PC`、`Opcode` 和 `PCAndOpcode` 模式的输出保存为可以选择压缩的二进制格式。这种格式的编码速度更快，并生成更小的输出文件。要保存到二进制文件，请使用以下命令：

```none
cpu CreateExecutionTracing "tracer_name" @path-to-file <mode> true
```

如果你还想压缩输出，你可以向此命令添加另一个 `true`：

```none
cpu CreateExecutionTracing "tracer_name" @path-to-file <mode> true true
```

您可以使用与 Renode 捆绑的脚本查看二进制文件的内容。可以通过在 shell 中运行以下命令来调用它：

```sh
python3 <renode>/tools/execution_tracer/execution_tracer_reader.py inspect path-to-dump-file
```

此命令会将文件的文本内容打印到标准输出。

要禁用执行跟踪，只需使用：

```none
cpu DisableExecutionTracing
```

## 使用收集的数据

Renode 附带的某些工具可以使用获取的执行跟踪对程序的执行进行事后分析。这些包括：
*  [Coverage Report Generator](./coverage-report.md) 可用于生成代码覆盖率报告，
*  [Guest CPU Cache Modelling tool](./guest-cache-modelling.md) 用于模拟 CPU 缓存并生成使用情况统计信息。

## 执行跟踪的特殊用途

除了通用的程序执行跟踪之外，Renode 还具有更专业的跟踪相关工具，例如：
* [对执行的作码进行计数](./metrics-and-profiling.md#opcode-counting)
* [生成客户机执行跟踪的交互式火焰图](./metrics-and-profiling.md#guest-application-profiling)
* [获取执行指标](../basic/metrics.md)

有关更多详细信息 [，请参阅 执行指标、分析和作码计数](./metrics-and-profiling.md) 部分。
