# 协同仿真你的验证模型

使用 Verilator 在 Renode 中创建 HDL 模型并协同仿真是一个高级主题，因此请务必事先访问[使用 HDL 仿真器进行协同仿真](https://renode.readthedocs.io/en/latest/advanced/co-simulating-with-an-hdl-simulator.html)一章。如果您想使用示例验证模型作为构建自己的模型的灵感，请访问 [Antmicro 的 GitHub](https://github.com/antmicro/renode-verilator-integration)。


```{note}
您可以在 Linux 和 Windows 作系统上运行本教程。
```

## 创建已验证的外围设备

要制作自己的已验证外设，在已验证模型的主 cpp 文件中，您需要包含适用于您正在连接的总线的 C++ 标头。

您还需要指定要与 Renode 集成的外部接口的类型 - 例如，UART 的 rx/tx 信号。

```cpp
// uart.h and axilite.h can be found in Renode's CoSimulationPlugin
#include "src/peripherals/uart.h"
#include "src/buses/axilite.h"
```

接下来，您需要定义一个函数，该函数将调用模型的 eval 函数，并将其作为 Integration Library 结构的回调以及总线和外设信号提供。

```cpp
void eval() {
   top->eval();
   #if VM_TRACE
   main_time++;
   tfp->dump(main_time);
   tfp->flush(); // optional
   #endif
}

void Init() {
AxiLite* bus = new AxiLite();

//==========================================
// Init bus signals
//==========================================
bus->clk = &top->clk;
bus->rst = &top->rst;
bus->awaddr = (unsigned long *)&top->awaddr;
bus->awvalid = &top->awvalid;
bus->awready = &top->awready;
bus->wdata = (unsigned long *)&top->wdata;
bus->wstrb = &top->wstrb;
bus->wvalid = &top->wvalid;
bus->wready = &top->wready;
bus->bresp = &top->bresp;
bus->bvalid = &top->bvalid;
bus->bready = &top->bready;
bus->araddr = (unsigned long *)&top->araddr;
bus->arvalid = &top->arvalid;
bus->arready = &top->arready;
bus->rdata = (unsigned long *)&top->rdata;
bus->rresp = &top->rresp;
bus->rvalid = &top->rvalid;
bus->rready = &top->rready;

//==========================================
// Init eval function
//==========================================
bus->evaluateModel = &eval;

//==========================================
// Init peripheral
//==========================================
uart = new UART(bus, &top->txd, &top->rxd,
   prescaler);
}
```

作为最后一步的一部分，在 `main` 函数中，您必须使用两个端口号调用 `sime，Renode` 在其上等待通信。

当 Renode 启动已验证的外围设备时，这些端口号将作为命令行参数传递。

```cpp
Init();
uart->simulate(atoi(argv[1]), atoi(argv[2]));
```

(building-verilated-peripheral)=

## 构建已验证的外围设备

有一些先决条件：

* [renode-verilator-integration 存储库](https://github.com/antmicro/renode-verilator-integration)的本地副本，
* [Renode 存储库](https://github.com/renode/renode)的本地副本，因为它的 [IntegrationLibrary](https://github.com/renode/renode/tree/master/src/Plugins/CoSimulationPlugin/IntegrationLibrary)，
* 验证器 >= v4.024

`$RVI_PATH`、`$RENODE_PATH` 和 `$VERILATOR_PATH` 将分别用于引用这三个先决条件的路径。如果正确安装了 Verilator 的路径，则不需要它，即可执行文件在 `PATH` 中可用，并且可以根据 [CMake 的 find_package 搜索程序](https://cmake.org/cmake/help/latest/command/find_package.html#search-procedure)找到 `verilator-config.cmake`。

包含 Verilog 和 C/C++ 源文件的目录的路径将称为 `$SRC_PATH。` 最好将该目录直接放在 `renode-verilator-integration` 存储库根目录中，以确保包含所有外围设备通用的 CMake 逻辑的 [configure-and-verilate.cmake](https://github.com/antmicro/renode-verilator-integration/blob/master/cmake/configure-and-verilate.cmake) 文件的默认路径是正确的。

```{note}
To run shell commands without any modifications, set all ``*_PATH`` shell variables before running the commands.
```

### 准备外设目录

首先，将所有 Verilog 和 C/C++ 源文件放在 `$SRC_PATH` 中。然后，将 `$RVI_PATH/cmake/CMakeLists.txt.template` as `CMakeLists.txt` 复制到 `$SRC_PATH` 目录：

```sh
# Execute from a directory containing peripheral's source files
mkdir "$SRC_PATH"
cp *.v *.c *.cpp "$SRC_PATH"
cp "$RVI_PATH/cmake/CMakeLists.txt.template" "$SRC_PATH/CMakeLists.txt"
```

项目的 `$SRC_PATH/CMakeLists.txt` 文件需要稍作调整才能与特定的外围设备很好地配合（只需要前两个）：

* 将 `<PROJECT_NAME>` 替换为所选名称，
* 将 `<MODULE_FILES>` 和 `<C_SRC_FILES>` 替换为相对于 `$SRC_PATH` 的外围文件的路径，
* 通过删除 `#` 并将 `<ARGS>` 替换为设置 `COMP_ARGS`、`LINK_ARGS` 和 `VERI_ARGS` 变量的行中的实际参数，添加在构建的某个阶段始终使用的选定参数。
* 如果外围设备的源目录未直接放置在 `<RVI_PATH>` 中，则在 `include` 命令中调整 `configure-and-verilate.cmake` 的路径。

```{note}
使用空格分隔多个文件或参数，替换 `<*_FILES>` 和 `<ARGS>` 占位符。
```

(build-commands)=

###  构建命令

准备好 CMake 源目录后，现在可以构建经过验证的外围设备了。使用 CMake 时，最好将构建文件保存在单独的构建目录中：

```sh
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DUSER_RENODE_DIR="$RENODE_PATH" ${VERILATOR_PATH:+"-DUSER_VERILATOR_DIR=$VERILATOR_PATH"} "$SRC_PATH"
make
```

如果构建成功， `则 libVtop` 是构建的已验证外设。

```{note}
可以使用 `RENODE_ROOT` 和 `VERILATOR_ROOT` 环境变量来代替 CMake `USER_RENODE_DIR` 和 `USER_VERILATOR_DIR` 变量（分别）。如果为 Renode 或 Verilator 同时指定了环境变量和 CMake 变量，则 CMake 变量具有更高的优先级。
```

```{note}
使用 `make -j $（nproc）` （ `make -j $(sysctl -n hw.logicalcpu)` 在 macOS 上）通过创建与可用 CPU 内核数量一样多的作业来优化构建速度。
```

### Linux 特定的构建信息

在 Linux 上，建议使用 OpenLibm 来启用使用较旧的 GNU libc 版本运行经过验证的外围设备。这是因为最近在 GNU libm（GNU libc 的一部分）中更新了一些常见的数学函数。如果外围设备与它们相连，您将需要这些更新的函数来运行外围设备。

将附加 `-DLIBOPENLIBM=$RVI_PATH/lib/libopenlibm-Linux-x86_64.a` 参数传递给 `cmake` 命令以使用 Renode 当前也在使用的 OpenLibm 库。

### Windows 特定的构建信息

上述步骤经过测试，经过一些细微的更改，可在 Windows 上使用 [Cygwin](https://www.cygwin.com/) 和 [MSYS2](https://www.msys2.org/) 成功构建外围设备。MSYS2 有一个支持良好的 [mingw-w64-verilator](https://packages.msys2.org/base/mingw-w64-verilator) 包，因此不需要构建 Verilator。

```{note}
在 Windows 上，使用绝对路径更为重要。这些可以是 Cygwin/MSYS2 绝对路径，即 `/home/<...>`.支持 Windows 和 Unix 路径样式。
```

[Build commands](build-commands) 部分中的 CMake 命令需要添加以下参数才能在使用 Cygwin/MSYS2 和 MinGW 的 Windows 上工作：

* `-g “MinGW Makefiles”` – 为 `MinGW make` 生成 `Makefile`，
* `-DCMAKE_SH=CMAKE_SH-NOTFOUND` – 尽管在 `PATH` 中有 `sh.exe`，但 CMake 在 Windows 上运行是必需的。

此外，在最常见的工具链设置中，应该使用 `mingw32-make` 命令而不是 `make`，即使两者都可用。

因此，在 Windows 上，可以使用以下方法构建经过验证的外围设备：

```sh
cmake -G "MinGW Makefiles" -DCMAKE_SH=CMAKE_SH-NOTFOUND -DCMAKE_BUILD_TYPE=Release -DUSER_RENODE_DIR="$RENODE_PATH" ${VERILATOR_PATH:+"-DUSER_VERILATOR_DIR=$VERILATOR_PATH"} "$SRC_PATH"
mingw32-make
```

## 运行已验证的外围设备

构建完经过验证的可执行文件后，是时候将其附加到 [Renode 机器](working-with-machines)上了，因此它实际上被用作外围设备。

首先，必须将专用外围设备添加到将用于配置机器的 [Renode 平台描述 （.repl） 文件中](../basic/describing_platforms.md) 。对于名为 `myCoSimulatedPeripheral` 的协同仿真 UART 外设，请将以下行添加到您的 `.repl` 文件中：

```none
myCoSimulatedPeripheral: CoSimulated.CoSimulatedUART @ sysbus <0x70000000, +0x100>
   frequency: 100000000
```

```{note}
对于除 UART 以外的协同仿真外围设备，应使用 `CoSimulated.CoSimulatedPeripheral` type 而不是 `CoSimulated.CoSimulatedUART`。
```

```{note}
该示例使用基于库调用集成的通信。要使用基于套接字的集成，您应该将 `address` 参数添加到 REPL 文件中，并改用 `Vtop` 二进制文件。
```

在 Renode 中，直接在 [Renode 监视器](monitor)中使用命令或使用适当的 [Renode 脚本 （.resc） 文件](scripts)加载此类平台描述后，需要附加经过验证的可执行文件。假设 `libVtop` 二进制文件位于 Renode 根目录中，则可以将其附加为：

```none
(machine-0) myCoSimulatedPeripheral SimulationFilePath @libVtop
```

否则，可以使用绝对路径或相对于 Renode 根目录的路径来代替 `libVtop`。

```{note}
路径必须以 `@` 符号开头或用双引号 `“`.
```

## 使用 Renode Verilator 示例外围设备

要构建和使用[来自 renode-verilator-integration 存储库](https://github.com/antmicro/renode-verilator-integration)的经过验证的外设模型之一，只需将自定义 HDL 替换为其中一个示例，然后[按照上述相同的步骤进行作](building-verilated-peripheral) 。

```{note}
您可以跳过**准备外围目录** ，因为它们的目录已经准备好了。
```

## 模拟性能

您可以从两个方面控制已验证外设的性能：相对于主 CPU 的虚拟时间性能和执行的实时性能。

如上面的示例所示，您需要定义每个 peripheral 的 clock frequency 。`frequency` 参数需要一个以 `Hz` 为单位的值。

该值用于驱动 verilated design 的 clock 信号，并在 virtual time domain 中定义。这意味着 CPU 执行的每条指令，配置了特定的 `PerformanceInMips` 值，导致 design 中的时钟滴答声数量恒定。有关更多详细信息，请参阅 [Time Framework 一章](../advanced/time_framework.md) 。

由于在 CPU 执行每条指令后触发 clock signals 是不切实际的，因此您可以缓冲这些事件并在达到特定阈值时发送它们。这可以使用可选的 `limitBuffer` 构造函数参数轻松配置：

```none
myCoSimulatedPeripheral: CoSimulated.CoSimulatedUART @ sysbus <0x70000000, +0x100>
   frequency: 100000000
   limitBuffer: 10000
```

`limitBuffer` 的默认值为 1000000，这意味着 Renode 在累积 100 万个时钟周期之前不会触发时钟信号。

## Verilator 跟踪

您还可以通过在与已验证模型相对应的 `CMakeList.txt` 中设置 `--trace` 或 `--trace-fst` Verilator 选项来启用信号跟踪转储。

请按照以下说明作，以确保正确初始化和使用已验证的 dump 对象。

首先，包括以下定义。这些将启用跟踪，并允许您使用上述 Verilator 选项在 `fst` 和 `vcd` 文件类型之间切换。

```cpp
#if VM_TRACE_VCD
# include <verilated_vcd_c.h>
# define VERILATED_DUMP VerilatedVcdC
# define DEF_TRACE_FILEPATH "simx.vcd"
#elif VM_TRACE_FST
# include <verilated_fst_c.h>
# define VERILATED_DUMP VerilatedFstC
# define DEF_TRACE_FILEPATH "simx.fst"
#endif
```

接下来，声明已验证的 dump 对象，并在每次模型评估中收集信号数据。

```cpp
#if VM_TRACE
   VERILATED_DUMP *tfp;
#endif
vluint64_t main_time = 0;

void eval() {
top->eval();
#if VM_TRACE
   main_time++;
   tfp->dump(main_time);
   tfp->flush();
#endif
}
```

最后，初始化已验证的转储并运行跟踪。如果您想从套接字上运行的协同仿真中获取经过验证的 dump，请将此部分包含在 `main（）` 函数中;否则，请将其放在 `Init（）` 函数中。

```cpp
#if VM_TRACE
   Verilated::traceEverOn(true);
   tfp = new VERILATED_DUMP;
   top->trace(tfp, 99);
   tfp->open(DEF_TRACE_FILEPATH);
#endif
```

根据指定的选项，结果跟踪将被写入 vcd 或 fst 文件，并且可以在 [GTKWave 查看器](http://gtkwave.sourceforge.net/)等中查看。

![gtkwave-trace](img/gtkwave-trace.png)

## 使用经过验证的 UART 的 Core-v-mcu “Hello World” 示例

### 准备二进制文件

有关如何设置 SDK 的说明，请参阅 [pulp-builder 存储库](https://github.com/pulp-platform/pulp-builder/tree/arnold) 。配置完成后，设置 `PULPRT_HOME` 环境变量，并指定 `pulp-rules` 目录的路径。

您还需要编辑 SDK 源代码。要将字符写入 `txd` UART 寄存器，请在 [io.c 文件](https://github.com/pulp-platform/pulp-rt/blob/eaf528a1926b9e12f94e4aa66e3f5768263db678/libs/io/io.c)中添加 `__rt_putc_uart` 函数：

```cpp
*((volatile uint32_t*)(0x50000004)) = c;
```

“Hello World” 代码源代码可以在 [pulp-rt-examples](https://github.com/pulp-platform/pulp-rt-examples/tree/master/hello) 中找到。要编译，请运行：

```sh
make all io=uart
```

应在 `pulp-rt-examples/hello/build/arnold/test` 目录中创建生成的二进制文件。

### 在 Renode 模拟中运行

要在 core-v-mcu hello world 示例中启用经过验证的 UART 外设，您需要在 [core-v-mcu.repl](https://github.com/renode/renode/blob/master/platforms/cpus/core-v-mcu.repl) 中注册 `CoSimulatedUART`，例如：

```none
verilated_uart: CoSimulated.CoSimulatedUART @ sysbus <0x50000000, +0x100>
   frequency: 100000000
```

然后，您必须在 Renode 监视器类型中向 Renode 模拟提供二进制文件：

```none
(monitor) using sysbus
(monitor) mach create
(machine-0) machine LoadPlatformDescription @platforms/cpus/core-v-mcu.repl
```

将二进制文件附加到模拟：

```none
(machine-0) sysbus LoadELF @path_to_your_binary
```

您可以使用经过验证的 UART 模型：

```none
(machine-0) verilated_uart SimulationFilePath @path_to_verilated_uart_model
```

或者您可以使用我们提供的预构建版本：

```none
(machine-0) $uart?=@https://dl.antmicro.com/projects/renode/verilator--uartlite_trace_off-s_252704-c703fe4dec057a9cbc391a0a750fe9f5777d8a74
(machine-0) verilated_uart SimulationFilePath $uart
```

要启用 UART 分析器窗口并开始仿真，请键入：

```none
(machine-0) showAnalyzer verilated_uart
(machine-0) s
```
