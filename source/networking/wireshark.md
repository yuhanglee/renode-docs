# 使用 Wireshark 检查流量

[Wireshark](https://www.wireshark.org/) 是一个开源网络数据包分析器，可用于嗅探模拟节点和/或主机网络接口之间的网络流量。

Renode 使用 libpcap 格式向 Wireshark 提供数据。

## 记录整个流量

Renode 支持多种链路层协议，即以太网、低功耗蓝牙、IEEE 802.15.4 和 CAN。每个协议都有自己的一组命令，用于启用所有流量的日志记录或筛选。

这些 `emulation Log<Protocol Name>Traffic` 命令会自动将 Wireshark 连接到给定协议的所有现有和新网络。

```{list-table} Protocol specific global logging
:header-rows: 1

* - Protocol
  - Command
* - Ethernet
  - LogEthernetTraffic
* - Bluetooth Low Energy
  - LogBLETraffic
* - IEEE 802.15.4
  - LogIEEE802_15_4Traffic
* - CAN
  - LogCANTraffic
```

您也可以在设置网络之前手动打开 Wireshark 窗口：

```text
(monitor) host.wireshark-all<Protocol Name>Traffic Run
```

## 观察特定接口

可以使用`仿真 LogToWireshark` 命令检查特定交换机、总线或无线介质的流量。您还可以将观察限制为连接到该交换机、总线或介质的特定接口。


```{list-table} Protocol specific filtered logging
:header-rows: 1

* - Protocol
  - Interface type
* - Ethernet
  - Switch
* - Bluetooth Low Energy
  - BLEMedium
* - IEEE 802.15.4
  - IEEE802_15_4Medium
* - CAN
  - CANHub
```

### 以太网示例

要在 `switch` 对象上运行日志记录，请执行以下作：

```text
(monitor) emulation LogToWireshark switch
```

要仅观察连接到`交换机`的 `sysbus.ethernet` 接口，请运行：

```text
(machine-0) emulation LogToWireshark switch sysbus.ethernet
```

创建的 Wireshark 对象的名称取决于计算机名称、交换机名称和接口名称。在上述情况下，Renode 会创建一个名为 `host.wireshark-switch-machine-0-sysbus-ethernet` .
