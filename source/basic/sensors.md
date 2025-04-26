# 传感器和虚拟环境

Renode 使您能够纵仿真中传感器的虚拟环境条件，例如 `Temperature` 和 `Humidity`。传感器可以通过直接设置其值来单独控制（在简单的用例中），也可以使用`环境`介质进行全局控制（以实现多个传感器读取一致的虚拟环境条件的场景）。

## 控制单个传感器

要控制 Renode 计算机中的单个传感器，您需要访问传感器可用的适当属性。要更改传感器的 `Temperature` 属性，请使用：

```none
(machine-0) spi0.temperatureSensor Temperature 36.6
```

然后可以通过访问相同的属性来读取此值：

```none
(machine-0) spi0.temperatureSensor Temperature
36.6
```
    
## 全局控制传感器

对于更高级的用例，Renode 提供了 `Environment` 对象，这是一种抽象介质，表示具有物理属性的空间。它允许您对应观察相同环境条件的传感器进行分组和管理。

### 创建环境

要创建名为 `env` 的环境，请执行：

```none
(monitor) emulation CreateEnvironment "env"
```

您可以根据需要创建多个环境。目前 `，Environment` 支持两个参数：temperature 和 pressure。

您可以通过调用来设置环境中的温度：

```none
(monitor) env Temperature 36.6
```

要检查当前设置：

```none
(monitor) env Temperature
36.6
```

### 将传感器添加到环境中

有两种方法可以将传感器添加到环境中：

* 从机器添加单个传感器
* 将计算机添加到环境中

您必须处于 {ref}`计算机上下文中 <machine-context>` 才能执行以下命令。

#### 添加单个传感器

要添加通过 `i2c` 总线连接的 `temperatureSensor` 外设，请运行：

```none
(machine-0) i2c.temperatureSensor SetEnvironment env
```

只有指定的传感器才会被添加到`环境`环境中，并会观察该环境的温度值。

#### 添加机器

要将当前计算机添加到环境中，请运行：

```none
(machine-0) machine SetEnvironment env
```

此命令会将机器及其所有传感器连接到环境`环境` 。它们将具有与环境相同的温度值。

```{note}
将计算机连接到环境时，之前连接到其他环境的所有传感器也将重新连接。
```

每次环境更改时，添加到环境的传感器都会更新。但是，您仍然可以在特定传感器上设置自己的值，并且该值不会传播到其他传感器（直到环境更改覆盖此值）。
