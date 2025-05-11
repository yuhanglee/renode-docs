(basic-control)=

# 基本执行控制

Renode 允许您精确控制仿真的执行。

## 开始和暂停执行

开始时，仿真处于 *暂停*   状态，这意味着没有计算机正在运行， *虚拟时间*  也没有进行。

要启动仿真，请执行：

```none
(machine-0) start
Starting emulation...
```

要暂停它，请键入：

```none
(machine-0) pause
Pausing emulation...
```

## 逐条执行指令

```{note}
尽管使用 Renode 的原生步进是一种可行的解决方案，但我们建议使用 [GDB](https://www.sourceware.org/gdb/) 来执行分步执行。在[专门针对此问题的文档章节](../debugging/gdb.md)中详细介绍了将 GDB 与 Renode 模拟计算机一起使用。

有关如何在 GDB 中执行步骤的更多信息，请参阅 [GDB 文档](https://sourceware.org/gdb/download/onlinedocs/gdb/Continuing-and-Stepping.html) 。
```

当您需要详细分析二进制文件的执行如何影响硬件的状态时，您可以切换到  *单步执行*   模式：

```none
(machine-0) sysbus.cpu ExecutionMode SingleStepBlocking
```

这将在执行每条指令后停止仿真。要移动到下一个，请键入：

```none
(machine-0) sysbus.cpu Step
```

当您想要返回到 *正常* 执行模式时，请键入：

```none
(machine-0) sysbus.cpu ExecutionMode Continuous
```

通过连接外部 GDB 可以获得更高级的控制。

## 阻塞和非阻塞步进

当您使用 `SingleStepBlocking` 模式时，模拟不会在步骤之间进行。这可能会导致在逐条指令执行多个内核时阻止仿真的问题，在这种情况下，`SingleStepNonBlocking` 模式是首选选项。非阻塞模式的缺点是虚拟时间将在步骤之间进行，这可能会引入不同步和超时相关问题。

`Step` 命令将在逐条指令流中保持当前模式，否则使用默认值 （`SingleStepBlocking`）。可以通过显式选择非阻塞模式来覆盖此行为：

```none
(machine-0) sysbus.cpu Step false
```

## 检查当前位置

Renode 允许您轻松检查应用程序的当前状态：

```none
(machine-0) sysbus.cpu PC
0xC01890A8
```

因此，您将获得当前正在执行的指令的十六进制地址。

如果您想知道当前正在执行的函数的名称（假设您的二进制文件已使用内部的符号进行编译），请键入：

```none
(machine-0) sysbus FindSymbolAt `sysbus.cpu PC` # equivalent of 0xC01890A8
uart_console_write
```

这将打印元件的名称。
