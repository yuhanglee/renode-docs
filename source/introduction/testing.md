# 使用 Renode 进行测试

## Renode 的测试能力

Renode 非常适合作为自动化测试场景的一部分，例如在 CI 服务器的后台运行。

Renode 与 [Robot Framework](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#introduction) 测试套件集成，并提供用户友好的脚本来运行测试。

它附带了各种准备好的测试脚本，但也允许您扩展它们或添加新的测试脚本。

## 运行 robot 测试脚本

在 Renode 中运行机器人测试脚本就像执行单个命令一样简单：

```
$ renode-test my_test.robot
```

上述命令将：

* 在后台启动 Renode 实例，
* 在端口 9999 上启用 Renode 的内置 Robot Framework 服务器（提供 Robot Framework 和 Renode 之间的接口）（用户可以更改端口号），
* 启动 Robot Framework 测试引擎并连接到 Renode，
* 运行提供的 `my_test.robot` 测试用例
* 在控制台上打印进度状态，
* 完成测试后生成 Log 和 Summary。

下面，您可以看到一个示例输出：

```none
Preparing suites
Started Renode instance on port 9999; pid 2293056
Starting suites
Running tests/platforms/LiteX-VexRiscv.robot
+++++ Starting test 'LiteX-VexRiscv.Timer Test'
+++++ Finished test 'LiteX-VexRiscv.Timer Test' in 46.79 seconds with status OK
+++++ Starting test 'LiteX-VexRiscv.I2C Test'
+++++ Finished test 'LiteX-VexRiscv.I2C Test' in 8.10 seconds with status OK
Cleaning up suites
Closing Renode pid 2293056
Aggregating all robot results
Output:  /home/antmicro/renode/output/tests/robot_output.xml
Log:     /home/antmicro/renode/output/tests/log.html
Report:  /home/antmicro/renode/output/tests/report.html
Tests finished successfully :)
```

```{note}
输出中的两个条目与单个测试用例相关联 - 一个在测试开始时，另一个在测试结束时（在并行运行测试时很有用）。第二条消息包含有关测试的持续时间和状态的信息。
```

运行的详细信息可在以下位置找到：

* `robot_output.xml` 报告（适用于自动解析）
* `log.html` 和 `report.html` 文件（适用于交互式检查）。

## 创建测试文件

Robot Framework 使用基于自定义语法的文本文件来表示测试用例。语法的详细信息可以在[官方文档](http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html)中找到。为了使机器人文件与 Renode 一起使用，需要适当的配置（如下所述）。

以下是与 Renode 一起使用的简单机器人测试文件的示例：

```
*** Settings ***
Suite Setup     Setup
Suite Teardown  Teardown
Test Teardown   Test Teardown
Resource        ${RENODEKEYWORDS}

*** Test Cases ***
Should Print Help
    ${x}=  Execute Command     help
           Should Contain      ${x}    Available commands:
```

`Should Print Help` 测试用例在 Renode 的监视器中执行 `help` 命令并验证结果。

与 Renode 的集成是通过向 settings 部分添加条目来实现的。`RENODEKEYWORDS` 变量（由 `renode-test` 脚本初始化）包含负责设置与 Renode 连接的 [renode-keywords.robot](https://github.com/renode/renode/blob/master/tests/renode-keywords.robot) 脚本的路径。其他设置配置套件/测试设置和拆解。

建议将上述 `Settings` 部分复制到每个新的 robot 测试文件中。

### 添加新的测试用例

每个 robot 测试文件可能包含许多测试用例。有关如何定义测试用例的一般说明，请参阅 [Robot Framework 文档](http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html) 。在本节中，我们将重点介绍如何使用 Robot Framework-Renode 集成。

Robot Framework-Renode 集成层提供关键字，允许用户以与内置关键字类似的方式直接从 Robot Framework 测试文件控制和检查仿真的状态。 [基本关键字](https://github.com/renode/renode/blob/master/src/Renode/RobotFrameworkEngine/RenodeKeywords.cs)允许用户：

* start the emulation （ `启动仿真` ）
* 清除仿真 （`Reset Emulation`）
* 在 Monitor 中执行命令 （`Execute Command`）
* 在 Renode 临时文件夹中分配一个文件 （`Allocate Temporary File`）
* 将文件下载到 Renode 临时文件夹（ `下载文件` ）

等等（有关详细信息，请参阅源代码）。

此外，Renode 还提供了[一组用于与 UART 设备交互的关键字](https://github.com/renode/renode/blob/master/src/Renode/RobotFrameworkEngine/UartKeywords.cs) 。它们允许：

* 将文本写入 UART（`Send Key To Uart`、 `Write Char On Uart`、`Write Line To Uart`）
* 等待特定行出现在 UART 上（`Wait For Line On Uart`， `Wait For Prompt On Uart`）
* 等待 UART 上的任何输出（ `等待 UART 上的下一行` ）
* 等待 UART 上缺少输出 （`Test If uart is idle`）

还有[一组用于与网络设备交互的关键字](https://github.com/renode/renode/blob/master/src/Renode/RobotFrameworkEngine/NetworkInterfaceKeywords.cs) ，这些关键字允许：


* 等待下一个传出网络数据包 （`Wait For Outgoing Packet`）
* 等待特定的传出网络数据包 （ `Wait For Outgoing Packet With Bytes At Index` ）

如有必要，可以通过在 C# 中实现更多关键字来扩展 Renode-Robot Framework 接口。

有关如何使用本节中提到的关键字的参考，请参阅 Renode 附带的 robot 测试文件。

## 高级用法

(robot-dependencies)=

### 测试用例依赖项

通常，单个 `.robot` 文件中的测试用例应彼此独立。事实上，默认的 test teardown 关键字调用 `Reset Emulation` 以确保一个测试的状态不会影响下一个测试。

但是，在某些情况下，可能需要将长时间运行的测试场景拆分为多个测试用例。这样可以更好地报告执行进度并提高整体性能（与重新运行测试的常见部分相比）。

Renode 提供了自定义 Robot 关键字来注释测试应从另一个测试提供的状态继续执行的情况：

* `Provides` - 创建仿真状态的命名快照（请参阅{ref}`状态保存与加载 <state-saving>` ）
* `Requires` - 加载命名快照并从该状态恢复执行测试

使用示例：
```
*** Test Cases ***
Boot Linux
    [...]
    Provides               booted-linux

Write to flash
    Requires               booted-linux
    Write Line To Uart     ...
    [...]

Ping another node
    Requires               booted-linux
    Write Line To Uart     ...
    [...]
```

有关实际示例，请参阅 [Zedboard.robot](https://github.com/renode/renode/blob/master/tests/platforms/Zedboard.robot#L29)。

````{note}
该机制还有另一种可用的实现，其中我们使用关键字 recording 来代替 state snapshots。在这种情况下，当使用 `Requires` 关键字时，将重新执行在相应 `Provides` 之前执行的所有关键字。要使用此模式，请在使用 Provides `关键字时`将 `Reexecution` 作为附加参数传递：

```
*** Test Cases ***
Boot Linux
    [...]
    Provides               booted-linux        Reexecution
```
````

### 使用单个命令运行多个测试文件

上一节中的示例显示了如何运行单个测试文件（可能仍包含许多测试用例）。可以运行许多测试文件并将结果聚合到单个报告中。为此，您需要将许多测试文件作为参数传递给 `renode-test` 命令：

```
renode-test my_tests.robot additional_tests.robot extra_tests.robot
```

测试将按照提供参数的顺序执行。

另一种方法是准备一个包含要运行的测试列表的 `yaml` 文件，例如：

```
- my_tests.robot
- additional_tests.robot
- extra_tests.robot
```

并使用特殊开关调用 `renode-test`：

```
$ renode-test -t my_tests.yaml
```

```{note}
`.yaml` 表示法允许用户包含其他 `.yaml` 文件，并对不应并行执行的项进行分组（请参阅下一节）。
```

### 并行运行测试

来自单个文件的测试用例始终按顺序执行（按文件中定义的顺序），但也可以并行运行来自不同文件的测试。为此，请使用特殊开关运行 `renode-test` 命令：

```
$ renode-test -j12 my_tests.yaml
```

这将允许您运行多达 12 个 Renode 实例，每个实例运行来自不同文件的测试用例。使用 `.yaml` 文件可以对不应并行运行的项进行分组（例如，因为它们共享端口号等资源）：

```
- my_tests.robot
- my_group:
    - my_test2.robot
    - my_test3.robot
```

在上面的示例中，`my_test2.robot` 在 `my_test3.robot` 之前执行，但与 `my_tests.robot` 并行执行。

你也可以将许多测试文件作为参数传递（即没有 `.yaml` 文件），但你将无法进行分组：

```
$ renode-test -j3 my_tests.robot my_tests2.robot my_tests3.robot
```

### 出错时停止

默认情况下，`renode-test` 将运行所有提供的测试用例。但是，可以在遇到第一个错误时停止执行。为此，请使用

```
renode-test --stop-on-error my_tests.robot
```

### 一次运行多个 renode-test 实例

Renode 通过网络套接字与 Robot Framework 执行器通信。这意味着同时运行两个 `test-renode` 实例将导致网络端口冲突。

为避免这种情况，您可以明确指定要用于 Robot Framework 和 Renode 之间通信的端口号：

```
$ renode-test -P 9997 my_test.robot &
$ renode-test -P 9998 my_test2.robot &
```

### 重复测试

可以使用

```
$ renode-test -n 10 my_test.robot
```

这将重新运行 `my_tests.robot` 中的所有测试用例 10 次。

### 运行选定的 Fixtures

可以使用以下方法仅运行文件中选定的测试用例：

```
$ renode-test -f "*GDB*" my_tests.robot
```

在上面的示例中，将仅执行名称中包含 `GDB` 的测试用例。

### 以交互方式运行测试

默认情况下，`renode-test` 命令在后台运行测试，并且仅将结果报告给控制台。但是，可以像运行 `renode` 命令时一样启用将日志消息打印到控制台：

```
$ renode-test --show-log my_tests.robot
```

```{note}
这将导致测试进度消息与日志消息混合在一起。
```

您还可以显示 Monitor 和 Analyzer 窗口并与之交互：

```
$ renode-test --enable-xwt my_tests.robot
```

```{note}
与正在运行的测试交互可能会影响结果。
```

### 保存失败测试的状态

Renode 的测试框架允许自动创建失败测试的快照，以便稍后加载它们以检查模拟的状态和/或进一步运行它们。此功能在非交互式 CI 环境中特别有用。

要启用失败测试快照的自动创建，请在运行 `renode-test` 命令之前设置 `RENODE_CI_MODE` 环境变量：

```
$ RENODE_CI_MODE=YES renode-test my_test.robot
```

每次创建快照时，都会为其指定一个与失败测试对应的名称，您将在控制台中看到一条消息，告知您快照的路径。所有快照都存储在 `output/tests/snapshots` 目录中。

```{note}
启用 CI 模式也会影响外部资源的处理方式 - 二进制文件缓存将被禁用，因此每次引用每个外部文件时都会下载它。
```

### 交互式检查失败的测试

使用 Renode，可以停止测试套件的执行，以便使用标准 Renode 接口（监视器、UART 分析器等）以交互方式调试失败的测试用例。

要启用此功能，请使用以下开关运行 `renode-test` 命令：

```
$ renode-test --debug-on-error my_test.robot
```

要启用此功能，请使用以下开关运行 `renode-test` 命令：

```{note}
此功能目前在无头环境中不可用。
```
