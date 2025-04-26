# 在主机和模拟平台之间共享文件

Renode 有 4 种主要方法可以在 guest 和 host 之间共享文件：

1.  VirtIO block device,
2.  directory sharing,
3.  built-in TFTP server,
4.  [file transfer via the TAP interface](../networking/host-network.md).  [通过 TAP 接口传输文件](../networking/host-network.md)

有关基于 TAP 的传输的教程，请参阅[主机-来宾网络一章](../networking/host-network.md)。为了获得最佳性能和可移植性，建议使用 TFTP 或 VirtIO。

## 使用 VirtIO 块设备共享文件

VirtIO 是现代作系统中支持的虚拟化设备广泛使用的标准。使用 VirtIO 在 Renode 中共享文件的优势：

- 无处不在的驱动程序可用于各种来宾作系统
- 无处不在的驱动程序可用于各种来宾作系统，
- 无需在 Renode 中建模特定于平台的网络控制器
- 与模拟网络传输相比，传输速度更快

要使用 VirtIO 块设备，您需要准备文件系统镜像并使用您选择的资源填充它。首先准备一个目录，其中包含要构成文件系统的文件。

在 Linux 上，您可以使用以下命令创建文件系统：

```sh
$ truncate drive.img -s 128MB
$ mkfs.ext4 -d your-directory drive.img
```

然后，您需要将 VirtIO 块设备支持添加到模拟的 Linux 和 Renode 中的虚拟平台。

:::{note}
由于 Linux v2.6.25 支持 VirtIO 驱动程序，并且应该默认启用（检查 CONFIG_VIRTIO、CONFIG_VIRTIO_MMIO 和 CONFIG_VIRTIO_BLK 配置选项）。
:::

要启用 VirtIO，请添加一个描述设备总线位置和中断配置的设备树条目：

```dts
virtio@100d0000 {
    compatible = "virtio,mmio";
    reg = <0x100d0000 0x150>;
    interrupt-parent = <&plic>;
    interrupts = <42>;
};
```

现在，您需要向 Renode 平台定义（`.repl` 文件）添加相应的扩展名：

```none
virtio: Storage.VirtIOBlockDevice @ sysbus 0x100d0000
    IRQ -> plic@42
```

:::{note}
地址和中断行号在 `.repl` 和 DTS 之间必须一致。
:::

您可以使用以下方法为 VirtIO 块设备设置底层镜像：

```none
virtio LoadImage @drive.img
```

默认情况下，Renode 以非持久模式加载映像。如果你想让 VirtIO 块设备持久化，请在命令末尾添加 `true` 参数：

```none
virtio LoadImage @drive.img true
```

您可以使用 `dd` 等标准工具或`挂载`设备来访问它。默认情况下，VirtIO 设备在模拟 Linux 中列为 `/dev/vda`。

## 目录共享

目录共享功能在使用 VirtIO 文件系统设备时可用。

此设备支持客户机和主机之间的目录共享。使用此功能的优点：

- 实时主机和客户机文件传输
- 无需重新启动计算机即可上传新文件
- 无需重新打包映像或文件系统

### Libfuse

主机上的共享目录是使用 FUSE 文件系统守护程序执行的，该守护程序处理来自 Renode 的对目录的请求。共享目录必须是通过 Unix 域套接字连接的 FUSE 文件系统。已准备了一个直通文件系统，以便使用 libfuse 库与此设备一起使用。

### 安装

**要求** ：

- [Meson](http://mesonbuild.com/)
- [Ninja](https://ninja-build.org/)

文件系统守护程序安装：

```sh
$ git clone https://github.com/antmicro/libfuse --branch passthrough-hp-uds
$ cd renode-filesystem-sharing-libfuse
$ mkdir build; cd build
$ meson setup ..
$ ninja
```

文件系统守护程序二进制文件将位于 `example/passthrough_hp_uds`。为了便于使用，请将该二进制位置导出到 `$PATH`。

### 用法

:::{note}
从 5.4 版本开始，Linux 就支持 Virtiofs 驱动程序，并且应该默认启用（您可以在编译过程中检查 CONFIG_VIRTIO_FS config 选项）。请记住将 `virtiofs` 设备包含在设备树中的 `soc` 下。
:::

添加描述设备总线位置和中断配置的设备树条目：

```dts
virtio@100d0000 {
    compatible = "virtio,mmio";
    reg = <0x100d0000 0x150>;
    interrupt-parent = <&plic>;
    interrupts = <0x2>;
};
```

现在，您需要向 Renode 平台定义（`.repl` 文件）添加相应的扩展名：

```none
virtio: Storage.VirtIOFSDevice @ sysbus 0x100d0000
    IRQ -> plic@2
```

:::{note}
地址和中断行号在 `.repl` 和 DTS 之间必须一致。
:::

启动文件系统守护程序：

```sh
$ passthrough_hp_uds path_to_share
```

默认情况下，这会在 中创建 `/tmp/libfuse-passthrough-hp.sock` USD 套接字。

在 Renode 中创建 virtiofs 设备：

```none
virtio Create @/tmp/libfuse-passthrough-hp.sock "tag"
```

其中 `tag` 是您选择的名称。

在 guest 中，您现在可以挂载共享目录：

```none
# mount -t virtiofs tag /mnt
```

## 使用 TFTP 共享文件

TFTP（简单文件传输协议）是一种允许在客户端和远程主机之间传输文件的协议。使用 TFTP 在 Renode 中共享文件的优势：

- 单纯,
- 配置不需要对机器结构进行任何干预
- 一切都可以在 Monitor 中完成
- 不需要主机集成，适用于所有主机平台

在 Renode 中拥有内置的 TFTP 服务器，您不仅可以传输文件，还可以在确定性的模拟环境中轻松验证网络堆栈的正确性。

### 启动 TFTP 服务器

要在 Renode 中配置 TFTP，您需要创建一个[交换机并将其连接到您的计算机](../networking/wired.md) 。

现在，您可以创建 TFTP 服务器并将其连接到交换机：

```none
emulation CreateNetworkServer "server" "192.168.100.100"
connector Connect server switch
server StartTFTP 69
```

:::{note}
端口 69 是 TFTP 协议的默认值，但您可以提供 TFTP 客户端可接受的任何其他端口。
:::

成功启动服务器后，您可以通过 `server.tftp` 在 Monitor 中访问它

### 使用 TFTP 服务器

可以使用 `ServeFile` 命令通过 TFTP 服务器共享单个文件。

`ServeFile` 接受两个参数。第一个参数是主机文件的路径，第二个参数是通过 TFTP 公开它的名称：

```none
server.tftp ServeFile @path/to/file "filename"
```

:::{note}
第二个参数是可选的，如果未指定，则文件将以其原始名称公开。
:::

同样，您可以使用 `ServeDirectory` 通过 TFTP 共享目录：

```none
server.tftp ServeDirectory @path/to/directory
```

:::{note}
请记住，内置 TFTP 服务器不处理从客户机到主机的上传文件。
:::
