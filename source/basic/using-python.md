# 在 Renode 中使用 Python

Renode 提供了几种不同的 Python 集成，可用于从内部和外部扩展它。区分它们以避免混淆非常重要：

* 内置 IronPython 集成（这是在 Renode 中运行的 Python 2），这是本章的主要关注点
* 外部 [pyrenode](https://github.com/antmicro/pyrenode) 集成（这是一个使用 RPC 调用来调用 Renode / 执行机器人关键字的 Python 3 包装器）
* （新！外部 [pyrenode3](https://github.com/antmicro/pyrenode3) 集成（这是新的原生 CLR 绑定，本质上是围绕所有 Renode 的 Python 3 包装器，设置为在稳定时替换“旧”的 pyrenode）

本章目前重点介绍 Renode 中内置的 IronPython 集成和对 Python 的使用，因此您会在整个过程中注意到带有旧式 `print` 语句的 Python 2 语法。最终，内部 （Iron）Python 集成将迁移到 IronPython3 或移动到 `pyrenode3` 使用的相同机制。

同时，将添加 `pyrenode3` 的文档以补充本章。

得益于 IronPython 集成，您可以在运行时使用 Python 来执行各种作，例如对 UART 上的行做出反应、用户状态更改、外围设备访问或内存中出现的值。

Python 通过其流控制结构、循环等提供了一种真正的编程语言语法。

Python 还允许您访问特定于上下文的变量名称下可用的仿真组件。这些变量在相应的部分中进行了介绍。它们充当脚本和 Renode 其余部分之间的 API。

## 在 Monitor 中直接执行 Python

可以使用监控器中的 `python` 命令直接在 Renode 监控器中执行 Python 代码：

```
(monitor) python [ py ]
```

您还可以使用 `include` 命令加载现有的 `.py` 文件：

```
(monitor) include @path/to/file.py
```

## 从 Monitor 提供 Python 脚本

要从 Monitor 或 `.resc` 文件提供多行 Python 脚本，请使用以下表示法：

```none
(monitor) set my_script """
> print "Hello"
> print "This is a multiline Python script"
> if 0 > 1:
>    print "This will not be printed"
> """
(monitor) python $my_script
```

使用 `print` Python 命令将在 Monitor 上输出文本：

```none
(monitor) python "print 'Hello'"
Hello
```

## 在 Python 中创建 Monitor 命令

在 Monitor 中运行 `python` 命令会立即执行提供的代码。但是，可以使用这些 Python 脚本来定义可供以后使用的函数。

此类函数可以通过后续的 `python` 调用来执行，但如果您遵循特定的命名约定，您也可以将它们直接公开给 Monitor。

如果您定义的函数名称以 `mc_` 前缀开头，它将作为新的 Monitor 命令提供，去掉此前缀。

例如，请考虑以下 Python 定义：

```
def mc_sleep(time):
    sleep(float(time))
```

在 Renode 中执行时，此定义在 Monitor 中提供 `sleep` 函数：

```none
(monitor) sleep 5
```
[monitor.py](https://github.com/renode/renode/blob/master/scripts/monitor.py) 文件中提供了一小部分预定义的 Python 命令。

## 平台描述中的 Python 外围设备

您可以在平台描述的 `.repl` 文件中使用 Python。Renode 中包含的多个平台（如 [Zynq 7000](https://github.com/renode/renode/blob/master/platforms/cpus/zynq-7000.repl#L71-L124)）使用 Python 外设来实现简单的逻辑。Python 外围设备最常见的用途是模拟某些未完全实现但软件需要的块。

要创建 Python 外围设备，您需要指定几个变量：

```none
variableName: Python.PythonPeripheral @ sysbus 0x7000F410
    size: 0x4
    initable: false
    script: "request.value = 0x60000"
```

要定义 Python 外围设备，您可以指定以下变量：

```{list-table} Request variable fields
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - size
  - Size of the peripheral as a hexadecimal value
* - initable
  - If `true` the peripheral can be initialized and executes code from the ``isInit`` section
* - script
  - Python script you want to execute (exclusive with ``filename``)
* - filename
  - Path to a Python script file implementing the peripheral's logic (exclusive with ``script``)
```

对 Python 外围设备进行编程时，您可以访问描述当前事务的 `request` 变量。

Renode 中的变量 `request` 允许您访问以下字段：

```{list-table} Request variable fields
:header-rows: 1

* - Field name
  - Description
* - isInit
  - Is `true` during construction of the peripheral if it's marked as "initable"
* - isRead
  - Is `true` when the CPU is trying to read from a peripheral
* - isWrite
  - Is `true` when the CPU is trying to write to a peripheral
* - value
  - When `isWrite == true`, this is the value to be written, when `isRead == true` this is the return value
* - offset
  - Offset within the peripheral
* - type
  - Width of the access (8bit, 16bit, 32bit)
```

```{note}
当外围设备被标记为可能初始化并且当前正在初始化时，变量 `isInit` 为 `true`。它可用于初始化 peripheral 的内部状态。
```

主线 Renode 中可用的 Python 外围设备示例是 [repeater.py](https://github.com/renode/renode/blob/master/scripts/pydev/repeater.py)，它返回与软件写入的值相同的值：

```
if request.isInit:
    lastVal = 0
elif request.isRead:
    request.value = lastVal
elif request.isWrite:
    lastVal = request.value

self.NoisyLog("%s on REPEATER at 0x%x, value 0x%x" % (str(request.type), request.offset, request.value))
```

还有其他变量可以在 Python 外围设备中使用：

```{list-table} Other Python peripherals variables
:header-rows: 1
:widths: 25 75

* - Field name
  - Description
* - self
  - The peripheral itself
* - size
  - Size of the peripheral as a hexadecimal value
```

## Renode 中的 Python 钩子

Renode 中有多种类型的钩子，允许您在满足某些指定条件时执行代码，例如 Python 脚本。每个单独的 hook 都为您提供不同的功能，并且可以在单个项目中使用多种类型的 hook。Renode 中的钩子类型：

- {ref}`UART hooks,<uart-hooks>`
- {ref}`CPU hooks.<cpu-hooks>`
- {ref}`system bus hooks,<system-bus-hooks>`
- {ref}`watchpoint hooks,<watchpoint-hooks>`
- {ref}`packet interception hooks,<packet-interception-hooks>`
- {ref}`user state hooks,<user-state-hooks>`

(uart-hooks)=

### UART 钩子

UART 挂钩使您能够在 UART 上出现特定字符或子字符串时对事件的反应进行编码。

Renode 中的 UART 钩子可以访问以下变量：

```{list-table} UART hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - line
  - Current line matching the searched string
* - uart
  - UART object that produced the line
* - self
  - Alias for `uart`
```

您必须指定要在 UART 的输出中搜索的字符串，以及如果子字符串出现在 UART 上，将执行的 Python 脚本：

```
(machine) uart AddLineHook "searched value" "print 'Found the %s string' % line"
```

```{note}
您必须使用与您的使用案例匹配的 UART 名称。
```

(cpu-hooks)=

### CPU 钩子

CPU 挂钩使您能够对到达代码执行中的特定阶段的反应进行编码。钩子可以在已执行块的代码开头、结尾或应用程序中的任何指定点触发。CPU 挂钩还可以对代码中的中断做出反应，并在中断开始或结束时或 CPU 进入或退出 WFI（“等待中断”）状态时触发。

Renode 中的所有 CPU 钩子都可以访问以下变量：

```{list-table} CPU hooks common variables:
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - machine
  - Current machine, allows you to access other components and peripherals
* - self
  - CPU executing the current block
```

特定的钩子可以有额外的可用变量：

```{list-table} CPU block hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - pc
  - Program counter of the current block
* - size
  - Size of the block as the number of instructions
* - cpu
  - Alias for `self`
```

```{list-table} CPU interrupt hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - exceptionIndex
  - Exception index delivered to the CPU
```

```{list-table} CPU Wait-For-Interrupt hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - isInWfi
  - Whether the CPU is in WFI when executing the hook
```

要使用 CPU 钩子，你需要命名要附加钩子的 `cpu` 并指定方法。要创建在 interrupt 开始时触发的 hook：

```
(machine) cpu AddHookAtInterruptBegin "print 'Interrupt has started'"
```

要创建在 interrupt 结束时触发的 hook：

```
(machine) cpu AddHookAtInterruptEnd "print 'Interrupt has ended'"
```

要创建一个在 CPU 进入或退出 “Wait-For-Interrupt” 状态时触发的钩子：

```
(machine) cpu AddHookAtWfiStateChange "print 'Entered/Exited WFI'"
```
```{note}
WFI 状态可由各种 CPU 指令触发，具体取决于体系结构。例如，在 Arm 上，`WFI` 和 `WFE` 指令都可用于进入此状态。
```

要创建在已执行块的开头触发的钩子：

```
(machine) cpu SetHookAtBlockBegin "print 'Execution of a code block has started'"
```

要创建在已执行块的末尾触发的钩子：

```
(machine) cpu SetHookAtBlockEnd "print 'Execution of a code block has ended'"
```

您还可以为应用程序中的任何指定点创建一个钩子：

```
cpu AddHook 0x60000000 "print 'You have reached a hook'"
```

甚至将前一个命令与另一个命令组合在一起，以在品种处添加钩子：

```
cpu AddHook `sysbus GetSymbolAddress "main"` "print 'You have reached the main function'"
```

或者，您可以使用

```
cpu AddSymbolHook "main" "print 'You have reached the main function'"
```

设置一个 “跟随” 符号的钩子，并在它被加载到不同的地址时设置额外的钩子。这对于在运行时重新定位自身的软件非常有用。
(os-aware-modes)=

### 操作系统感知模式

操作系统感知模式利用特定于作系统的行为来增强仿真性能并增加生活质量的改进。
注：  
OS-aware mode   操作系统感知模式
OS-specific  特定于操作系统

```{list-table} OS-aware modes
:header-rows: 1
:widths: 10 45 45

* - Mode
  - U-Boot mode
  - Zephyr mode
* - Features
  - Skips busy loop in `_udelay`, automatically reloads symbols after U-Boot relocates.
  - Skips busy loop in `z_impl_k_busy_wait`
* - How to enable?
  - `cpu EnableUbootMode`
  - `cpu EnableZephyrMode`
```

(system-bus-hooks)=

### 系统总线钩子

System bus hooks 使您能够在访问外围设备进行读取或写入时对事件的反应进行编码。这个 hook 可以访问 peripherals，并且可以分配它只对特定的 peripheral 做出反应。

Renode 中的系统总线钩子可以访问以下变量：

```{list-table} System bus hooks value to write variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - self
  - The target peripheral
* - sysbus
  - System bus of the current machine
* - machine
  - Current machine, allows you to access other components and peripherals
* - value
  - Value that will be written to the given `offset`
* - offset
  - An offset, to which the `value` will be written
```

```{list-table} System bus hooks value to read variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - self
  - The target peripheral
* - sysbus
  - System bus of the current machine
* - machine
  - Current machine, allows you to access other components and peripherals
* - value
  - Value read from the `offset`
* - offset
  - An offset, from which the `value` will be read
```

要创建一个系统总线钩子，该钩子在访问特定外设后执行 Python 脚本以读取：

```
(machine) sysbus SetHookAfterPeripheralRead peripheral "print '%s peripheral has been accessed to read'"
```

要创建一个系统总线钩子，在访问特定外设之前执行 Python 脚本以写入：

```
(machine) sysbus SetHookBeforePeripheralWrite peripheral "print '%s peripheral has been accessed to write'"
```

(watchpoint-hooks)=

### 监视点钩子

监视点钩子使您能够在特定内存地址中出现特定值时对事件的反应进行编码。

Renode 中的观察点钩子可以访问以下变量：

```{list-table} Watchpoint hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - cpu
  - CPU issuing the memory access
* - address
  - Address of the access triggering the hook
* - width
  - Width of the access in bits: Byte = 1, Word = 2, DoubleWord = 4, QuadWord = 8
* - value
  - Value written to the address or read from it.
* - self
  - Sysbus of the current machine
```

要创建执行 Python 脚本的观察点钩子，当`地址` 0x70001000 处出现宽度为 32 位`的``读取`访问 （DoubleWord） 时，运行：

```
(machine) sysbus AddWatchpointHook 0x70001000 DoubleWord Read "print '32 bit value appeared at the address 0x70001000'"
```

```{note}
您可以在`宽度`中使用单词 （DoubleWord） 或数字 （4） 表示法，如上表所示。
```

(packet-interception-hooks)=

### 数据包拦截钩子

数据包拦截钩子使您能够在数据包出现在无线电介质上时对事件的反应进行编码。

Renode 中的数据包拦截钩子可以访问以下变量：

```{list-table} Packet interception hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - packet
  - Byte array with the contents of a packet
* - self
  - Radio that is about to receive the packet
* - machine
  - Your current machine
```

```{note}
请记住，钩子是为收件人执行的，而不是为发件人执行的。
```

要有一个正常运行的 Packet interception 钩子，你的机器需要一个无线介质。要创建数据包拦截钩子，请运行：

```
(machine) wireless SetPacketHookFromScript sysbus.radio "if packet[5] == 0x4: print('I\'m interested in packets with 0x4 as their sixth byte')"
```

您还可以从文件中执行脚本：

```none
(machine) wireless SetPacketHookFromFile sysbus.radio @path/to/file.py
```

(user-state-hooks)=

### 用户状态钩子

用户状态钩子使您能够在计算机的 “UserState” 字符串更改时对事件的反应进行编码。

Renode 中的用户状态钩子可以访问以下变量：

```{list-table} User states hooks variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - state
  - A new string set as user state
* - self
  - A current machine
```

要创建在设置特定`状态`时执行 Python 脚本的用户状态钩子：

```
(machine) machine AddUserStateHook "state" "print 'User state has changed to: %s state '"
```

## 支持 Python 的类

前面部分中描述的钩子使用特殊方法来设置 Python 脚本的执行环境，但 IronPython 还允许您将 Python 函数附加到 C# 事件。

例如，让我们创建一个普通的以太网嗅探器：

```none
(machine-0) emulation CreateSwitch "switch"
(machine-0) python "externals.switch.FrameProcessed += lambda switch, sender, data: sender.LogDebug(str(data))"
```

有关设置网络的深入说明，请参阅 {doc}`设置有线网络 <../networking/wired>` 一章。

### 虚拟 I2C 从机

`Mocks.DummyI2CSlave` 外设提供了一组方法和事件，允许您实现简单的 I2C 外设。

```{list-table} DummyI2CSlave events
:header-rows: 1
:widths: 10 45 45

* - Event name
  - Description
  - Argument
* - DataReceived
  - Invoked when the peripheral is written to
  - byte array containing the received data
* - ReadRequested
  - Invoked when a read is requested from the peripheral
  - The number of bytes requested to be read
* - TransmissionFinished
  - Invoked when the controller finishes transmission
  - n/a
```

`DataReceived` 事件不需要提供任何数据，但您可以使用下面列出的方法将响应字节排入队列。默认情况下，当内部缓冲区为空时，外设返回零，并在缓冲区中字节数不足时用零填充任何排队的数据。

```{list-table} DummyI2CSlave methods
:header-rows: 1
:widths: 10 45 45

* - Method name
  - Description
  - Argument
* - Reset
  - Clears the internal buffer, but doesn't remove added events
  - n/a
* - EnqueueResponseByte
  - Enqueues a single byte to the internal buffer
  - A byte
* - EnqueueResponseBytes
  - Enqueues a collection of bytes
  - A byte enumerable, e.g. an array of bytes
```

有了这些，我们可以将虚拟 I2C 外设更改为简单的 I2C 回声外设：

```
dummy = monitor.Machine["sysbus.i2c.dummy"]
dummy.DataReceived += lambda data: dummy.EnqueueResponseBytes(data)
```

### 虚拟控制台

`VirtualConsole` 旨在成为 Python 脚本的帮助器，因为它受相同的基础设施支持，例如，您可以在 Robot Framework 测试中使用它和 UART 关键字，并创建分析器以与控制台交互。它为下面列出的接收数据、方法和事件提供内部缓冲区，允许从 Python 脚本进行控制。

可以使用 Monitor 命令创建控制台的实例：

```none
(machine-0) machine CreateVirtualConsole "vconsole"
(machine-0) showAnalyzer vconsole
```

或在 `repl` 文件中：

```
vserial: UART.VirtualConsole @ sysbus
```

```{list-table} VirtualConsole methods
:header-rows: 1
:widths: 10 30 30 30

* - Method name
  - Description
  - Returns
  - Arguments
* - Clear
  - Removes all data from the buffer
  - n/a
  - n/a
* - IsEmpty
  - Tests for data in the buffer
  - True if no data is in the buffer, false otherwise
  - n/a
* - Contains
  - Checks whether a byte is in the buffer
  - True if provided byte is in the buffer, false otherwise
  - The byte to check against
* - WriteChar
  - Writes a byte to be received
  - n/a
  - The byte to receive
* - ReadBuffer
  - Retrieves data from the buffer and removes it
  - The retrieved data
  - optional limit to number of bytes returned, defaults to 1
* - GetBuffer
  - Retrieves the buffer without consuming the data
  - A copy of the buffer
  - n/a
* - WriteBufferToMemory
  - Writes some or all of the bytes from the buffer to the memory at a specified address
  - Number of bytes written
  - Address to which the data should be written, limit of the number of bytes to write, optional CPU context (defaults to global)
* - DisplayChar
  - Transmits a byte
  - n/a
  - The byte to be transmitted
```

使用上面列出的方法和其他一些访问钩子，您可以实现基于轮询的外围设备，例如瑞[萨电子的 Segger RTT](https://github.com/renode/renode/blob/master/scripts/single-node/renesas-segger-rtt.py)，它用于带有交互式控制台的演示（例如 [scripts/single-node/ek-ra2e1.resc](https://github.com/renode/renode/blob/master/scripts/single-node/ek-ra2e1.resc)）和使用 UART 关键字（例如 [tests/platforms/EK-RA2E1.robot](https://github.com/renode/renode/blob/master/tests/platforms/EK-RA2E1.robot)）的机器人框架测试。

```{list-table} VirtualConsole events
:header-rows: 1
:widths: 10 45 45

* - Event name
  - Description
  - Argument
* - CharReceived
  - Invoked when the peripheral transmits a byte
  - The byte transmitted
* - CharWritten
  - Invoked when a byte is pushed to the receive buffer
  - The byte received by the peripheral
```

`VirtualConsole` 还具有 `Echo` 属性，用于启用或禁用接收字符的显示。


(python-riscv)=

## RISC-V 扩展

Renode 对 RISC-V 架构提供了广泛的支持，并允许您使用 Python 编写扩展代码。您可以编写 Control/Status Register 的逻辑和自定义指令，然后从文件或字符串安装它们。这些指令可以是 16 位、32 位或 64 位。

RISC-V 自定义指令的 Python 实现可以访问以下变量：

```{list-table} RISC-V custom instructions variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - cpu
  - CPU that executes the instruction
* - machine
  - Current machine
* - state
  - Current user state of a CPU
* - instruction
  - Current opcode
```

```{note}
Variable `state` 为您提供额外的 CPU 状态，将用户定义的字符串映射到任意对象。当您想在不同指令之间共享状态时，它很有用。
```

可以使用以下方法安装字符串中的自定义指令：

```
(machine) sysbus.cpu InstallCustomInstructionHandlerFromString "10110011100011110000111110000010" "cpu.DebugLog('custom instruction executed!')"
```

如果要从文件安装自定义说明，请使用：

```
(machine) sysbus.cpu InstallCustomInstructionHandlerFromFile "10110011100011110000111110000010" "path/to/file.py"
```

Renode 中的 RISC-V 自定义指令扩展可以访问以下变量：

```{list-table} RISC-V custom CSR variables
:header-rows: 1
:widths: 25 75

* - Variable name
  - Description
* - cpu
  - CPU that accesses the custom CSR
* - machine
  - Current machine
* - request
  - Structure of this variable is described below
```

自定义指令可以访问 `request` 对象，这类似于 bus 访问。它可以访问各种属性：

```{list-table} The request structure
:header-rows: 1
:widths: 25 75

* - Property name
  - Description
* - CSR number
  - Number of the register as a hexadecimal value
* - Value
  - Read of written value
* - Type
  - Type of request: READ, WRITE, INIT
* - isInit
  - Is `true` during construction of the CPU if it's marked as "initable"
* - isRead
  - Is `true` when the CPU is trying to read from the register
* - isWrite
  - Is `true` when the CPU is trying to write to the register
```

要从字符串安装自定义 Control/Status Register，请使用：

```
(machine) sysbus.cpu RegisterCSRHandlerFromString 0xf0d "print 'CSR has been accessed'"
```

要从文件提供自定义 CSR，请改用：

```
(machine) sysbus.cpu RegisterCSRHandlerFromString 0xf0d "path/to/file.py"
```

```{note}
您应该根据需要调整 CSR 编号，`0xf0d` 本例中所示。
```

自定义 CSR 和自定义指令的示例可以在 [Renode 的自定义指令测试](https://github.com/renode/renode/blob/master/tests/unit-tests/riscv-custom-instructions.robot#L178)中找到。
