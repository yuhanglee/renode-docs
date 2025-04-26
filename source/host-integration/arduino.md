# Arduino IDE/CLI 集成

Renode 支持与 [Arduino IDE](https://www.arduino.cc/en/software) 集成，从而可以直接从 IDE 轻松上传和运行针对基于 Arm 的 Arduino 板（目前支持 Arduino Nano 33 BLE）的软件。相同的接口也可用于使用 [Arduino CLI](https://www.arduino.cc/pro/cli) 从控制台上传二进制文件。

本文档介绍了如何在 TensorFlow Lite 的 Hello World 项目示例中使用集成层，但这也适用于任何自定义项目。

:::{note}
Renode-Arduino 集成层仅在 Linux 上可用。
:::

## 安装 TensorFlow 示例

TensorFlow Lite 示例未预装 Arduino IDE。要将它们添加到您的系统中，请选择`工具 -> 管理库`并搜索 “Arduino_TensorFlowLite” 库。选择 2.1.0-ALPHA 版本（ **不是**预编译版本），然后按 `Install` 按钮：

![image](img/arduino_ide_libraries.png)

目前，Renode 中对 `USBSerial` 的支持仍处于试验阶段，因此您需要使用以下命令修补库：

```sh
sed -i'' '/#define DEBUG_SERIAL_OBJECT/s/(Serial)/(Serial1)/' ~/Arduino/libraries/Arduino_TensorFlowLite/src/tensorflow/lite/micro/arduino/debug_log.cpp
```

这会将串行输出从 USBSerial 切换到 UART。

:::{note}
请确保已安装的 Arduino 库的路径正确。
:::

(ard-configuring-renode)=

## 配置 Renode

Renode 提供了 [ArduinoLoader](https://github.com/renode/renode/blob/master/src/Renode/Integrations/ArduinoLoader.cs) 伪设备，用于与 Arduino IDE/CLI 集成。它充当 USB 设备（CDC/ACM 配置文件）并实现 arduino 引导加载程序协议 （SAM-BA）。

要将 Renode 连接到 Arduino IDE/CLI，请执行以下步骤：

1. 在主机中启用对 USB/IP 的支持：

   ```sh
   $ sudo modprobe vhci_hcd
   ```

2. 在 Monitor 中创建 Arduino Nano 33 BLE 平台：

   ```none
   (monitor) mach create
   (machine-0) machine LoadPlatformDescription @platforms/boards/arduino_nano_33_ble.repl
   ```

   :::{note}
  `ArduinoLoader` 支持任何基于 Cortex-M CPU 的平台。
   :::

2. 在 Renode 中启动 USB/IP 服务器并将加载器连接到它：

    ```none
   (machine-0) emulation CreateUSBIPServer
   (machine-0) host.usb CreateArduinoLoader sysbus.cpu 0x10000 0 "arduinoLoader"
    ```

:::{note}
创建加载器时，您可以指定二进制加载地址（在 Arduino Nano 33 BLE 板的情况下为 [0x10000]{.title-ref}）、引导加载程序连接到 [host.usb]{.title-ref} 控制器的端口（如果您已经连接了其他设备，则应更改）和加载器的名称（在本例中为 `arduinoLoader`）。

下面的值是默认值，因此您可以跳过所有值，只留下：

```none
(machine-0) host.usb CreateArduinoLoader sysbus.cpu
```
:::

3. 一旦您的模拟完全设置完毕，并且您准备好接收和运行二进制文件，请启动加载器：

    ```none
   (machine-0) arduinoLoader WaitForBinary 120 true
   ```

这将使用 `usbip` 命令自动将 Renode 连接到主机（这使用 `sudo`，因此系统可能会要求您输入密码）并等待 120 秒，以便 Arduino IDE/CLI 上传二进制文件。

:::{note}
如果您不希望 Renode 使用 usbip 命令自动连接到您的主机，请不要传递最后一个参数 （`true`）。请记住，在这种情况下，您必须在上传二进制文件之前手动执行此作，否则 Arduino IDE/CLI 将无法检测到 Renode，并且该过程将失败。
:::

## 从 Arduino IDE 加载

启动 Arduino IDE 并选择您的速写本（在本例中，我们将使用 [TensorFlow Lite Hello World](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/lite/micro/examples/hello_world) 示例）。

![image](img/arduino_ide_examples.png)

:::{note}
默认情况下，Arduino Nano 33 BLE 板支持不附带 Arduino IDE。您需要通过安装“Arduino Mbed OS Nano Boards”包将其添加到 `Tools -> Board -> Boards Manager` 菜单中。
:::

从 `Tools -> Board -> Arduino Mbed OS Nano Boards` 菜单中选择 `Arduino Nano 33 BLE` 板。

按 `Verify` 按钮编译您的项目

![image](img/arduino_ide_verify.png)

一切编译正确后，在 Renode 中启动 `ArduinoLoader` （如[上一节](#ard-configuring-renode)所述）。

从 `Tools -> Port` 菜单中选择正确的 `/dev/ttyACMx` 设备作为端口。

![image](img/arduino_ide_port.png)

按 `Upload` 按钮上传二进制文件

![image](img/arduino_ide_upload.png)

## 从 Arduino CLI 加载

您不必使用 IDE 即可上传二进制文件 - 还有一个 Arduino CLI 工具，允许您直接从命令行编译和上传您的项目。.

:::{note}
确保您的系统中已安装 `arduino-cli`（默认情况下不附带 Arduino IDE）并在 PATH 中可用。有关详细信息，请参阅[项目的 github 页面](https://github.com/arduino/arduino-cli) 。
:::

首先，使用以下命令编译一个项目：
```sh
arduino-cli compile -b arduino:mbed:nano33ble hello_world.ino
```

确保一切编译正常，并在 Renode 中启动 `ArduinoLoader` （如[上一节](#ard-configuring-renode)所述）。

现在，使用以下命令上传二进制文件：

```sh
arduino-cli upload -b arduino:mbed:nano33ble --port /dev/ttyACM0 hello_world.ino
```

:::{note}
请确保选择正确的 `/dev/ttyACMx` 设备。
:::

## 开始模拟

收到二进制文件后，您将在 Monitor 中看到以下消息：

```none
(machine-0) arduinoLoader WaitForBinary 120 true
Binary of size 217088 bytes loaded at 0x10000
```

现在，您可以通过以下方式开始模拟：

```none
(machine-0) showAnalyzer sysbus.uart0
(machine-0) start

```

在 UART 上，您应该看到以下输出

```
123
123
128
128
128
128
135
135
135
135
136
136
136
136
141
141
141
141
142
142
142
142
148
148
148
148
153
153
153
153
155
165
165
165
168
168
168
172
172
172
172
173
173
173
173
178
178
178
178
178
178
178
178
178
178
178
178
181
181
181
181
184
184
```
