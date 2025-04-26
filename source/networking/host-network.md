# 连接到主机网络

Renode 允许您将主机网络接口连接到[模拟有线网络](../wired.md) 。

为此，您需要能够创建 `TAP` 接口。Renode 将尝试创建一个，前提是您有足够的权限。

## 关于时间流

在 Renode 中模拟的所有设备都在[虚拟时间](../advanced/time_framework.md)中运行，这通常比实时流慢。此外，网络数据包在周期性同步点传递，从而进一步延迟通信。

这意味着可能需要更改尝试从主机连接到模拟网络的应用程序设置的时间约束（例如超时），以考虑这些延迟。

## 打开 TAP 接口

要创建并打开一个 TAP 接口，该接口将在主机系统上列为 `tap0`，在 Renode 中列为 `host.tap`，请运行：

```none
(monitor) emulation CreateTap "tap0" "tap"
```

如果您希望在 Renode 关闭后保留接口，请添加 `true` 参数：

```none
(monitor) emulation CreateTap "tap0" "tap" true
```

根据您的系统配置，系统可能会要求您输入密码以打开界面。

:::{note}
需要在主机上启用和配置新创建的接口。

默认情况下，它没有分配 IP 地址，并且处于`关闭`状态。

有关进一步说明，请参阅您的系统文档。
:::

## 在 Windows 上使用 TAP 接口

为了在 Windows 上创建 TAP 设备，您需要安装作为 [OpenVPN 项目](https://openvpn.net/community-downloads/)一部分的第三方驱动程序。如果您的计算机上安装了 OpenVPN，Renode 应在尝试创建 TAP 接口时检测到它。该功能已在 `OpenVPN 2.5.6` 中进行了专门测试。

:::{note}
您需要管理员权限才能在 Windows 上创建 TAP。
:::

要在 Windows PC 上创建 TAP 接口，首先，您需要使用常用命令在 Renode 中创建新的 TAP 接口：

```none
(monitor) emulation CreateTap "tap0" "tap"
```

然后，您需要使用命令提示符为 TAP 分配一个 IP 地址：

```none
netsh interface ipv4 set address name=tap0 static X.X.X.X
```

:::{note}
您还可以使用 OpenVPN 的 `tapctl.exe` 驱动程序直接创建 TAP 接口：

```none
tapctl.exe create --name tap0
```

然后为其分配一个 IP 地址：

```none
netsh interface ipv4 set address name=tap0 static X.X.X.X
```

最后，使用以下方法将其连接到 Renode：

```none
(monitor) emulation CreateTap "tap0" "tap"
```
:::

## 连接 TAP 接口开关

假设您已经[配置了模拟网络](../wired.md) ，则可以通过运行以下命令将 TAP 接口连接到`交换机`设备：

```
(monitor) emulation CreateSwitch "switch"
(monitor) connector Connect host.tap switch
```

## 启动界面

TAP 接口创建为 “paused”。要启用与主机系统的通信，您必须通过运行以下命令手动启动它：

```none
(monitor) start
```

或者，如果您的仿真已启动，请使用：

```none
(monitor) host.tap Start
```

## 从主机传输文件

如果成功创建了 TAP 接口，则可以使用 `wget` 将文件从主机传输到仿真。为此，您需要使用与主机上的 TAP 接口关联的 IP 地址，例如：

```none
wget http://192.168.100.1/home/user/file.txt
```

:::{note}
还有其他文件传输方法： [内置 TFTP 服务器和 Virtio](../host-integration/sharing-files.md)。如果您想避免通过 TAP 进行主来宾联网的限制，建议使用这些方法。
:::
