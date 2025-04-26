# UART 集成

Renode 可以将仿真的 UART 设备公开给主机，并使用标准工作流程（工具、脚本等）与它们进行交互，就像它们是实际的硬件串行端口一样。这为设计混合设置打开了大门，其中系统的一部分在 Renode 中模拟，其余部分由在 “真实世界” 中运行的物理设备或软件组成。

Renode 提供了两种独立的机制，用于将虚拟 UART 设备暴露给主机：

- 使用 pty 设备（仅限 Linux/macOS）
- 通过网络套接字（在所有平台上都可用）

您还可以将 UART 输出重定向到文件，但这不允许读取用户输入。

## UART pty 终端

:::{note}
此功能仅在 Linux/macOS 上可用。
:::

UART pty 终端集成允许在主机文件系统中创建一个 pty 设备，作为现实世界和模拟之间的桥梁。在现实世界中运行的软件写入 pty 设备/从 pty 设备读取的数据会自动传输到虚拟 UART 设备（反之亦然）。

要公开虚拟 UART 设备，请在 Monitor 中使用以下命令创建 UART pty 终端：

```none
(monitor) emulation CreateUartPtyTerminal "term" "/tmp/uart"
```

这将在主机文件系统 （`/tmp/uart`） 中创建一个新文件，该文件可以在模拟中作为`术语`引用。

现在您需要[加载您的平台](#loading-platforms)并将新创建的 UART pty 终端连接到模拟的 UART 设备：

```none
(machine-0) connector Connect sysbus.uart0 term
```

这假设您的 UART 名为 `sysbus.uart0`，但您可能需要调整它以匹配您的平台。

最后，在主机上打开终端应用程序（`screen`/`picocom`/`PuTTY`/等）并将其附加到 `/tmp/uart` 文件。您可以像与硬件一样与它交互。

## 套接字终端

:::{note}
此功能在所有支持的主机平台 （Linux/macOS/Windows） 上都可用。
:::

套接字终端集成允许通过网络套接字公开虚拟 UART 设备，并使其可用于网络中的任何设备。在现实世界中运行的软件（在本地主机或通过网络）发送到所选端口号上可用套接字的数据会自动传输到虚拟 UART 设备（反之亦然）。

为了公开虚拟 UART 设备，请在 Monitor 中使用以下命令创建一个 Socket 终端：

```none
(monitor) emulation CreateServerSocketTerminal 12345 "term"
```

这将在主机上打开一个 tcp 网络端口 `12345`，该端口可以在模拟中作为`术语`引用。

现在，将新创建的 Socket 端子连接到模拟的 UART 设备：

```none
(machine-0) connector Connect sysbus.uart0 term
```

最后，在您的主机上打开终端应用程序（`netcat`/`telnet`/`PuTTy`/等）并将其连接到端口 12345。您可以像与硬件一样与它交互。.

### 发出配置字节

默认情况下，Server Socket 将发出以下初始配置字节，以便正确配置新连接的终端：

```
0xff, 0xfd, 0x00, // IAC DO    BINARY
0xff, 0xfb, 0x01, // IAC WILL  ECHO
0xff, 0xfb, 0x03, // IAC WILL  SUPPRESS_GO_AHEAD
0xff, 0xfc, 0x22  // IAC WONT  LINEMODE
```

为了避免生成它们，请在创建 Socket 终端时传递额外的 `false` 参数：

```none
(monitor) emulation CreateServerSocketTerminal 12345 "term" false
```

## 重定向到文件

Renode 可以将 UART 输出重定向到主机上的文件。要启用此功能，请调用：

```none
(machine-0) uart CreateFileBackend @uart_file
```

默认情况下，UART 输出将由主机 IO 系统缓存。如果您希望在发送输出后立即刷新输出，请使用：

```none
(machine-0) uart CreateFileBackend @uart_file_flush true
```

如果需要，您可以通过以下方式停止此输出：

```none
(machine-0) uart CloseFileBackend @uart_file
```

请记住，对 `CreateFileBackend` 方法的后续调用不会覆盖同名的上一个文件，而是复制，将连续数字追加到其名称中。
