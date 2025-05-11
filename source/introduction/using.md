# 使用 Renode

## 运行和退出 Renode

要启动 Renode，请在作系统的终端中使用 `renode` 命令。

当前终端现在将包含 Renode 日志窗口，并且将为“监视器”弹出一个新窗口 - Renode 的 CLI。

要正常退出 Renode 并关闭所有相关窗口，请随时在 Monitor 中使用 `quit` 命令。

(monitor)=

## 监控 （Renode CLI）

Monitor 用于与 Renode 交互并控制仿真。它公开了一组基本命令，并允许用户访问仿真对象。使用 Monitor，用户可以执行这些对象提供的作，以及检查和修改它们的状态。

Monitor 带有几个内置功能，使用户体验类似于常规终端应用程序。

### 使用内置命令

`help` 命令提供了可用内置命令的列表，并附有简短说明：

```none
(monitor) help
Available commands:
Name              | Description
================================================================================
alias             : sets an alias.
allowPrivates     : allow private fields and properties manipulation.
analyzers         : shows available analyzers for peripheral.
commandFromHistory: executes command from history.
createPlatform    : creates a platform.
currentTime       : prints out and logs the current emulation virtual and real time
...
```

您可以通过将 `help` 命令与另一个内置命令作为参数一起使用来获取有关所选命令的更多详细信息：

```none
(monitor) help analyzers
Usage:
------
analyzers [peripheral]
 lists ids of available analyzer for [peripheral]
analyzers default [peripheral]
 writes id of default analyzer for [peripheral]
```

键入任何具有错误或不完整参数的命令也将打印帮助字符串。

为了便于使用，我们提供了部分自动完成功能。只需按 <kbd>Tab</kbd> 一次即可完成当前命令，按两次即可查看所有可用建议。

对于带有 file 参数的命令，`@` 符号表示文件的路径;为方便起见，Renode 还为文件名提供了自动补全功能。

```{note}
在 `@` 符号之后，Monitor 将建议运行 Renode 的当前工作目录中的文件和 Renode 安装目录中的文件作为后备 - 在出现歧义时，前者优先。对于不存在的文件，Renode 目录优先。如果要在本地目录中创建文件，请使用 `$CWD` 变量或提供完整的路径。
```

最常见的命令（例如，`start` 或 `quit`）提供简短的别名，通常是单字母的别名（分别为 `s` 和 `q`）。

CLI 提供带有交互式搜索  <kbd>Ctrl</kbd>+<kbd>R</kbd> 的命令历史记录（箭头 <kbd>↑</kbd>/<kbd>↓</kbd> ），以便轻松重新执行以前的命令。

也可以使用  <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>V</kbd> 进行粘贴，也可以通过右键单击的上下文菜单进行粘贴。要擦除当前命令并返回干净提示符，请使用： <kbd>Ctrl</kbd>+<kbd>C</kbd> 。

## 基本的交互式工作流程

当以交互方式运行 Renode 时，用户通常会首先通过一系列命令 {ref}`创建模拟环境 <working-with-machines>` ，从而构建、配置和连接相关的仿真（来宾）平台或平台（称为“机器”）。

这通常使用嵌套的 {ref}`scripts` 来完成，这些脚本有助于封装此活动中的一些可重复元素（通常，用户希望在两次运行之间一遍又一遍地创建相同的平台，甚至完全编写执行脚本）。

创建仿真并加载所有必要的元素（包括要执行的二进制文件）后，可以启动仿真本身 - 为此，请使用 Monitor 中的 `start` 命令。

此时，您将能够在[记录器窗口中](../basic/logger.md)看到有关模拟环境作的大量信息，提取其他信息并使用监视器（或 [Wireshark](../networking/wireshark.md) 等插件）作正在运行的仿真，以及与模拟机器的外部接口（如 UART 或[以太网控制器](../networking/wired.md)）进行交互。

有关从 Monitor 创建和作计算机时有用的一些典型命令，您可以参考  {ref}`working-with-machines` 部分。

有关与仿真交互的更多命令和信息，请参阅  {ref}`basic-control`  部分。

(scripts)=

## .resc 脚本
-------------

Renode 脚本 （.resc） 使您能够封装项目的可重复元素（如创建机器和加载二进制文件），以便方便地多次执行它们。`.resc` 文件中使用的语法与 Monitor 的语法相同。

Renode 有许多内置的 `.resc` 文件，比如这个 [Intel Quark C1000 脚本](https://github.com/renode/renode/blob/master/scripts/single-node/quark_c1000.resc) 。

要在 Renode 中加载它，请使用带有路径的 `include` 命令：

```
include @scripts/single-node/quark_c1000.resc
```

如果在上述命令中使用 `start` （或仅使用 `s`） 而不是 `include`，则仿真将在加载脚本后立即开始。

```{note}
请记住使用 `@` 后面的 Tab 键进行路径自动补全，如 {ref}`上一节 <monitor>` 所述。
```



[内置的 Renode 演示脚本](https://github.com/renode/renode/tree/master/scripts)是一个很好的切入点 - 要运行您的第一个演示，请继续阅读[运行您的第一个演示](https://renode.readthedocs.io/en/latest/introduction/demo.html)章节。

## 配置用户界面

用户界面的外观可以通过用户配置文件`配置`进行自定义。它位于类 Unix 系统上的 `~/.config/renode` 目录中，以及 Windows 上的 `AppData\Roaming\renode` 中。

在 `[termsharp]` 部分中，可以使用以下设置：
  
| Name            | Description                                                                           |
| --------------- | ------------------------------------------------------------------------------------- |
| append-CR-to-LF | 此设置控制是否将回车附加到每个换行符。允许的值为 `true` 和 `false`。默认值为 `true`。 |
| font-face       | 日志和监控窗口中使用的 TrueType 字体的名称。默认值为 `Roboto Mono`。                  |
| font-size       | 字体大小（以磅为单位）。默认值在 Windows 上为 12，在 Linux 上为 10。                  |
| window-width    | 日志和监视器窗口的初始宽度（以像素为单位）。                                          |
| window-height   | 日志和监视器窗口的初始宽度（以像素为单位）。                                          |
