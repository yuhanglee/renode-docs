# Renode 中的低功耗蓝牙仿真

低功耗蓝牙是一种广泛使用的无线协议，最常用于血压监测仪、可穿戴设备和智能家用电器等消费类设备。由于其多节点网络功能，Renode 可用于使用 BLE 在这些领域和其他领域开发和测试完整的产品。

Renode 对 BLE 的支持主要是在 [Zephyr RTOS](https://docs.zephyrproject.org/latest/introduction/index.html) 和 Nordic 的 [nRF52840 SoC](https://www.nordicsemi.com/Products/nRF52840) 的上下文中开发的。

## 运行预编译的 demo

您可以从预编译的 demo 开始，该 demo 将只运行两个通过 BLE 通信的节点。在 Monitor 中使用以下命令执行此作：

```none
(monitor) include @scripts/multi-node/nrf52840-ble-zephyr.resc
```

您应该看到两个串行端口打开并正在发送数据包。

## 构建你自己的 Zephyr 示例

您可以在 [Zephyr 文档中](https://docs.zephyrproject.org/latest/samples/index.html)查看 Zephyr 示例和演示的完整列表，其中更详细地描述了如何安装和使用 RTOS。

在这里，我们将重点介绍本演示特有的一些基本要素。

要构建相关示例， [在像往常一样设置 Zephyr](https://docs.zephyrproject.org/latest/getting_started/index.html) 后，您可以使用以下命令。

```sh
cd ~/zephyrproject/zephyr
west build -b nrf52840dk_nrf52840 -d central samples/bluetooth/central_hr
cp central/zephyr/zephyr.elf ./zephyr-ble-central_hr.elf

west build -b nrf52840dk_nrf52840 -d peripheral samples/bluetooth/peripheral_hr
cp peripheral/zephyr/zephyr.elf ./zephyr-ble-peripheral_hr.elf
```

```{note}
如果要运行新的生成并删除先前生成的副产品，则应将 `-p auto` 参数添加到命令中。
```

要在 Renode 附带的演示中使用这些二进制文件，您可以在加载脚本之前覆盖相关变量：

```none
(monitor) $central_bin=@zephyr-ble-central_hr.elf
(monitor) $peripheral_bin=@zephyr-ble-central_hr.elf
(monitor) include scripts/multi-node/nrf52840-ble-zephyr.resc

```

## 查看脚本

您可以在下面找到一个完整的示例，该示例创建了 2 个通过 BLE 生成和读取心率监测器数据的设备：

```none
using sysbus

$central_bin?=@zephyr-ble-central_hr.elf
$peripheral_bin?=@zephyr-ble-central_hr.elf

emulation CreateBLEMedium "wireless"

mach create "central"
machine LoadPlatformDescription @platforms/cpus/nrf52840.repl
connector Connect sysbus.radio wireless

showAnalyzer uart0

mach create "peripheral"
machine LoadPlatformDescription @platforms/cpus/nrf52840.repl
connector Connect sysbus.radio wireless

showAnalyzer uart0

emulation SetGlobalQuantum "0.00001"

macro reset
"""
    mach set "central"
    sysbus LoadELF $central_bin

    mach set "peripheral"
    sysbus LoadELF $peripheral_bin
"""
runMacro $reset

echo "Script loaded. Now start with the 'start' command."
echo ""
```

该脚本负责创建两台计算机，打开它们的 UART 分析器，将它们连接到单个网络中，并加载提供的二进制文件。

第一台 Renode“机器” - 称为 `central` - 运行 [central_hr 样本](https://github.com/zephyrproject-rtos/zephyr/tree/main/samples/bluetooth/central_hr) ，该样本使用 BLE 查找有源心率监测器，并连接到信号最强的设备。第二个 - `外围设备` - 运行 [peripheral_hr 样本](https://github.com/zephyrproject-rtos/zephyr/tree/main/samples/bluetooth/peripheral_hr) ，这将创建一个用作心率监测器并生成虚拟心率值的机器。

建立连接后，`central` 将报告来自`外围设备`的数据接收。

要了解有关特定命令的更多信息，请参阅 [Renode 存储库上的](https://github.com/renode/renode/blob/master/scripts/multi-node/nrf52840-ble-zephyr.resc) “ [使用计算机](https://renode.readthedocs.io/en/latest/basic/machines.html) ”章节和脚本中的注释。


## 使用 BLE 的数据包拦截钩子

您可以通过 Python 钩子使模拟机器对无线电介质上出现的无线网络（例如 BLE）数据包做出反应。这样的 hook 将直接从 Monitor 或指定文件执行 Python 代码。要在计算机上设置上述示例中的数据包拦截钩子，您可以运行：

```none
(peripheral) wireless SetPacketHookFromScript radio "self.DebugLog('Received a packet of {} bytes'.format(len(packet)))"
```
