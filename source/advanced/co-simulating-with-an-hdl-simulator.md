# 与 HDL 仿真器协同仿真

Renode 包括两种用于 HDL 协同仿真的集成方法，允许您将 HDL 外设与中断和外部接口（如 UART Rx/Tx 线路）连接起来：

* 基于 DPI 的 SystemVerilog 方法（适用于支持 DPI 的模拟器，如 Verilator、Questa 或 VCS）
* 自定义仅限 [Verilator](https://www.veripool.org/verilator/) 的 C++ 方法

## 添加协同仿真块

集成层的 Renode 端是一个名为 CoSimulationPlugin 的插件，它由启动通信的 [C# 类](https://github.com/renode/renode/tree/master/src/Plugins/CoSimulationPlugin)组成。

通常，`CoSimulatedPeripheral` 类的实例对应于 HDL 模型。要将协同模拟块添加到您的平台，请在 [REPL 文件](https://renode.readthedocs.io/en/latest/basic/describing_platforms.html#describing-platforms)中使用以下代码段：

```none
block: CoSimulated.CoSimulatedPeripheral @ sysbus <0x20000000, +0x100000>
```

## 基于 DPI 的集成方法

在这种情况下，Renode 和 HDL 仿真器之间的消息使用许多仿真器（包括 Verilator、VCS 或 Questa）支持的标准 DPI 接口通过 TCP 套接字传输。

由于 DPI 层依赖于 TCP 套接字连接，因此您还需要为外围设备指定 `address` 参数：

```none
block: CoSimulated.CoSimulatedPeripheral @ sysbus <0x20000000, +0x100000>
    address: "127.0.0.1"
```

HDL 侧使用 [SystemVerilog 接口](https://github.com/renode/renode/tree/master/src/Plugins/CoSimulationPlugin/IntegrationLibrary/hdl) ，该接口使用特定于您正在仿真的总线的信号直接连接到您的 HDL 仿真。

对于协同仿真，您可以通过 `SimulationPath` 属性传递外部模拟器位置，以便由 Renode 自动生成它。在这种情况下，您可以使用 `SimulationContext` 属性传递自定义命令行参数，例如对于 Verilator：

```none
block: CoSimulated.CoSimulatedPeripheral @ sysbus <0x20000000, +0x100000>
    address: "127.0.0.1"
    SimulationPath: build/verilated
    SimulationContext: "--custom_features_enabled +RENODE_RECEIVER_PORT={0} +RENODE_SENDER_PORT={1} +RENODE_ADDRESS={2}"
```

否则，HDL 仿真器必须在连接到它之前运行。

### 支持的总线

* AHB
* APB3
* AXI4
* AXI4-Lite

### 示例

您可以在 [renode-dpi-examples 存储库](https://github.com/antmicro/renode-dpi-examples)中找到每个受支持总线的示例以及构建它们的说明。所有示例均已使用 [Verilator](https://www.veripool.org/verilator/) 和 [Questa](https://www.intel.com/content/www/us/en/software/programmable/quartus-prime/questa-edition.html) 进行了测试。

## 自定义直接积分方法 （仅限 Verilator）

使用 Verilator 时，您还可以将 HDL 仿真编译为动态库，并在运行时将其与 Renode 链接，而不是使用 TCP 套接字。Renode 可以自行生成或链接模拟。

```{note}
您不应为作为动态库连接的外围设备指定 `address` 参数
```

要将 HDL 仿真与 Renode 连接，您需要将 HDL 设计的信号与 [C++ 接口](https://github.com/renode/renode/tree/master/src/Plugins/CoSimulationPlugin/IntegrationLibrary/src)连接。

### 支持的总线

* APB3
* AXI4
* AXI4-Lite
* Wishbone
* CFU Custom Function Unit interface

### 示例

您可以在 [renode-verilator-integration 存储库](https://github.com/antmicro/renode-verilator-integration)中找到许多经过验证的外设模型示例，例如 [CFU](https://github.com/antmicro/renode-verilator-integration/tree/master/samples/cfu_basic) 、 [UART](https://github.com/antmicro/renode-verilator-integration/tree/master/samples/uartlite) 、 [RAM](https://github.com/antmicro/renode-verilator-integration/tree/master/samples/ram) 甚至 [CPU](https://github.com/antmicro/renode-verilator-integration/tree/master/samples/cpu_ibex) 。有关如何从存储库或你自己的存储库构建模型的详细说明，请参阅[协同仿真你的验证模型](https://renode-docs-chinese.readthedocs.io/en/latest/tutorials/co-simulating-custom-hdl.html) 。

## 连接到已验证的模型

使用 Verilator 时，您的 HDL 模型首先被转换为 C++（或“verilated”），然后进行编译。

Renode 附带了几个 `.resc` 文件，这些文件使用以这种方式预编译的 HDL 模型（我们有时也以编译形式称它们为“verilated”，以解释它们的来源），例如[经过验证的 UART 模型](https://github.com/renode/renode/blob/master/scripts/single-node/riscv_verilated_liteuart.resc)或[经过验证的 Ibex](https://github.com/renode/renode/blob/master/scripts/single-node/verilated_ibex.resc)。

用于加载已验证模型库的 Monitor 命令取决于您的主机作系统。在 Linux 上，要将预编译的 FastVDMA HDL 模型添加到名为 `dma` 的外围设备，您可以运行：

```none
dma SimulationFilePathLinux @https://dl.antmicro.com/projects/renode/zynq-fastvdma_libVfastvdma-Linux-x86_64-1246779523.so-s_2057616-93e755f7d67bc4d5ca33cce6c88bbe8ea8b3bd31
```

如果要为不同的作系统提供不同的负载，则可以在同一脚本中使用 `SimulationFilePathLinux`、`SimulationFilePathMacOS` 和 `SimulationFilePathWindows`，从而使脚本成为多平台脚本。Renode 将选择与您的主机作系统匹配的系统。

如果您只对单个作系统感兴趣，只需使用 `SimulationFilePath，Renode` 会将其解释为与您当前作系统匹配的模型。

按照惯例，我们托管两种类型的预编译模型：
* 前缀为 `V` 的模型使用基于套接字的集成
* 前缀为 `libV` 的模型通过库调用与 Renode 通信

大多数内置脚本使用带有 `libV` 前缀的二进制文件。
