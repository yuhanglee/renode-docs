# 使用 GDB 进行调试

Renode 允许您调试在使用 [GDB](https://www.gnu.org/software/gdb/) 的模拟计算机上运行的应用程序。

它使用 GDB 远程协议，并允许使用最常见的 GDB 函数，如断点、观察点、步进、内存访问等。

与在实际硬件上进行调试最重要的区别是，当仿真 CPU 停止时，虚拟时间不会进行。这使得调试过程对模拟的计算机透明。

## 连接到 GDB

要在端口 3333 上启动 GDB 服务器，请运行：

```none
(machine-0) machine StartGdbServer 3333
```

这允许您从适当的工具链启动 GDB 并连接到远程目标：

```
$ arm-none-eabi-gdb /path/to/application.elf
(gdb) target remote :3333
```


:::{note}
如果存在多个不同架构的 CPU，则此行为可能会有所不同。有关在复杂多核平台上启动 GDB 服务器的信息，请参阅[复杂场景](#complex-scenarios)以了解有关如何继续的详细信息。
:::

## 开始仿真

GDB 连接到 Renode 后，需要启动仿真。简单地告诉 GDB 继续并不足以启动时间流，因为它会破坏更复杂的多节点场景。

有三种方法可以启动整个设置。

您可以从 Renode 的 Monitor 手动启动仿真，键入通常的：

```none
(machine-0) start
```

然后，在 GDB 中运行：

```
(gdb) continue
```

或者，可以使用 GDB 的 `monitor` 命令将命令传递给 Renode 的 Monitor：

```
(gdb) monitor start
(gdb) continue
```

第三个选项适用于最简单的场景，使 Renode 在 GDB 连接后立即启动整个仿真。它需要一个名为 `autostartEmulation` 的 `StartGdbServer` 附加参数：

```none
(machine-0) machine StartGdbServer 3333 true
```

(complex-scenarios)=
## 复杂场景

默认情况下，命令 `StartGdbServer` 会尝试将机器的所有 CPU 添加到新创建的服务器中，前提是所有 CPU 都具有相同的架构。否则，需要使用 `cpuCluster=“cluster-name”` 指定集群，或使用 `cpu=cpuName` 指定特定 CPU：

```none
(machine-0) machine StartGdbServer 3333 true cpuCluster="cortex-r5f"
```

如果提供的集群无效，则将打印出可用集群的列表以供选择。

但是，如果您确定调试器可以处理异构 CPU，只需使用 `cpuCluster=“all”` 将所有可用内核附加到一个存根。您还可以逐个添加集群/CPU：

```none
(machine-0) machine StartGdbServer 3333 true cpuCluster="cortex-r5f"
(machine-0) machine StartGdbServer 3333 true cpu=sysbus.apu0
(machine-0) machine StartGdbServer 3333 true cpu=sysbus.apu2
```

这将导致 GDB 服务器在端口 3333 上运行，并连接来自 `cortex-r5f` 集群的 CPU，另外还连接了名为 `apu0` 和 `apu2` 的 CPU。

还可以将特定 CPU 添加到现有服务器，或使用该 CPU 创建新服务器。这允许您创建更复杂的设置，其中多个 GDB 实例使用不同的 CPU 运行调试会话。

要在端口 3333 上使用一个 CPU 启动 GDB 服务器，还需要两个参数 - 前面提到的 `autostartSimulate` 和 `cpu`：

```none
(machine-0) machine StartGdbServer 3333 true sysbus.cpu1
```

要向该服务器添加第二个 CPU，请运行：

```none
(machine-0) machine StartGdbServer 3333 true sysbus.cpu2
```

要使用另一个 CPU 在端口 3334 上启动新的 GDB 服务器，请运行：

```none
(machine-0) machine StartGdbServer 3334 true sysbus.cpu3
```

这些命令将为您提供一个由两个 GDB 服务器组成的设置 - 在端口 3333 上有两个 CPU，在端口 3334 上有一个 CPU。

此外，`StartGdbServer` 命令将禁止您将一个 CPU 添加到多个 GDB 服务器。

如果通过提供 `autostartEmulation` 和 `cpu` 参数将 CPU 添加到 GDB 服务器，则无法在该计算机上运行该命令的常规版本。
