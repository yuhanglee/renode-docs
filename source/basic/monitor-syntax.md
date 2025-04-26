# 监视器和脚本语法

Monitor 是 Renode 的命令行界面 （CLI）。它允许您与 Renode 通信并使用各种内置功能控制仿真。它们允许您访问外围设备、计算机和外部连接器等仿真对象。

```{note}
如果您想了解有关 Monitor 及其基本用法的更多信息，请访问  {doc}`../introduction/using` 章节。
```

通常，Renode 中的命令是逐行执行的，参数需要用空格分隔。您可以使用 Python 进一步扩展可用函数的范围（有关示例，请参阅 [monitor.py](https://github.com/renode/renode/blob/master/scripts/monitor.py) 和有关如何使用 Python 扩展 Renode 的说明，请参阅  {doc}`using-python`）。

## 使用内置命令

Renode 带有数十个内置命令。您可以使用 Monitor 中的 `help` 命令访问命令的完整列表及其描述。要获取有关某个命令的更多详细信息，请使用 `help <command>`，例如：

```
help start
```

## 访问仿真对象

要访问计算机的仿真对象（例如外围设备）可用的命令，您可以键入对象的名称，然后单击 Tab 键两次以激活 Tab 键完成。

仿真对象位于层次结构中，`sysbus` 是所有外围设备的机器根。如果要访问机器的 UART 外围设备 `uart`，则需要使用 `sysbus.uart`。你可以使用 `using` 命令来设置一个默认前缀，例如`使用 sysbus` 允许你直接引用 UART：

```
uart
```

您会发现 `using sysbus` 命令用于与 Renode 捆绑的大多数脚本中。

将仿真对象的名称作为命令传递将返回此对象的可用方法、属性和值字段的完整列表。在此列表中，您可以找到有关该方法是否返回值、它接受哪些数据类型以及如何使用它的信息：

```none
(machine-0) uart
The following methods are available:
 - Void AddLineHook (String contains, String pythonScript)
 - Void CloseFileBackend (String path)
 - Void CreateFileBackend (String path, Boolean immediateFlush = False)
 - Void DebugLog (String message)
 - String DumpHistoryBuffer (Int32 limit = 0)
 - Endianess GetEndianness (Endianess? defaultEndianness = null)
 - IEnumerable<tuple<string,< span="">IGPIO>> GetGPIOs ()</tuple<string,<>
...
Usage:
 sysbus.uart MethodName param1 param2 ...
The following properties are available:
 - UInt32 BaudRate
     available for 'get'
 - Parity ParityBit
     available for 'get'
 - Int64 Size
     available for 'get'
 - Bits StopBits
     available for 'get'
Usage:
 - get: sysbus.uart PropertyName
 - set: sysbus.uart PropertyName Value
```

要检查对象可用的特定方法的参数，只需提供对象的名称和方法：

```none
(machine-0) uart WriteWordUsingByte
The following methods are available:
 - Void WriteWordUsingByte (Int64 address, UInt16 value)
Usage:
 sysbus.uart MethodName param1 param2 ...
```

请记住，如果一个方法不需要任何参数，则提供其名称将调用它：

```none
(machine-0) machine GetTimeSourceInfo
Elapsed Virtual Time: 00:00:00.000000
Elapsed Host Time: 00:00:00.000000
Current load: NaN
...
```

许多命令要求您在使用时在命令后指定一个参数：

```none
using sysbus
```

## Accessing attributes of an object

Renode 中的对象可以根据其类型访问不同的方法、属性、字段和索引器。访问外围设备的参数需要这些方法，要成功执行此作，您需要遵循以下语法：

```none
(machine) registrationPoint.peripheral MethodName param1 param2
```

要从 `sysbus` 上注册的 `uart` 外设读取偏移量为 0 的字节值，您可以键入：

```none
(machine-0) sysbus.uart ReadByte 0
```

Renode 使您能够获取和设置对象属性的值。如果未在命令末尾指定值，则将返回当前值。如果要设置值，则需要在末尾使用正确的 command 和 value。

例如，要将 `CyclesPerInstruction` 属性设置为 `0x000002`，您可以使用：

```none
(machine-0) cpu CyclesPerInstruction 0x000002
```

要获取这个新设置的属性的值，您必须使用：

```none
(Mi-V) cpu CyclesPerInstruction
0x00000002
```

设置索引器与设置属性非常相似，主要区别在于必须将参数放在方括号中：

```none
(machine-0) machine IndexerName [param1 param2 ...] Value
```

若要获取索引器的值，请使用：

```none
(machine-0) machine IndexerName [param1 param2 ...]
```

仿真对象属性的最后一种类型是字段。可以像访问属性一样访问它们的值：

```none
(machine-0) machine SystemBusName
sysbus
```

## 支持的数据类型

监控器支持多种数据类型:

```{list-table} Data types supported by the Monitor
:header-rows: 1
:widths: 50 50
* - Name of the data type
  - Example
* - comment
  - #comment or :comment
* - integer
  - 1
* - float
  - 1.1
* - hexadecimal
  - 0xfffff000
* - strings
  - "string"
* - range
  - <0xdefg, 0xefgh>
* - relative range
  - <0x6 0x2>
* - index
  - [0]
* - path
  - @path/to/file or @path\ with\ spaces
* - logical values
  - true or false (case insensitive)
* - variables and macros
  - $var
* - enum
  - Literal
```

```{note}
你可以将 `path` 参数传递给接受`字符串`的函数，它将被转换为`字符串` 。请记住，额外的文件名自动完成仅在 `@` 符号后可用。
```

## 监控变量类型

有三种方法可以在 Monitor 中创建单行变量：

* `$var="hello"`,
* `$var?="hello"`,
* `set var "hello"`.

`$var=` 和 `$var？=` 之间的区别在于，如果未设置变量，则后者表示默认值。Monitor 还允许您使用以下语法创建多行变量：

```none
set var """
"hello"
"""
```

````{note}
加载脚本时，您可以对多行变量使用略有不同的语法，将 `“”“` 标记放在第一行下方：

```none
set var
"""
"hello"
"""
```
````

Renode 中的变量是上下文相关的，这意味着对于每台计算机，您可以拥有具有相同名称的不同变量。您可以通过其完整路径访问它们。要在 `machine-0` 的计算机上下文中创建变量，您可以使用：

```none
(machine-0) $machine_0.var
```

如果使用`全局`前缀，您还可以创建全局变量：

```none
(monitor) $global.CWD
```

大多数情况下，一个短名称（没有上下文前缀）应该足以访问变量。

Monitor 允许您访问两个特殊变量：`$ORIGIN` 和 `$CWD`。

Renode 中可用的其他类型的变量是宏，它使您能够封装脚本的片段并轻松执行它们。您可以使用 `macro` 命令设置新的宏，其中包含新变量的名称和您希望它执行的命令：

```none
macro newMacro
> sysbus LoadELF $bin
```

您可以使用 Monitor 多行语法创建多行宏。然后，您可以使用 `runMacro` 命令执行新创建的宏：

```none
runMacro $newMacro
```

您可能会注意到，许多 Renode 示例脚本都定义了 `$reset` 宏。每当用户或通过模拟逻辑调用`机器 Reset` 方法时，都会使用此宏。

## 文件路径

每个 Renode 项目都需要您提供项目所需组件的文件路径。它们中的绝大多数使用 `@` 符号，它激活自动完成建议并表示文件的路径：

```none
include @/path/to/platform.repl
```

在解释路径时，Renode 会根据配置的内部`路径`在多个位置进行查找。默认情况下：

* 它首先检查 Renode 根目录
* 如果在根目录中找不到该文件，它将检查当前工作目录

您可以在 Monitor 中使用 `path` 命令检查和修改路径配置。

你也可以将路径作为 `“string”` 传递，但补全建议在这种情况下将不起作用。

```{note}
在 Renode 中，您可以使用绝对路径或相对于当前目录的路径。
```

### Relative paths

如果要表示相对于当前执行的 Renode 脚本 （.resc） 的路径，可以使用 `$ORIGIN` 变量：

```none
include $ORIGIN/my_subscript.resc
```

可以在 [fomu 脚本](https://github.com/renode/renode/blob/8ae7fdfc6cbe7b01952a8b2d4517d14aff7a297e/scripts/complex/fomu/renode_etherbone_fomu.resc#L5)中找到用法的示例。

```{note}
不要在基于 `$ORIGIN` 的路径的开头使用 `@`。
```

请记住，`$ORIGIN` 变量仅在脚本中可用 - 它不会在 Monitor 中交互工作。

在 Monitor 中，您可以使用特殊的 `$CWD` 变量来提供相对于当前工作目录的路径：

```none
(machine-0) include $CWD/my_script.resc
```

```{note}
There is no `@` at the beginning of the `$CWD`-based path.
```

```{note}
在 Robot 文件中，您还可以使用另一个变量：`${CURDIR}`。它是在 Robot Framework 级别处理和解决的，与 Renode 无关。以 `${CURDIR}` 开头的路径是相对于机器人文件位置的。

可以在 [LSM9DS1 测试](https://github.com/renode/renode/blob/8ae7fdfc6cbe7b01952a8b2d4517d14aff7a297e/tests/peripherals/LSM9DS1.robot#L24)中找到用法示例。

基于 `${CURDIR}` 的路径需要以 `@` 开头。由于它是在 Robot Framework 级别解析的，因此对于 Renode，它看起来就像用户提供的任何其他路径一样。

如果您想了解有关使用 Renode 进行测试的更多信息，请访问[专门讨论此主题的章节](../introduction/testing.md) 
```

(renode-script-syntax)=
## Renode Script 语法

您在 Renode 中的许多项目将涉及多次使用相同的平台和命令。使用 `.resc` 文件（即 Renode 脚本）可以显著加快此过程。`.resc` 文件的语法与 Monitor 的语法相同。

要加载 Renode 脚本，请使用 `include` 或 `i` 命令：

```none
include @/path/to/script.resc
```

```{note}
您可以使用 `start` 命令而不是 `include` 立即启动仿真。
```

`.resc` 文件的语法可以使用 Renode 中内置的许多脚本之一进行示例，例如 [Nordic Semiconductor NRF52840 脚本](https://github.com/renode/renode/blob/master/scripts/single-node/nrf52840.resc) ，我们将逐行介绍：
```none
:name: nRF52840
:description: This script runs Zephyr Shell demo on NRF52840.

using sysbus

mach create
machine LoadPlatformDescription @platforms/cpus/nrf52840.repl

$bin?=@https://dl.antmicro.com/projects/renode/renode-nrf52840-zephyr_shell_module.elf-gf8d05cf-s_1310072-c00fbffd6b65c6238877c4fe52e8228c2a38bf1f

showAnalyzer uart0

macro reset
"""
    sysbus LoadELF $bin
"""
runMacro $reset
```

前两行是注释，它们不由 Monitor 解释：

```none
:name: nRF52840
:description: This script runs Zephyr Shell demo on NRF52840.
```

接下来，我们使用 `using` 命令在引用外围设备时省略前缀。 `使用 sysbus` 的命令使您能够使用 `cpu` 而不是 `sysbus.cpu` 来引用 CPU 设备：

```none
using sysbus
```

然后，我们创建一个新机器

```none
mach create
```

我们从 `.repl` 文件加载平台描述：

```none
machine LoadPlatformDescription @platforms/cpus/nrf52840.repl
```

现在，我们加载一个示例 shell 二进制文件：

```none
$bin?=@https://dl.antmicro.com/projects/renode/renode-nrf52840-zephyr_shell_module.elf-gf8d05cf-s_1310072-c00fbffd6b65c6238877c4fe52e8228c2a38bf1f
```

接下来，我们为 `uart0` 设置一个 UART 分析器：

```none
showAnalyzer uart0
```

脚本的下一个元素是宏，每次重置计算机时都会执行该宏。

要在 `.resc` 中创建宏，您需要用 `“”`“（3 个双引号）将宏的代码括起来。宏的内容不需要缩进四个空格，但我们通常这样做是为了可读性。

下面的宏加载以前指定的 ELF 文件：

```none
    sysbus LoadELF $bin
```

最后，我们调用宏来运行它，因为它仅在调用后执行：

```none
runMacro $reset
```
