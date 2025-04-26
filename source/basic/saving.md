(state-saving)=

# 状态保存和加载

Renode 允许您将仿真的状态保存到文件中。

可以将此类文件传输给其他用户，然后加载以完全重新创建原始设置。不需要额外的二进制文件或配置文件。

要将仿真状态保存到名为 `statefile.save` 的文件中，请运行：

```none
(monitor) Save @statefile.save
```

此文件可与 `Load` 命令一起使用：

```none
(monitor) Load @statefile.save
```

或者，您可以在直接从 CLI 启动 Renode 时加载它：

```sh
$ renode statefile.save
```

请务必记住，在一个版本的 Renode 上创建的状态文件可能与另一个版本不兼容。

请注意，加载状态文件会清除当前仿真，相当于：

```none
(monitor) Clear
(monitor) Load @statefile.save
```

````{note}
加载状态后，您必须手动设置 Monitor 的上下文并重新打开 UART 窗口：

```none
(monitor) mach set 0
(machine-0) showAnalyzer sysbus.uart
```
````

## 测试中的状态保存

It's possible to use the state在定义复杂的 Robot 测试时，可以使用状态保存和加载机制。有关详细信息，请参阅 {ref}`Test cases dependencies <robot-dependencies>`  部分。 

## 加载 gzip 压缩的保存文件

Renode 还支持加载使用 gzip 压缩的快照。

您可以像加载常规保存文件一样加载它们：

```none
(monitor) Load @statefile.save.gz
```

或直接从 CLI 获取：

```sh
$ renode statefile.save.gz
```
