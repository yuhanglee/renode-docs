# 设置有线网络

Renode 允许您使用 Monitor 创建复杂的网络拓扑。

有线和[无线网络](./wireless.md)可以共存，但使用不同的命令创建。

## 创建开关

有线网络接口可以通过交换机连接。根据所需的拓扑，您可以创建一个或多个交换机并相应地连接接口。

网络流量不受连接到一台交换机的节点数的影响。

要创建名为 `switch1` 的交换机，请运行：

```
(monitor) emulation CreateSwitch "switch1"
```

## 连接接口

要将接口连接到交换机，您必须设置适当的[机器上下文](../basic/machines.md#switching-between-machines) 。

然后，使用`连接器`机制连接接口：

```none
(machine-0) connector Connect sysbus.ethernet switch1
```

虽然这不是常见的设置，但每个接口可以同时连接到许多交换机。

## 断开接口

您可以通过运行以下命令来断开网络接口与交换机的连接：

```none
(machine-0) connector Disconnect sysbus.ethernet switch1
```

要断开与所有连接的交换机的连接，请使用：

```none
(machine-0) connector DisconnectFromAll sysbus.ethernet
```

## 启动界面

`Switch` 对象创建为 “paused”。要启用通过开关的通信，您必须通过运行以下命令手动启动它：

```
(monitor) start
```

或者，如果您的仿真已启动，请使用：

```none
(monitor) switch1 Start
```

## 控制流量

如果数据包具有指定的目标 MAC 地址，交换机将尝试将其传送到相应的接口。但是，如果未设置地址或交换机不知道具有此 MAC 的接口，则数据包将广播到所有连接的接口。

您可以通过为指定接口启用混杂模式来更改此行为：

```none
(machine-0) switch1 EnablePromiscuousMode sysbus.ethernet
```

在此命令之后，`sysbus.ethernet` 接口将接收从其他接口传送到 `switch1` 的所有数据包。

要禁用混杂模式，请运行：

```none
(machine-0) switch1 DisablePromiscuousMode sysbus.ethernet
```
