# 在不同模式下运行 Renode

默认情况下，Renode 在 GUI 模式下运行。启动后，它会为 [Monitor](#monito) 打开一个新窗口，并为不同的分析器（例如 UART）打开其他窗口。

但是，也可以在其他模式下启动 Renode。下面我们介绍运行 Renode 和与 Renode 交互的替代方法。

## Telnet 模式

Renode 可以通过网络套接字而不是窗口提供的 Monitor 界面启动。在此模式下，它仍将在本地打开其他窗口，但可以远程控制模拟。

要在 telnet 模式下启动 Renode，请使用 `-P` 开关运行它：

```
$ renode -P 1234
17:01:20.6362 [INFO] Loaded monitor commands from: /home/antmicro/renode/scripts/monitor.py
17:01:20.6800 [INFO] Monitor available in telnet mode on port 1234
17:02:17.2373 [INFO] Script: hello
```

之后，您可以通过以下方式连接到它：

```
$ telnet 127.0.0.1 1234
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Renode, version 1.12.0.33182 (0cd6e174-202108311750)

(monitor) log "hello"
(monitor)
```

您可以断开与 telnet 会话的连接，Renode 实例将继续在后台运行。

为了关闭以 telnet 模式打开的 Renode，您必须连接到它并发出 `quit` 命令。

## 无头模式

Renode 可以在无头模式下构建/运行，不需要系统中存在任何图形环境（例如 X11）。在 CI 环境中运行仿真时，此模式特别有用。

### 为 Headless 使用而构建

要在无头环境中构建 Renode，请使用 `--no-gui` 开关：

```sh
$ ./build.sh --no-gui
```

### 在 Headless 环境中运行

即使您的 Renode 具有编译的 GUI 支持（默认配置），您也可以通过以下方式在 Headless 模式下启动它：

```sh
$ renode --disable-gui
```

它类似于 telnet 模式，因为监视器默认在端口 1234 上可用（可以使用 `-P` 开关更改端口号）。区别在于，在 Headless 模式下，不会为分析器创建图形窗口 - 例如，默认情况下，UART 分析器将输出到 log。

## 控制台模式

可以在启动 Renode 的同一控制台窗口中启动 Monitor。在此模式下，提示将与日志消息交织在一起：

```
$ renode --console
16:54:37.2215 [INFO] Loaded monitor commands from: /home/antmicro/renode/scripts/monitor.py
Renode, version 1.12.0.33182 (0cd6e174-202108311750)

(monitor) log "hello"
16:54:41.0966 [INFO] Script: hello
(monitor)
```

:::{note}
通过传递 `--console` 和 `--disable-gui`，可以将控制台模式与无头模式混合在一起。
:::

## Monitor 中的 UART 交互

使用 `uart_connect` 命令，可以在 Monitor 和交互式 UART 会话之间切换：

```
[...]
(machine-0) uart_connect sysbus.uart0
Redirecting the input to sysbus.uart0, press <ESC> to quit...

Welcome to buildroot!
master login: root
# ls
# ls /
bin      etc      lib      media    opt      root     sbin     tmp      var
dev      home     linuxrc  mnt      proc     run      sys      usr
# Disconnected from sysbus.uart0
(machine-0)
```

在此模式下，用户的所有输入都定向到 UART，并且 UART 的所有输出都显示在 Monitor 提示符的位置。这适用于所有 Monitor 模式 （window， telnet， console）。
