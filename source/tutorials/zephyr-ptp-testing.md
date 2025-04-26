# 测试 Zephyr PTP 支持

本教程将指导您如何使用 Renode 运行一组测试，以验证 Zephyr 的 [TSN/PTP 支持](https://en.wikipedia.org/wiki/Precision_Time_Protocol) 。

## 先决条件

要开始测试，您需要根据 {doc}`../advanced/building_from_sources` 下载并构建 Renode。

该测试套件使用 [Robot Framework](https://robotframework.org/)，可以使用单个脚本运行。

要创建您自己的 Zephyr 二进制文件进行测试，您需要遵循 [Zephyr 的入门指南](https://docs.zephyrproject.org/latest/getting_started/index.html) 。

这些测试需要两个从 `zephyr/samples/net/gptp` 示例构建的 Zephyr ELF 文件，以 `sam_e70_xplained` 板为目标。

要构建它们，请创建两个覆盖文件，一个用于 Grand Master 节点，一个用于 slave 节点。

Grand Master 的配置 （gm.conf）：

    CONFIG_NET_GPTP_GM_CAPABLE=y
    CONFIG_ETH_SAM_GMAC_RANDOM_MAC=y
    
    CONFIG_NET_CONFIG_MY_IPV4_ADDR="192.0.2.1"
    CONFIG_NET_CONFIG_MY_IPV6_ADDR="2001:db8::1"
    CONFIG_NET_GPTP_NEIGHBOR_PROP_DELAY_THR=200000

从属节点 （slave.conf） 的配置：

    CONFIG_NET_GPTP_GM_CAPABLE=n
    CONFIG_ETH_SAM_GMAC_RANDOM_MAC=y
    
    CONFIG_NET_CONFIG_MY_IPV4_ADDR="192.0.2.2"
    CONFIG_NET_CONFIG_MY_IPV6_ADDR="2001:db8::2"
    CONFIG_NET_GPTP_NEIGHBOR_PROP_DELAY_THR=200000

按照 Zephyr 文档构建这些示例。例如，要构建 Grand Master 应用程序，请运行：

    west build --board sam_e70_xplained -- -DOVERLAY_CONFIG=gm.conf

## 测试套件

该套件执行在两个 SAM E70 节点上运行的一组测试，这些节点通过以太网连接。将实施以下测试：

* 节点应发送 PDelay 请求数据包
* 从属节点应通过调用特定的 Zephyr 回调来接受 Grand Master 节点
* 主节点应发送带有预期参数的 Announce 数据包
* 主节点应发送格式正确的 Sync 和 Sync Follow Up 数据包
* 主节点应以有效的间隔发送 Sync 数据包
* 从 节点 应将其时钟同步到 主节点

### 运行测试

要开始测试，只需从 Renode 根目录运行以下命令：

    renode-test tests/platforms/SAME70.robot

这将运行整套测试。测试完成后，结果将存储在 `output/tests/report.html` 中。

要切换这些测试中使用的二进制文件，请编辑提供的 `.robot` 文件。或者，如果您不想进行任何更改，可以使用 `--variable` 开关来指定要使用的文件：

    renode-test --variable ZEPHYR_MASTER_ELF:path/to/zephyr.elf --variable ZEPHYR_SLAVE_ELF:path/to/another/zephyr.elf tests/platforms/SAME70.robot
