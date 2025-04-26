# Renode 传感器数据格式 （RESD）

有关对模拟传感器进行简单作的介绍和说明，例如将常量值分配为传感器读数，请参阅  {doc}`../basic/sensors` 一章。

## 什么是 Renode 传感器数据格式

Renode 传感器数据 （RESD） 是一种统一且可移植的方式，可为 Renode 中实现的传感器模型提供样本数据。`.resd` 文件中描述的传感器数据对所有数据类型使用标准单位和固定格式，因此它们可以被任何现有或未来的模型使用。

存储在 `.resd` 文件中的数据被划分为多个独立的通道，每个通道具有给定的类型（例如温度或加速度）。这允许将单个输入文件用于多个传感器，例如 IMU。也可以包含多个相同类型的通道（每个通道由唯一的通道 ID 标识），例如两个用于温度读数的通道。

每个 `.resd` 文件都包含一个文件头，后跟一个数量可变的数据块：

```
00000000  52 45 53 44 01 00 00 00  02 01 00 00 00 a8 01 00  |RESD............|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 70 c9  |..............p.|
00000020  b2 8b 00 00 00 00 00 00  00 00 00 00 00 cc 9b 03  |................|
00000030  00 c4 95 03 00 79 9b 03  00 18 9a 03 00 75 a3 03  |.....y.......u..|
00000040  00 3b a1 03 00 b1 b4 03  00 8d 9d 03 00 41 a5 03  |.;...........A..|
00000050  00 4d 9b 03 00 21 af 03  00 6f 96 03 00 20 ba 03  |.M...!...o... ..|
```

您可以使用 [CSV - RESD 解析器](csv-resd)从 CSV 文件轻松创建如上所示的二进制文件。

文件头由一个以 ASCII 编码的 4 字节魔术字符串值 `RESD` 组成，后跟一个定义文件使用的格式版本的字节（上面的代码使用版本 0x1），后跟 3 个字节的保留填充（当前定义为全零）。

## RESD 数据块

每个区块头的结构如下：

```{csv-table} Block header structure
:header-rows: 1
:delim: "|"

Bytes | 0          | 1 - 2       | 3 - 4      | 5 - 8
Name  | Block type | Sample type | Channel id | Data size
```

### RESD 区块类型

目前，您可以在 `.resd` 文件中使用以下块类型：

```{csv-table} Block types
:header-rows: 1
:delim: "|"

ID  | Block type
0x0 | Reserved
0x1 | [Arbitrary Timestamp Sample Blocks](arbitrary-timestamp-sample-blocks)
0x2 | [Constant Frequency Sample Blocks](constant-frequency-sample-blocks)
```

(arbitrary-timestamp-sample-blocks)=

#### 任意时间戳示例块

任意时间戳样本块包含一系列样本，每个样本都有自己的[时间戳](timestamps) 。在这种类型的样本模块中，您不指定样本之间的时间段，而是为每个样本提供特定的时间戳，以模拟不规则的传感器读数。

(constant-frequency-sample-blocks)=

#### 恒频采样块

在恒定频率块中，块头比任意时间戳块长 16 字节，并包含一个额外的子头，该子头提供有关序列中第一个样本的时间戳和连续样本之间的时间段的信息。

```{note}
如果您的数据块包含超过 1 个样本，则 period 值 `0` 无效。
```

(timestamps)=

#### 时间戳

`.resd` 中的时间戳编码为无符号的 8 字节值，以虚拟纳秒表示，从文件开头开始计数。

```{note}
`.resd` 解析器可以选择支持应用于文件中的所有样本的全局时间戳加数（例如，允许在虚拟时间的不同时刻加载同一输入文件两次）。
```

### RESD 样本类型

您的 `.resd` 文件可以包含以下样本类型：

```{csv-table} Sample types
:header-rows: 1
:delim: "|"

ID              | Sample Type           | Sample Unit
0x0000          | Reserved              | N/A
0x0001          | Temperature           | signed 4-byte value in millidegrees (10^-3) Celsius
0x0002          | Acceleration          | set of 3 signed 4-byte values in micro g (10^-6)<br> mapped to X, Y, Z dimensions
0x0003          | Angular rate          | set of 3 signed 4-byte values in tens of microradians<br> (10^-5)  per second mapped to X, Y, Z dimensions
0x0004          | Voltage               | unsigned 4-byte value in microvolts (10^-6) 
0x0005          | ECG                   | signed 4-byte value in nanovolts (10^-9) 
0x0006          | Humidity              | unsigned 4-byte value in per cent mille (PCM or 1 thousandth of a percent) of relative humidity
0x0007          | Pressure              | unsigned 8-byte value in milliPascals (10^-3)
0x0008          | Magnetic Flux Density | set of 3 signed 4-byte values in nanoteslas (10^-9) mapped to X, Y, Z dimensions
0xF000 - 0xFFFF | Custom                | defined by model-specific input
```

请记住，除 Custom 以外的样本类型不使用[元数据](metadata)字典（元数据大小设置为 0）。

```{note}
使用自定义传感器数据类型会使输入数据与特定传感器实现紧密耦合，因此，它可能与其他传感器不兼容。
```

(metadata)=

#### 元数据

元数据是一个二进制编码的字典，其中每个条目都由键名、键类型和值组成。前 8 个字节表示元数据部分的大小，可以设置为 0 以指示给定块不包含其他元数据。

键名称是由 [a-z0-9_] 个字符组成的以 null 结尾的字符串。它后跟一个字节，用于描述给定键的值类型：

```{csv-table} Metadata Types
:header-rows: 1
:delim: "|"

ID   | Metadata type
0x00 | Reserved
0x01 | int8
0x02 | uint8
0x03 | int16
0x04 | uint16
0x05 | int32
0x06 | uint32
0x07 | int64
0x08 | uint64
0x09 | float
0x0A | double
0x0B | string (null-terminated)
0x0C | blob
```

```{note}
在 blob 类型元数据中，前 4 个字节将 blob 内容长度重置编码为无符号整数。
```

例如，由两个条目组成的元数据部分的编码：字符串类型的描述和 blob 类型的数据将如下所示：

```{csv-table} Example metadata string and blob
:header-rows: 1
:delim: "|"

Bytes   | Name             | Value 
0 - 7   | Size             | 78
8 - 19  | Key #1 (string)  | Description\0
20      | Type #1 (string) | 0x0B
21 - 59 | Value #1         | This is a very important sample stream\0
60 - 64 | Key #3 (string)  | Data\0
65      | Type #3 (blob)   | 0x0C
66 - 69 | Blob size        | 0x5
70 - 74 | Blob content     | 0xDE 0xAD 0xC0 0xFF 0xEE
```

### 自定义示例数据示例 - MAX86171 AFE

以下说明显示了如何使用 RESD 创建基于真实传感器 [MAX86171 AFE](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX86171.pdf) 的自定义示例数据示例。

MAX86171 AFE 传感器进行的测量取决于通道配置，例如 LED 曝光驱动电流值直接影响光电二极管输出。因此，每个 MAX86171 AFE 示例数据块都以包含以下信息字典的元数据部分开头：

```{csv-table} Metadata string and blob example
:header-rows: 1
:delim: "|"

Key name          | Type   | Description
led_a_exposure    | uint16 | LED A exposure drive current in mA
led_a_source      | uint8  | LED A source
led_b_exposure    | uint16 | LED B exposure drive current in mA
led_b_source      | uint8  | LED B source
led_c_exposure    | uint16 | LED C exposure drive current in mA
led_c_source      | uint8  | LED C source
pd_a_source_flags | uint8  | PD A source/flags
pd_a_adc_range    | uint32 | PD A ADC range
pd_a_dac_offset   | int16  | PD A DAC offset
pd_b_source_flags | uint8  | PD B source/flags
pd_b_adc_range    | uint32 | PD B ADC range
pd_b_dac_offset   | int16  | PD B DAC offset
```

#### MAX86171 AFE 示例数据

单个 MAX86171 AFE 样本被描述为一个单字节的无符号值，其中包含多个活动通道，后跟一个测量帧列表，每个测量帧编码为一个四字节的无符号值：

```{csv-table} MAX86171 AFE sample data structure
:header-rows: 1
:delim: "|"

Bytes            | 0                | 1 - 4    | 5 - 8    | (8N + 1) - (8N + 4)
Values [raw AFE] | Number of frames | Frame #0 | Frame #1 | Frame #N
```

RESD 中的 AFE 样本对应于 MAX86171 AFE 生成的单个帧：每次有效测量都需要两个标记样本，如 MAX86171 数据表的 FIFO 描述部分所述：

```{csv-table} MAX86171 AFE sample details
:header-rows: 1
:delim: "|"

Bits   | 31..24   | 23..20 | 19..0
Values | Reserved | Tag    | Value
```

tag 部分遵循传感器数据表的 FIFO Description 部分的规范：

```{csv-table} MAX86171 AFE sample details
:header-rows: 1
:delim: "|"

Tag  | Description
0x00 | Reserved
0x01 | Measurement 1 Data
0x02 | Measurement 2 Data
0x03 | Measurement 3 Data
0x04 | Measurement 4 Data
0x05 | Measurement 5 Data
0x06 | Measurement 6 Data
0x07 | Measurement 7 Data
0x08 | Measurement 8 Data
0x09 | Measurement 9 Data
0x0A | Dark Data
0x0B | ALC Overflow Event
0x0C | Exposure Overflow Event
0x0D | Picket Fence Event
0x0E | Invalid Data
0x0F | Reserved
```

数据为 2 测量 OC 通道 1 的样本将如下所示：

```{csv-table} MAX86171 AFE sample details
:header-rows: 1
:delim: "|"

Bytes  | 0          | 1 - 4                   | 5 - 8
Names  | No. frames | Frame 1 (Measurement 1) | Frame 2 (Measurement 1)
Values | 0x2        | 0x00 0x01 0x12 0x34     | 0x00 0x01 0x12 0x34
```

这些帧可以解码为：

```{csv-table} MAX86171 AFE decoded frames
:header-rows: 1
:delim: "|"

Bits   | 31..24   | 23..20            | 19..0
Names  | Reserved | Measurement 1 tag | Measurement value
Values | 0x000    | 0x1               | 0x1234
```

(csv-resd)=

### CSV - RESD 解析器使用

[CSV - RESD 解析器](https://github.com/renode/renode/tree/master/tools/csv2resd)是 Renode 存储库中的一个工具，允许您将 CSV 文件转换为 RESD 文件格式。

要使用该工具，请遵循以下语法：

```none
./csv2resd.py [GROUP]
GROUP ::= -i <csv-file> [-m <type>:<field(s)>:<target(s)>*:<channel>*]
          -s <start-time> -f <frequency> -t <timestamp> -o <offset> -c <count>
```

语法允许多个组规范，其中 –input 是组之间的分隔符。您可以为每个 `--input` 指定多个映射 （`-map`）。

`--map` 中的 `*` 表示给定的属性是可选的。要使 `--map` 参数正确，必须采用以下方式之一进行结构构建：

* `--map <type>:<field(s)>`
* `--map <type>:<field(s)>:<target(s)>`
* `--map <type>:<field(s)>:<target(s)>:<channel>`
* `--map <type>:<field>::<channel>`

有关更多信息，请参见 `--help`。

如果要从文件 `first.csv` 中提取 `temp1` 和 `temp2` 列，从文件 `second.csv` 中提取前 3 个样本 `temp`，然后将它们分别映射到 RESD 中的温度通道 `0`、`1` 和 `2`，则可以使用以下参数运行脚本：

```none
./csv2resd.py \
    -i first.csv \
        -m temperature:temp1::0 \
        -m temperature:temp2::1 \
        -s 0 \
        -f 1 \
    -i second.csv \
        -m temperature:temp::2 \
        -s 0 \
        -f 1 \
        -c 3 \
    output.resd
```

### RESD 内省

Renode 附带一个用于检查 RESD 文件的命令，而无需将它们加载到任何特定的传感器型号。

要分析 RESD 文件的内容，请首先使用以下命令加载它：

```none
(monitor) resd load r1 @my_samples.resd
RESD file from 'my_samples.resd' loaded under identifier 'r1'
```

现在，您可以使用以下方法打印有关示例模块的信息：

```none
(monitor) resd list-blocks r1
Blocks in r1:
1. [00:00:00.000000..00:00:02.000000] Acceleration:0
2. [00:00:02.000000..00:01:05.500000] Acceleration:0
(monitor) resd describe-block r1 1
Index: 1
Sample type: Acceleration
Channel ID: 0
Start Time: 00:00:00.000000
End Time: 00:00:02.000000
Duration: 00:00:02.000000
Samples count: 2
Period: 00:00:01.000000
Frequency: 1Hz
```

您还可以使用以下方法转储选定的样本（例如，在 2.2 秒和 2.3 秒的时间戳之间）：

```none
(monitor) resd get-samples-range r1 2 "2.2" "2.3"
00:00:02.203125: [0, 0, 0.1] g
00:00:02.218750: [0, 0, 0.2] g
00:00:02.234375: [0, 0, 0.3] g
00:00:02.250000: [0, 0, 0.2] g
00:00:02.265625: [0, 0, 0.2] g
00:00:02.281250: [0, 0, 0.1] g
00:00:02.296875: [0, 0, 0.0] g
```

有关更多详细信息，请参阅命令的帮助输出：

```none
(monitor) help resd
resd
introspection for RESD files

You can use the following commands:
'resd load NAME PATH'   loads RESD file under identifier NAME
'resd unload NAME'      unloads RESD file with identifier NAME
'resd list-blocks NAME' list data blocks from RESD file with identifier NAME
'resd describe-block NAME INDEX'        show informations about INDEXth block from RESD with identifier NAME
'resd get-samples NAME INDEX "START_TIME" COUNT'        lists COUNT samples starting at START_TIME from INDEXth block of RESD with identifier NAME
'resd get-samples-range NAME INDEX "START_TIME" "DURATION"'     lists DURATION samples starting at START_TIME from INDEXth block of RESD with identifier NAME
'resd get-samples-range NAME INDEX "START_TIME..END_TIME"'      lists samples between START_TIME and END_TIME from INDEXth block of RESD with identifier NAME
'resd get-prop NAME INDEX PROP' read property PROP from INDEXth block of RESD with identifier NAME
  possible values for PROP are: SampleType, ChannelID, StartTime, EndTime, Duration, SamplesCount
```

