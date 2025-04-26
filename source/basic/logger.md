(using-logger)=

# 使用 Logger

启动 Renode 后出现的第一个窗口专用于 Logger。

您可以使用许多日志记录选项来改善对所显示信息的体验。

(log-level)=

## 日志记录级别

有五个可用的日志记录级别：

* NOISY (-1),
* DEBUG (0),
* INFO (1),
* WARNING (2),
* ERROR (3).

您可以使用 `logLevel` 命令选择要记录的消息。每个仿真对象都可以单独配置。此外，每个 logger 后端（例如 {ref}`log file <log-file>` ）都可以有自己的配置。


默认情况下，将记录来自除 NOISY 级别之外的所有级别的消息。

要将全局日志级别设置为 NOISY，请键入：

```none
(machine-0) logLevel -1
```

要仅更改所选外设（在本例中为 UART 设备）的日志级别，请键入：

```none
(machine-0) logLevel -1 sysbus.uart
```

```{note}
增加记录的消息数可能会影响仿真的性能。
```

可以通过不带参数的 `logLevel` 来验证当前日志级别。

这是此命令在一些配置后的输出：

```none
(machine-0) logLevel
Currently set levels:
Backend           | Emulation element               | Level
=================================================================
console           :                                     : DEBUG
                  : machine-0:sysbus.plic               : ERROR
                  : machine-0:sysbus.uart               : NOISY
-----------------------------------------------------------------
```

(log-file)=

## 记录到文件

要分析长时间运行的仿真的输出，最好将日志重定向到文件。

为此，请使用 `logFile` 命令：

```none
(machine-0) logFile @some_file_name
```

这不会禁用控制台记录器，但会添加一个新的 sink，以便单独配置。从性能的角度来看，根据方案，提高最低控制台日志级别并在日志文件中保留更详细的数据可能是有益的。

要为文件后端设置 ERROR 日志级别，请键入：

```none
(machine-0) logLevel 2 file
```

外围设备也可以在不同的后端具有不同的日志级别：

```none
(machine-0) logLevel 1 file sysbus.uart
```

## 记录对外围设备的访问

除了标准的 logger 配置外，您还可以启用对特定外设的访问日志记录。此功能仅对在 System Bus 上注册的外围设备启用。

要启用它，请运行：

```none
(machine-0) sysbus LogPeripheralAccess sysbus.uart
```

现在，每当 CPU 尝试读取或写入此外围设备时，您都会看到类似于此的消息：

```none
14:32:28.6083 [INFO] uart: ReadByte from 0x0 (TransmitData), returned 0x0.
```

要启用对所有外围设备的日志记录访问，请运行：

```none
(machine-0) sysbus LogAllPeripheralsAccess true
```

### 创建执行跟踪

可以创建二进制文件执行的每个函数的跟踪：

```none
(machine-0) sysbus.cpu LogFunctionNames true
```

因此，函数的名称将打印到 `INFO` 级别的日志中：

```
17:05:23.8834 [INFO] cpu: Entering function kobject_uevent_env at 0xC014CD9C
17:05:23.8834 [INFO] cpu: Entering function dev_uevent_name (entry) at 0xC018FA5C
17:05:23.8834 [INFO] cpu: Entering function dev_uevent_name at 0xC018FA70
17:05:23.8834 [INFO] cpu: Entering function kobject_uevent_env at 0xC014CDA8
17:05:23.8835 [INFO] cpu: Entering function kobject_uevent_env at 0xC014CDB8
17:05:23.8835 [INFO] cpu: Entering function kmem_cache_alloc (entry) at 0xC0085610
17:05:23.8835 [INFO] cpu: Entering function kmem_cache_alloc at 0xC0085630
```

如果您只对函数的子集感兴趣，则可以通过提供以空格分隔的名称前缀来限制结果：

```none
(machine-0) sysbus.cpu LogFunctionNames true "dev kobject"
```

您还可以通过添加另一个 `true` 参数来避免记录后续的重复函数名称，而不是可选的函数名称前缀：

```none
(machine-0) sysbus.cpu LogFunctionNames true ["dev kobject"] true
```

在上述示例中，只有这三行将保持打印状态，同时应用函数名称筛选和重复删除：

```
17:05:23.8834 [INFO] cpu: Entering function kobject_uevent_env at 0xC014CD9C
17:05:23.8834 [INFO] cpu: Entering function dev_uevent_name (entry) at 0xC018FA5C
17:05:23.8834 [INFO] cpu: Entering function kobject_uevent_env at 0xC014CDA8
```

## 隐藏过多的未处理访问日志

默认情况下，Renode 会通知您对任何模型未涵盖的内存范围的未处理访问。您可能会看到如下日志：

```
    09:21:8.1960 [WARNING] sysbus: [cpu: 0x08001200] WriteDoubleWord to non existing peripheral at 0x400D0114, value 0xFFFFFFFF.
    09:21:9.4538 [WARNING] sysbus: [cpu: 0x080012E6] ReadDoubleWord from non existing peripheral at 0x400D0118, returning 0x0.
```

这些日志用于通知您平台的描述不完整，如果您发现模拟存在问题，则可能是可能的情况之一。

通常，这些未处理的区域不会影响执行的任何重要方面，您可能希望将这些日志静音。

虽然将 {ref}`log level <log-level>` 更改为 `ERROR` 以隐藏警告可能是一种选择，但这可能是一个过于激进的解决方案，因为您可能希望继续看到其他警告。

在这种情况下，实现细粒度日志记录控制的最佳方法是使用 `SilenceRange` 功能。例如，如果要禁用 `0x80000` 到 `0x801000` 范围内地址的日志记录，请运行：

```none
    sysbus SilenceRange <0x80000 0x1000>
```

您也可以从 `sysbus` init 部分的 REPL 级别执行此作：

```none
    sysbus:
        init:
            SilenceRange <0x80000 0x1000>
```

