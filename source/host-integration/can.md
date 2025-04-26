# CAN 集成

```{note}
此功能仅在 Linux 上可用。
```

Renode 可以使用内部 SocketCAN 桥接器连接到主机上的虚拟 CAN 接口。这种集成依赖于 [SocketCAN](https://www.kernel.org/doc/html/latest/networking/can.html) 来实现主机和内部 CAN 网络之间的通信。

使用这种标准化的通信方法，Renode 可以传输经典和 FD CAN 帧。XL CAN 帧由网桥处理，但它们不会转发到内部网络，因为 Renode 尚不支持这种帧类型。

要查看 CAN 主机集成的实际应用，您可以尝试 [SocketCAN 桥接演示](https://github.com/renode/renode/tree/master/scripts/complex/socketcan_bridge) 。

## 主机要求

该桥接器依靠 SocketCAN 连接到 CAN 接口，其中包括本机、虚拟和基于 SLCAN 的接口。以下示例假定使用虚拟 CAN 接口，其驱动程序在 `vcan` 内核模块中实现。要确保 host 中存在该模块，请加载它：

```text
$ sudo modprobe vcan
```

要创建并设置名为 `vcan0` 的虚拟网络接口，请运行：

```text
$ sudo ip link add dev vcan0 type vcan
$ sudo ip link set up vcan0
```

```{note}
根据 Linux 发行版的不同，`vcan` 模块可能不包括在内，或者它可能不支持 FD 和/或 XL CAN 帧。
```

## 创建 SocketCAN 网桥

SocketCAN 网桥连接到已经存在的 CAN 接口，其名称可以作为 `canInterfaceName` 参数提供。如果未提供，Renode 将尝试连接到名为 `vcan0` 的接口。

在仿真中，网桥可以连接到内部网络，如{ref}`基于 CAN 的连接 <can-based-connections>`中所述。

默认情况下，Renode 将尝试启用 FD 和 XL CAN 帧的处理，但不需要它们。要强制使用指定类型，请将可选参数 `ensureFdFrames` 和 `ensureXlFrames` 设置为 `true`。通过设置这些，无论创建方法如何，如果无法启用确保的框架类型，则命令或平台构建都将失败。

```{warning}
在 CAN 网络拓扑中创建循环时要小心。

从一个网桥发送的帧可以由另一个网桥通过通用 vcan 接口接收。如果这两个网桥属于同一网络，则可能会创建数据包的无限循环。
```

### 在 Monitor 中

如上所述，有两种方法可以创建 `SocketCANBridge`。第一个选项是使用 `CreateSocketCANBridge` 命令。

例如，下面的命令将创建一个名为 `socketcan` 的 `SocketCANBridge`，它连接到 `vcan1` 接口，如果无法处理 FD 帧，它将失败。

```text
(machine-0) machine CreateSocketCANBridge "socketcan" "vcan1" ensureFdFrames=true
```

该桥可以连接到 CAN 总线，如 {ref}`基于 CAN 的连接 <can-based-connections>` 中所述。

### 在平台描述中

另一种选择是将 SocketCAN bridge 声明添加到 `repl` 文件中。下面的示例展示了一个代码片段，该代码片段使用默认值设置 `SocketCANBridge` 的 `socketcan` 实例的所有参数。

```text
socketcan: CAN.SocketCANBridge @ sysbus
    canInterfaceName: "vcan0"
    ensureFdFrames: false
    ensureXlFrames: false
```

该桥可以连接到 CAN 总线，如 {ref}`基于 CAN 的连接 <can-based-connections>` 中所述。
