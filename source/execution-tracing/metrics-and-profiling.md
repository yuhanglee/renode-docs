# 执行指标、分析和作码计数

Renode 允许您定制执行跟踪功能，以满足指标、分析和作码计数方面的特定需求。

## 执行指标

Renode 可以收集并以图表的形式向您展示指标。开箱即用的指标包括：

* Executed instructions   已执行的指令
* Exceptions  异常
* Memory access  内存访问
* Peripheral access   外围设备访问

如果您想了解有关此功能的更多信息并查看其实际作，请访问 [Metrics analyzer 章节](../basic/metrics.md) 。

## 来宾应用程序分析

在 Renode 中，您可以以交互式火焰图的形式显示来宾应用程序执行的跟踪。您可以使用它来可视化应用程序流，分析每个函数所花费的相对时间，并发现已实现的功能或其性能的潜在问题。

```{note}
此功能适用于 RISC-V、Cortex-A、Cortex-R 和 Cortex-M CPU。
```

要启用 Guest 分析，请使用：

```none
cpu EnableProfiler <output-format> @path-to-output-file [<flush-instantly>]
```

Renode 支持两种输出格式：

* `CollapsedStack` - 一种基于文本的折叠堆栈格式，可以使用 [speedscope](https://www.speedscope.app/) 以交互方式查看
* `Perfetto` - 可以使用 [Perfetto](https://ui.perfetto.dev/) 查看的二进制格式

您还可以使用 `cpu EnableProfilerCollapsedStack` `和 cpu EnableProfilerPerfetto` 命令来实现相同的效果。

默认情况下，Renode 将缓冲文件作以获得更好的性能，但您可以通过将 `true` 作为可选的 `<flush-instantly>` 参数传递来立即将所有内容写入文件。如果您不想立即写入文件，但仍希望对刷新发生的时间进行一些控制，请使用以下命令手动刷新缓冲区：

```none
cpu FlushProfiler
```

您可以在 [System Designer](https://designer.antmicro.com/) 平台中检查从 Zephyr 样本生成的迹线 - 请参阅 [RISC-V](https://designer.antmicro.com/hardware/devices/hifive1) 和 [Cortex-M](https://designer.antmicro.com/hardware/devices/stm32f103_mini) 的示例。

## 作码计数

Renode 可以计算特定指令被执行了多少次。如果您想知道程序中是否执行了一些特殊指令（例如，vector 指令），这会很有帮助。

要启用作码计数，请使用以下命令：

```none
cpu EnableOpcodesCounting true
```

然后，您需要安装要跟踪的作码的模式。

作码模式必须由以下字符构建：

* `1` 匹配设置位
* `0` 匹配未设置的位
* 所有其他字符都匹配任何值

这些模式类似于与作码的位匹配的正则表达式。例如，您可以使用 [scripts/single-node/versatile.resc](https://github.com/renode/renode/blob/master/scripts/single-node/versatile.resc) 演示来检测 ARM 的 branch 和 branch-with-link 指令。要加载此演示，请使用：

```none
include @scripts/single-node/versatile.resc
```

要安装计数器模式，此示例使用以下两个命令，但您可以搜索所需的任何模式：

```none
cpu InstallOpcodeCounterPattern "bl" "****1011************************"
cpu InstallOpcodeCounterPattern "b"  "****1010************************"
```

要将计数器的值打印到 Monitor （监视器） 窗口，请使用：

```none
cpu GetAllOpcodesCounters
```

第一个参数是计数器的名称，第二个参数是模式。运行演示后，您可以使用：

```none
----------------
|Opcode|Count  |
----------------
|bl    |376816 |
|b     |9587263|
----------------
```

您还可以使用以下方法获取指定计数器的值：

```none
cpu GetOpcodeCounter "<counter-name>"
```

### 安装 RISC-V 作码模式

Renode 为 RISC-V 提供预定义功能，为您安装模式。要安装特定于 RISC-V 的作码模式，请使用以下函数：

* `cpu EnableRiscvOpcodesCounting` - 安装模式 获取一般说明
* `cpu EnableCustomOpcodesCounting` - 为自定义 RISC-V 指令安装模式
* `cpu EnableVectorOpcodesCounting` - 安装 RISC-V 载体指令的模式

这些模式是从 RISC-V 作码定义[文件](https://github.com/renode/renode-infrastructure/tree/master/src/Emulator/Cores/RiscV/opcodes)创建的。您可以按照说明为 RISC-V 创建一个类似的文件，并使用以下方法加载模式：

```none
cpu EnableRiscvOpcodesCounting @path-to-file
```
