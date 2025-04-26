# 设置无线网络

在 Renode 中连接无线网络中的节点就像创建[有线网络](./wired.md)一样简单。

就像使用 `Switch` 对象一样，您可以创建 `Wireless Medium` 的抽象，以将无线电接口连接到该抽象。

## 创建无线介质

Renode 允许您创建多个彼此独立的虚拟无线媒体。交换的数据包不会在这些媒体之间传输，因此您可以将此机制视为物理分离的方法，这可能有助于您调试无线设置。

网络流量不受连接到一个无线介质的节点数的影响。

Renode 支持两种类型的无线媒体：`IEEE802_15_4Medium` 和 `BLEMedium`。每个 API 都会创建不同类型无线连接的 abstration。他们相应地使用 IEEE802_15_4 标准和低功耗蓝牙。

要创建名为 `wireless` 的 IEEE802_15_4 介质，请运行：

```none
(monitor) emulation CreateIEEE802_15_4Medium "wireless"
```

要创建名为 `wireless` 的 IEEE802_15_4 介质，请运行：

```none
(monitor) emulation CreateBLEMedium "wireless"
```

## 连接接口

要将接口连接到无线介质，您必须设置适当的[计算机上下文](../basic/machines.md#switching-between-machines)

然后，使用`连接器`机制连接接口：

```none
(machine-0) connector Connect sysbus.radio wireless
```

虽然这不是常见的设置，但每个接口可以同时连接到许多媒体。

## 断开接口

您可以通过运行以下命令来断开网络接口与无线介质的连接：

```none
(machine-0) connector Disconnect sysbus.radio wireless
```

要断开与所有连接的无线媒体的连接，请使用：

```none
(machine-0) connector DisconnectFromAll sysbus.radio
```

## 定位节点

无线介质使用 3D 坐标（不带任何指定距离单位）来定位连接的节点。

要将节点的位置设置为坐标 {X = 3， Y = 5， Z = -8.5}，请运行：

```none
(machine-0) wireless SetPosition sysbus.radio 3 5 -8.5
```

同样，这些坐标的单位尚未确定，因此用户有责任在仿真中保持它们一致。

## 控制流量

默认情况下，无线介质将所有数据包传送到所有连接的接口。但是，这可以根据您的需要进行配置。

Renode 公开了 `Medium Functions` 的抽象。每个 medium 函数可以接受一组参数来决定两个节点之间交换的数据包（知道发送方和接收方的位置）是否能够成功传递。

Renode 默认提供三种 medium 功能：

- `SimpleWirelessFunction`

  默认设置，将所有数据包传送到所有连接的节点。

- `RangeWirelessFunction`

  此函数接受一个参数，该参数指示节点之间的最大笛卡尔范围。

  如果节点在此范围内，则所有数据包都投递成功。如果它们不在范围内，则无法进行通信。

- `RangeLossWirelessFunction`

  此功能引入了数据包的概率丢失，该数据包的丢失会随着节点之间的距离而逐渐增加。

  给定三个参数 `lossRange`（以距离为单位）、`txRatio` 和 `rxRatio`（范围从 0 到 1.0，包括 0 和 1.0），每个数据包都要接受两次测试。

  第一个测试确定传输是否成功（概率为 `p >> txRatio`）。

  第二个测试取发送方和接收方之间的距离，并根据以下公式计算成功率： `1 - ((distance/lossRange) * (1 - rxRatio))` 。

:::{note}
请记住，即使 `RangeLossWirelessFunction` 依赖于概率，您仍然可以使用`仿真 SetSeed` 配置 RNG 种子，以保持执行的确定性。
:::
