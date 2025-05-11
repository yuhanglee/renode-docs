# 运行您的第一个 Demo

您的 Renode 安装包含许多示例脚本，位于 [scripts](https://github.com/renode/renode/tree/master/scripts)  目录中，（例如，如果您从 Linux 软件包安装，这将位于 `PC 上的 /opt/renode/scripts` 中）。

您可以使用 `include` 或 `start` 命令（简称 `i` 和 `s`）运行这些演示，并将脚本的路径（默认情况下相对于 Renode 安装目录和当前工作目录）作为参数。例如，运行单个节点 STM32F4 Discovery 演示，如下所示：

```none
s @scripts/single-node/stm32f4_discovery.resc
```

记住 `Tab` 自动补全，它会提示你有哪些可用的演示。

演示的二进制文件托管在我们的服务器上，可以通过在加载脚本之前设置 `$bin` 变量（或在脚本中更改其值）来替换为您自己的二进制文件。

您可以自由地将提供的任何演示脚本复制到您的首选目录，并根据需要对其进行修改以满足您的需求，它们应该可以在不同的路径中工作，因为它们通常只使用相对于 Renode 安装目录的路径。

您还可以通过将脚本的路径传递给 `renode` 命令来运行脚本，这将被解释为运行 Renode 并使用 `include @/path/to/script.resc`（请注意，您必须手动启动仿真）。