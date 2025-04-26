# 度量分析器

Renode 支持从模拟中收集执行数据，并允许分析执行本身。当前支持的执行指标：

-   executed instructions, 已执行指令
-   memory accesses,  内存访问
-   peripheral accesses,  外围访问
-   exceptions.  异常

## 性能分析

要在 Renode 中启用性能分析，请键入：

```
(monitor) machine EnableProfiler "path_to_dump_file"
```

运行模拟。分析器现在正在从指标中收集数据。完成此步骤后，关闭 Renode。因此，您将获得一个包含收集的指标的转储文件。

可以使用 {rsrc}`metrics_parser Python 库library<tools/metrics_analyzer/metrics_parser/__init__.py>`分析转储，也可以使用提供的帮助程序脚本进行可视化。

## 可视化

要通过与 Renode 捆绑的可视化工具显示所收集数据的图形表示，请执行以下步骤：

### 其他先决条件

要安装指标可视化层的先决条件，请从根 Renode 目录运行以下命令：

```sh
python3 -m pip install --user -r tools/metrics_analyzer/metrics_visualizer/requirements.txt
```

### 运行脚本

运行以下脚本：

```sh
python3 tools/metrics_analyzer/metrics_visualizer/metrics-visualizer.py path_to_dump_file
```

因此，应该会出现一个带有图形的窗口，类似于下面显示的窗口。

![image](img/metrics.png)
