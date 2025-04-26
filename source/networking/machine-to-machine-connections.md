# 机器到机器的连接


在 Renode 中有多种方法可以连接两台机器。除了{doc}`wired <wired>`和 {doc}`wireless <wireless>` （在各自的章节中描述）之外，还有其他几种接口可用，分为两类：


- {ref}`对称的，如 UART 或 CAN<symmetrical-connections>`,
- {ref}`非对称的，如 GPIO 或 USB<asymmetrical-connections>`.

(symmetrical-connections)=

## 对称连接

在 Renode 中，UART 和 CAN 等连接始终是对称的，这意味着每一方都可以是传输的发起者或接收者。对称连接由一个 “hub” 对象表示，通信机器需要连接到该对象。

### 基于 UART 的连接

要连接两个 UART 设备，您需要创建一个 UART 集线器，可以通过执行以下命令在 Monitor 中创建：

```none
(monitor) emulation CreateUARTHub "uartHub"
```

此行创建的 UART 集线器名为 `uartHub`，但您可以使用任何名称。

然后，您必须使用`连接器`机制将 UART 设备连接到公共 UART 集线器：

```none
(monitor) mach set 0
(machine-0) connector Connect sysbus.uart uartHub
(machine-0) mach set 1
(machine-1) connector Connect sysbus.uart uartHub
```

要断开设备与 `uartHub` 的连接，请执行：

```none
(machine-1) connector Disconnect sysbus.uart uartHub
```

UART 集线器在创建时处于暂停状态，这意味着它不会在设备之间传输任何数据。要启用通信，您必须启动您的中心。

为简单起见，在默认情况下，您的 `uartHub` 将在开始整个模拟时自动启动：

```none
(monitor) start
```

如果您需要手动启动您的 `uarthub`（例如，将其添加到正在运行的模拟时）：

```none
(monitor) uartHub Start
```

现在，`machine-0` 上的 `sysbus.uart` 发送的数据将由 `machine-1` 上的 `sysbus.uart` 接收，反之亦然。

可以通过执行以下命令随时暂停 hub：

```none
(monitor) uartHub Pause
```

然后可以使用以下方法恢复它：

```none
(monitor) uartHub Resume
```

为了确保仿真的确定性，通过 uartHub 发送的字节被缓冲并在设定的时间传送给接收器。请注意，这可能会导致通信略有延迟。最大延迟可由 `quantum` 参数控制。有关同步的工作原理和配置方法的详细信息，请参阅文档的时间[框架同步部分](../advanced/time_framework.md#synchronization) 。

(can-based-connections)=

### 基于 CAN 的连接

用 CAN 总线连接两台机器与用 UART 连接它们非常相似。

首先，您需要在 Monitor 中创建一个 CAN 集线器：

```none
(monitor) emulation CreateCANHub "canHub"
```

然后，您必须使用`连接器`机制将两个设备都连接到 CAN Hub：

```none
(monitor) mach set 0
(machine-0) connector Connect sysbus.can canHub
(machine-0) mach set 1
(machine-1) connector Connect sysbus.can canHub
```

要断开设备与 `CANHub` 的连接，请执行：

```none
(machine-1) connector Disconnect sysbus.can canHub
```

```{note}
与 UART Hub 类似，CAN Hub 可以使用相同的命令自由启动、暂停和恢复。
```

(asymmetrical-connections)=

## 非对称连接

在 Renode 中，GPIO 和 USB 连接始终是不对称的。这需要您指定哪台计算机是`控制器` ，哪台计算机是`外围设备` 。

### GPIO 连接

Renode 中的 GPIO 连接允许您在两台机器的 GPIO 引脚之间发送布尔信号（`true` 或 `false`）。`GPIOConnector` 对象表示与源和目标的单向 GPIO 连接，这就是 GPIO 被列为非对称连接方法的原因。但是，您可以创建两个源和目标相反的 `GPIOConnector` 对象，以便有效地获得双向连接。

要创建 `GPIOConnector`，请使用：

```none
(monitor) emulation CreateGPIOConnector "gpio-con"
```

您最多可以将两台机器连接到您的 `GPIO 连接器` ，一台是 `INumber GPIO Output`，另一台是 `IGPIOReceiver`。尝试连接更多总是会导致错误。

接下来，您需要选择要在连接中使用的机器中可用的 GPIO 引脚。在下面的示例中，我们选择在`源`计算机中使用引脚 7，在`目标`计算机中使用引脚 4。

```none
(monitor) mach set "source"
(source) connector Connect sysbus.gpio gpio-con
(source) gpio-con SelectSourcePin sysbus.gpio 7

(monitor) mach set "destination"
(destination) connector Connect sysbus.gpio gpio-con
(destination) gpio-con SelectDestinationPin sysbus.gpio 4
```

```{note}
请记住，信号只能从 `SourcePin` 发送到 `DestinationPin`。要创建反向连接，请创建另一个 `GPIOConnector`。
```

### USB 连接

为了能够在 Renode 中的两台机器之间创建 USB 连接，首先，您需要创建一个 `USBConnector`。

```none
(monitor) emulation CreateUSBConnector "usb-connector"
```

在这种类型的连接中，您的计算机被分配了`设备`或`控制器`角色。与其他连接类型一样，您需要使用`连接器`机制将 `USB 连接器`连接到计算机上的 USB 端口。然后，您的设备必须在`控制器`设备中注册并连接到其 USB。

```none
(monitor) mach set "device"
(device) connector Connect sysbus.usb usb-connector

(device) mach set "controller"
(controller) usb-connector RegisterInController sysbus.usb-controller
```

## 对事件的时间进行建模

某些模型要求事件在指定时刻发生（例如，另一个事件之后的某个时间）。为此，您可以使用各种机制：

- 在外设中实现一个 `LimitTimer`，它可以定期执行 action。这主要用于实现定时器逻辑
- `machine.LocalTimeSource.ExecuteInNearestSyncedState` 将事件延迟一段时间，但未明确指定
- `machine.LocalTimeSource.ExecuteInSyncedState`, 这使您可以更好地控制事件发生的具体时间
- `machine.ObtainManagedThread`, 这是以指定频率执行作的最简单方法

所有这些机制都由机器及其时间源处理，因此 logic 对 peripheral model 是隐藏的。

有关 Renode 中 Time 框架的更多详细信息，请参见相关章节： {doc}`Time framework <../advanced/time_framework>`
