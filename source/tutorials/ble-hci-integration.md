# BLE HCI 与 Renode 的集成

低功耗蓝牙标准允许对 HCI 协议使用不同的传输方式（例如 UART、USB、SPI）。Renode 支持通过 [HCI UART](https://github.com/zephyrproject-rtos/zephyr/blob/21f473aaecf087e357df19c78ae2bd60aed345dd/samples/bluetooth/hci_uart/README.rst) 与外部 BLE 控制器集成，并且该集成可用于在 [Zephyr RTOS 中测试 BLE 主机堆栈](https://github.com/zephyrproject-rtos/zephyr/blob/21f473aaecf087e357df19c78ae2bd60aed345dd/doc/connectivity/bluetooth/bluetooth-arch.rst#build-types) 。

```{note}
本教程使用 [Zephyr 3.4.0](https://github.com/zephyrproject-rtos/zephyr/tree/zephyr-v3.4.0) 和随附的 [Zephyr SDK 0.16.1](https://github.com/zephyrproject-rtos/sdk-ng/releases/tag/v0.16.1) 进行了测试。
```

## 与 Android 模拟器集成

[Android Emulator](https://developer.android.com/studio/emulator_archive)（[Android Studio IDE](https://developer.android.com/studio/intro) 的一部分，也可以作为独立工具安装）支持通过 [netsim 工具](https://cs.android.com/android/_/android/platform/tools/netsim/+/a31d6d4930154cf0f211b645667056520a2a7209:;bpv=0;bpt=0)进行蓝牙通信。可以将 Renode 中模拟的 Zephyr 的 BLE 主机堆栈连接到 `netsim` 的通道，以测试 Android 设备和模拟板之间的通信。

### 为 BLE HCI 演示构建 Zephyr 示例

您可以在 [Zephyr 文档中找到 Zephyr](https://docs.zephyrproject.org/latest/samples/index.html) 示例和演示的完整列表，其中详细介绍了如何安装和使用 RTOS。

我们将选择 [nRF52840 DK](https://docs.zephyrproject.org/latest/boards/arm/nrf52840dk_nrf52840/doc/index.html) 板作为示例目标。为了能够在您的系统上生成一些 Zephyr 二进制文件，请先完成 [Zephyr 入门指南](https://docs.zephyrproject.org/latest/getting_started/index.html) 。接下来，在 `zephyr` 目录中创建一个 `nrf52840dk_nrf52840_ble_hci_uart.overlay` 文件。此文件用于将选定的 UART 分配给 BLE HCI 传输。此文件应包含以下内容：

```dts
/ {
	chosen {
		zephyr,bt-uart = &arduino_serial;
	};
};

&arduino_serial {
    status = "okay";
};
```

在大多数情况下，可以为其他本身不支持 BLE 的板创建类似的叠加层。

现在，您可以使用以下命令为 nRF52840 DK 构建 [peripheral_hr](https://docs.zephyrproject.org/latest/samples/bluetooth/peripheral_hr/README.html) 示例。该示例是在启用 HCI UART 传输的情况下构建的：

```sh
west build -p auto -b nrf52840dk_nrf52840 -d hci_peripheral_hr samples/bluetooth/peripheral_hr -- \
 -DCONFIG_BT_HCI=y -DCONFIG_BT_CTLR=n -DCONFIG_BT_H4=y \
 -DCONFIG_BT_EXT_ADV=n -DCONFIG_BT_HCI_ACL_FLOW_CONTROL=n \
 -DDTC_OVERLAY_FILE=$PWD/nrf52840dk_nrf52840_ble_hci_uart.overlay
```

构建的二进制文件应位于 中 `hci_peripheral_hr/zephyr/zephyr.elf` 。

构建期间使用的配置选项可能会有所不同，具体取决于仿真目标板和集成期间使用的任何外部 BLE 控制器。有关可能的选项 [，请参阅 Zephyr RTOS 文档](https://docs.zephyrproject.org/latest/kconfig.html#!CONFIG_BT) 。

要在 Renode 中使用这些二进制文件，请加载您的平台并通过 socket 终端将选定的 UART 公开给主机，以便与外部 BLE 控制器集成：

```none
(machine-0) emulation CreateServerSocketTerminal 3456 "ble_hci_uart" false
(machine-0) connector Connect sysbus.uart1 ble_hci_uart
```

您还可以使用与 Renode 一起分发的 `nrf52840dk_nrf52840` 板的通用脚本，您只需在加载脚本之前为端口号 （`$port`） 和二进制路径 （`$bin`） 设置一些变量。您可以从命令行运行此命令：

```sh
renode -e "$port=3456; $bin=@/home/user/zephyrproject/zephyr/hci_peripheral_hr/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
```

如果您尝试此时运行仿真，Zephyr RTOS 将由于缺少与 BLE 控制器的连接而断言超时。在下一步中，您将学习如何连接到 BLE 控制器以运行完整示例。

```{note}
在 Linux 或 macOS 上，您还可以使用 [pty 终端](https://renode.readthedocs.io/en/latest/host-integration/uart.html) 。
```

### 设置 Android 模拟器

```{note}
需要 Android Emulator 版本 33.1.14 或更高版本才能测试蓝牙集成。
```

在从命令行设置 Android 仿真器之前，您需要在系统上安装 `Java 运行时环境` 。要在基于 Debian 的系统上安装先决条件，请使用以下命令：

```sh
sudo apt install default-jre unzip wget
```

现在，您可以运行以下命令来设置 Android 模拟器：

```{note}
您可以将 `BASE_PATH` 设置为要下载 Android SDK 的目录。
```

```sh
export BASE_PATH=$HOME

export ANDROID_HOME=$BASE_PATH/android-sdk

export CMDLINE_TOOLS_DIR=$ANDROID_HOME/cmdline-tools
export SDK_LATEST=$CMDLINE_TOOLS_DIR/latest
export EMULATOR_DIR=$ANDROID_HOME/emulator
export PLATFORM_TOOLS_DIR=$ANDROID_HOME/platform-tools

export PATH=$PATH:$EMULATOR_DIR:$PLATFORM_TOOLS_DIR:$SDK_LATEST/bin/
export ANDROID_AVD_HOME=$ANDROID_HOME/system-images

export TEMP_CMDLINE_ZIP=$CMDLINE_TOOLS_DIR/commandlinetools.zip
export TEMP_EMULATOR_ZIP=$ANDROID_HOME/emulator.zip

mkdir -p $CMDLINE_TOOLS_DIR

# https://developer.android.com/studio#command-tools
CMD_DOWNLOAD_URL=https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip

wget -O $TEMP_CMDLINE_ZIP $CMD_DOWNLOAD_URL
unzip $TEMP_CMDLINE_ZIP -d $CMDLINE_TOOLS_DIR
mv $CMDLINE_TOOLS_DIR/cmdline-tools $SDK_LATEST
rm $TEMP_CMDLINE_ZIP

yes | sdkmanager --licenses
sdkmanager --install "platform-tools"
sdkmanager --install "platforms;android-34"

# https://developer.android.com/studio/emulator_archive
EMULATOR_VERSION=33.1.14
EMULATOR_DOWNLOAD_URL=https://redirector.gvt1.com/edgedl/android/repository/emulator-linux_x64-9997245.zip

wget -O $TEMP_EMULATOR_ZIP $EMULATOR_DOWNLOAD_URL
unzip $TEMP_EMULATOR_ZIP -d $ANDROID_HOME
rm $TEMP_EMULATOR_ZIP

sdkmanager --install "system-images;android-34;google_apis;x86_64"

export AVD_NAME_0=Android0
echo "no" | avdmanager --verbose create avd -n $AVD_NAME_0 -k "system-images;android-34;google_apis;x86_64"
```

此脚本下载并安装最新版本的 Android SDK 命令行工具和 Android 模拟器。它还会根据 Android API 级别 34 系统映像创建名为 `Android0` 的 Android 虚拟设备 （AVD）。

### 一起运行 Renode 和 Android Emulator

要将 HCI 数据包从在 Renode 中创建的套接字终端传输到 Android 中的虚拟控制器，您需要使用 [bumble](https://google.github.io/bumble/) 将 gRPC 协议（Android Emulator 用于通信）消息解码为 HCI 命令。

```{note}
不要在 conda 环境中安装 `bumble`，因为与 Anaconda 一起分发的`套接字`模块不支持蓝牙套接字。尝试绑定到 HCI 套接字时会导致异常：

    AttributeError: module 'socket' has no attribute 'AF_BLUETOOTH'
    Exception: Bluetooth HCI sockets not supported on this platform

You can use:  您可以使用：

    pipx install git+https://github.com/google/bumble.git@8eeb58e467 

将其安装在隔离的环境中。
```

安装 `bumble` 模块：

```sh
python -m pip install git+https://github.com/google/bumble.git@8eeb58e467
```

要查看 `bumble-hci-bridge` 是否可用作全局工具，请运行：

```sh
bumble-hci-bridge --help
```

运行 Android 模拟器：

```sh
emulator -avd Android0 -accel auto -gpu auto
```

现在，您可以使用 `bumble-hci-bridge` 在上一步中配置的 Renode 设备和 Android 模拟器之间建立连接：

```sh
bumble-hci-bridge tcp-client:127.0.0.1:3456 android-netsim 0x03:0x0031,0x08:0x013,0x08:0x032,0x08:0x016,0x03:0x035
```

要将 Renode 与 Android 模拟器连接：

1. 启动 Renode 模拟（您的平台应连接到之前创建的 HCI 网桥）。
2. 在模拟的 Android 设备上，打开 Chrome 浏览器并转到 https://webbluetoothcg.github.io/demos/heart-rate-sensor/。
3. 单击页面并授予蓝牙访问权限。

您应该能够查看并连接到在 Renode 中模拟的 BLE 外设，如下所示。

![https://webbluetoothcg.github.io/demos/heart-rate-sensor/](img/ble_hci_chrome_heart_rate_monitor.png) ![Connect BLE peripheral](img/ble_hci_chrome_heart_rate_monitor_connect.png) ![Connected BLE peripheral](img/ble_hci_chrome_heart_rate_monitor_connected.png)

## 蓝牙 Mesh 组网

Zephyr BLE 堆栈提供对 BLE Mesh 协议的支持。您可以创建具有多个 Renode 实例的虚拟网状网络，这些实例模拟单独的 BLE 设备。

### 构建 Zephyr BLE Mesh 示例

构建将在 Renode 中为 `nrf52840dk_nrf52840` 平台加载的[`网格`示例](https://github.com/zephyrproject-rtos/zephyr/tree/zephyr-v3.4.0/samples/bluetooth/mesh) ，如下所示：

```sh
west build -p auto -b nrf52840dk_nrf52840 -d hci_mesh samples/bluetooth/mesh -- \
 -DCONFIG_BT_HCI=y -DCONFIG_BT_CTLR=n -DCONFIG_BT_H4=y \
 -DCONFIG_BT_EXT_ADV=n -DCONFIG_BT_HCI_ACL_FLOW_CONTROL=n \
 -DCONFIG_BT_SETTINGS=n -DCONFIG_NVS=n -DCONFIG_SETTINGS=n -DCONFIG_HWINFO=n \
 -DDTC_OVERLAY_FILE=$PWD/nrf52840dk_nrf52840_ble_hci_uart.overlay
```

构建的二进制文件应位于 `hci_mesh/zephyr/zephyr.elf` 中。

### 与外部 BLE 控制器集成

在主机端，您可以使用支持连接到物理或虚拟 BLE 控制器的工具：[BlueZ](https://github.com/bluez) 的 `btvirt` 和 `btproxy` 或 `bumble 的 bumble-hci-bridge` 和 `bumble-link-relay`。

[将 BlueZ 与 Zephyr 结合使用中描述了 BlueZ](https://docs.zephyrproject.org/latest/connectivity/bluetooth/bluetooth-tools.html#using-bluez-with-zephyr) 用例，并且 BLE 集成的各个方面在 [bumble 项目中](https://google.github.io/bumble/platforms/index.html)都有详细记录，该项目为许多[常见的权限问题](https://google.github.io/bumble/platforms/linux.html)提供了解决方案。

### 创建虚拟控制器

```{note}
这部分依赖于 BlueZ 和主机系统上的 `/dev/vhci` 设备，因此仅限于 Linux 主机。在其他系统上，您可以在 Linux 虚拟机中对其进行测试。如果要在 Docker 容器中运行这些步骤，则应使用以下标志启动 `--device=/dev/vhci --net=host --cap-add=CAP_NET_ADMIN` 它。
```

`btvirt` 工具是 [Ubuntu 上的 `bluez-tests`](https://packages.ubuntu.com/kinetic/amd64/bluez-tests/filelist) 和 [Debian 上的 `bluez-test-tools`](https://packages.debian.org/bullseye/amd64/bluez-test-tools/filelist) 的一部分。可以使用 `apt` 包管理器安装它。

要在基于 Debian 的系统上从源代码构建 `bluez`：

```sh
sudo apt update

sudo apt install -y wget xz-utils git bc libusb-dev libdbus-1-dev libglib2.0-dev libudev-dev libical-dev libreadline-dev autoconf bison flex libssl-dev libncurses-dev libdbus-1-dev python3-docutils cmake udev systemd

wget https://github.com/json-c/json-c/archive/refs/tags/json-c-0.16-20220414.tar.gz
tar xvf json-c-0.16-20220414.tar.gz
cd json-c-json-c-0.16-20220414
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release -DBUILD_STATIC_LIBS=OFF ..
make
sudo make install

wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.68.tar.xz
tar -xvf bluez-5.68.tar.xz
cd bluez-5.68
./configure --enable-mesh --enable-testing --enable-tools --enable-deprecated --enable-experimental --prefix=/usr --mandir=/usr/share/man --sysconfdir=/etc --localstatedir=/var
make -j$(nproc)
```

从源代码构建时，`btvirt` 位于 `emulator` 目录中。

默认情况下，可以使用 `btvirt` 创建的虚拟蓝牙控制器的数量限制为 16 个设备。如果您想使用 Renode 来模拟由更多设备组成的 BLE Mesh 网络，您可以增加编译时间常[数 MAX_BTDEV_ENTRIES](https://github.com/bluez/bluez/blob/5.68/emulator/btdev.c#L251) 并重建 BlueZ。如果要超过 128 个虚拟设备，则还应增加另一个常[数 MAX_MAINLOOP_ENTRIES](https://github.com/bluez/bluez/blob/5.68/src/shared/mainloop.c#L47)。

```{note}
本教程中使用的命令假定您的系统上没有任何可用的 HCI 接口。您可以使用 `hciconfig` 命令进行验证。如果已注册 HCI 接口，请使用 `hci-socket：<i>` 中的相应索引来说明它。`btvirt` 命令为虚拟控制器创建 HCI。您可以使用 `hciconfig` 检查创建的接口。
```

要创建一个由 Renode 中模拟的三个设备组成的 BLE Mesh 网络，这些设备在 Renode 中相互对称通信，请在单独的终端中运行以下命令：

```sh
$ sudo btvirt -d -l3
$ renode -e "$port=3456; $bin=@/home/user/zephyrproject/zephyr/hci_mesh/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
$ renode -e "$port=3457; $bin=@/home/user/zephyrproject/zephyr/hci_mesh/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
$ renode -e "$port=3458; $bin=@/home/user/zephyrproject/zephyr/hci_mesh/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
$ sudo capsh --caps="cap_net_admin+eip cap_setpcap,cap_setuid,cap_setgid+ep" --keep=1 --user=\$USER --addamb=cap_net_admin  -- -c "\$(which bumble-hci-bridge) tcp-client:127.0.0.1:3456 hci-socket:0"
$ sudo capsh --caps="cap_net_admin+eip cap_setpcap,cap_setuid,cap_setgid+ep" --keep=1 --user=\$USER --addamb=cap_net_admin  -- -c "\$(which bumble-hci-bridge) tcp-client:127.0.0.1:3457 hci-socket:1"
$ sudo capsh --caps="cap_net_admin+eip cap_setpcap,cap_setuid,cap_setgid+ep" --keep=1 --user=\$USER --addamb=cap_net_admin  -- -c "\$(which bumble-hci-bridge) tcp-client:127.0.0.1:3458 hci-socket:2"
```

您可以调用 `hciconfig` 来确保虚拟 HCI 是由 `btvirt` 创建的。您应该会看到属于虚拟总线 （`Bus： Virtual`） 的 HCI。它们在您终止 `btvirt` 进程后消失。

```none
hci2:	Type: Primary  Bus: Virtual
	BD Address: 00:AA:01:02:00:02  ACL MTU: 192:1  SCO MTU: 0:0
	DOWN 
	RX bytes:0 acl:0 sco:0 events:46 errors:0
	TX bytes:523 acl:0 sco:0 commands:46 errors:0

hci1:	Type: Primary  Bus: Virtual
	BD Address: 00:AA:01:01:00:01  ACL MTU: 192:1  SCO MTU: 0:0
	DOWN 
	RX bytes:0 acl:0 sco:0 events:46 errors:0
	TX bytes:523 acl:0 sco:0 commands:46 errors:0

hci0:	Type: Primary  Bus: Virtual
	BD Address: 00:AA:01:00:00:00  ACL MTU: 192:1  SCO MTU: 0:0
	DOWN 
	RX bytes:0 acl:0 sco:0 events:46 errors:0
	TX bytes:523 acl:0 sco:0 commands:46 errors:0
```

```{note}
为了能够测试在 Renode 中运行的替代 Zephyr BLE 堆栈，请确保[禁用蓝牙服务](https://github.com/google/bumble/blob/v0.0.161/docs/mkdocs/src/platforms/linux.md#using-hci-sockets) 。要查看基于 Debian 的系统上蓝牙服务的状态，请运行 `sudo systemctl status bluetooth` 。

在某些情况下，您可能需要使用 `sudo rfkill unblock all` 来取消阻止您的蓝牙无线设备。
```

或者，您可以使用 `socat` 中继工具和 `btproxy` 将虚拟 HCI 连接到 Renode，当 BlueZ 从源代码构建时， `该工具位于 tools` 目录中。如果您不想为 `bumble-hci-bridge` Python 程序分配临时扩展功能，则可以使用这些命令。

```sh
$ sudo btvirt -d -l3
$ renode -e "$port=3456; $bin=@/home/user/zephyrproject/zephyr/hci_mesh/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
$ renode -e "$port=3457; $bin=@/home/user/zephyrproject/zephyr/hci_mesh/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
$ renode -e "$port=3458; $bin=@/home/user/zephyrproject/zephyr/hci_mesh/zephyr/zephyr.elf; i @scripts/complex/hci_uart/hci_uart.resc"
$ sudo ./tools/btproxy -u"/tmp/bt-server-bredr0" -i 0
$ sudo ./tools/btproxy -u"/tmp/bt-server-bredr1" -i 1
$ sudo ./tools/btproxy -u"/tmp/bt-server-bredr2" -i 2
$ socat TCP-CONNECT:127.0.0.1:3456 UNIX-CONNECT:/tmp/bt-server-bredr0
$ socat TCP-CONNECT:127.0.0.1:3457 UNIX-CONNECT:/tmp/bt-server-bredr1
$ socat TCP-CONNECT:127.0.0.1:3458 UNIX-CONNECT:/tmp/bt-server-bredr2
```

```{note}
您可以在 `btproxy` 中使用 TCP 服务器而不是 Unix 服务器，但命令略有修改：

    sudo ./tools/btproxy -l127.0.0.1 -p1000 -i 0
    socat TCP-CONNECT:127.0.0.1:3456 TCP-CONNECT:127.0.0.1:1000
```

下一个：

1. 启动 Renode 模拟（您的平台应连接到您之前创建的 HCI 网桥）。
2. 在 Renode 的监视器中为每个 Renode 实例输入 `gpio0.sw0 PressAndRelease`，以将 BLE 设备配置到 Mesh 网络。
3. 所有连续的按钮按下 （`gpio0.sw0 PressAndRelease`） 将导致消息发送到网络中的其他节点，并且 `led0` 在所有电路板上闪烁。
4. 您可以使用 `watch “gpio0.led0 State” 200` 命令在 Renode 的监视器中观察 `led0` 状态，该命令每 200 毫秒打印一次 LED 状态。

应在 `uart0` 控制台上打印以下消息：

```none
*** Booting Zephyr OS build zephyr-v3.3.0 ***
Initializing...
[00:00:00.041,351] <inf> bt_hci_core: bt_dev_show_info: Identity: C8:7F:54:3D:8E:49 (public)
[00:00:00.041,442] <inf> bt_hci_core: bt_dev_show_info: HCI: version 5.1 (0x0a) revision 0x09a9, manufacturer 0x005d
[00:00:00.041,442] <inf> bt_hci_core: bt_dev_show_info: LMP: version 5.1 (0x0a) subver 0x8a6b
Bluetooth initialized
[00:00:00.041,534] <inf> bt_mesh_prov_device: bt_mesh_prov_enable: Device UUID: 00000000-0000-0000-0000-00000000dddd
Mesh initialized
Self-provisioning with address 0x1fd8
[00:00:08.152,130] <inf> bt_mesh_main: bt_mesh_provision: Primary Element: 0x1fd8
[00:00:08.152,130] <dbg> bt_mesh_main: bt_mesh_provision: net_idx 0x0000 flags 0x00 iv_index 0x0000
Provisioned and configured!
Sending OnOff Set: on
set: on delay: 0 ms time: 0 ms
set: off delay: 0 ms time: 0 ms
set: on delay: 0 ms time: 0 ms
```

BLE 设备在单独的 Renode 实例中仿真，并通过 Linux 作系统拥有的虚拟 HCI 接口在虚拟网络中进行通信。您可以使用 HCI 协议剖析器在 Wireshark 中观察这些接口上的流量。

如果您有多台带有物理 BLE 控制器（USB BLE 适配器或内置 BLE 模块）的计算机，您可以使用 Renode 模拟不同计算机上的 BLE 设备，并使它们在 BLE Mesh 网络中进行通信，以实现真正的物理布置。

### 与物理 BLE 控制器集成

可以使用外部 USB BLE 适配器（您可以从 [Zephyr HCI USB 示例](https://github.com/zephyrproject-rtos/zephyr/blob/zephyr-v3.4.0/samples/bluetooth/hci_usb/README.rst)构建一个）或内置 BLE 模块进行集成。

要使用 `bumble-hci-bridge` 在 Renode 和 USB BLE 适配器之间建立连接：

```sh
bumble-hci-bridge tcp-client:127.0.0.1:3456 usb:0
```

```{note}
您可能需要更改 USB 设备的权限才能[以普通用户身份访问它](https://github.com/google/bumble/blob/v0.0.161/docs/mkdocs/src/platforms/linux.md#using-a-usb-dongle) 。
```

如果内核已经注册了一个接口，则可以直接连接到 HCI 套接字：

```sh
sudo capsh --caps="cap_net_admin+eip cap_setpcap,cap_setuid,cap_setgid+ep" --keep=1 --user=\$USER --addamb=cap_net_admin  -- -c "$(which bumble-hci-bridge) tcp-client:127.0.0.1:3456 hci-socket:0"
```

```{note}
在配置网桥之前不要启动仿真，否则 BLE 主机堆栈可能会因缺少与 BLE 控制器的连接而超时。
```
