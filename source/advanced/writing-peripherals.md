# 外设建模指南

Renode 允许用户以多种方式 “建模” 硬件外围设备：

* {rsrc}`SVD 文件中的 automatic 标记 </platforms/cpus/nrf52840.repl#L25>` ，主要用于日志记录目的，
* {rsrc}`带有返回值的 manual 标签 </platforms/cpus/vybrid.repl#L99>`，用于日志记录和简单的流控制，
* 用于实现非常简单[的逻辑的 ](https://github.com/renode/renode/blob/c16c7bceca07734f6f49b4e107d299aa04b8857c//platforms/cpus/tegra3.repl#L131-L134) ，
* 用于实现非常简单 {rsrc}`Python 外围设备 </platforms/cpus/tegra3.repl#L131-L134>` 
* C# 模型，用于描述高级外设逻辑和互连 - 详见下文。

## 如何访问系统总线？

CPU 执行的`读` / `写`作（通常在 `tlib` 子模块的 C 实现中）要么定向到内部内存，要么传递到系统总线并由 C# 级别的框架处理。

对建模为 {risrc}`MappedMemory </src/Emulator/Main/Peripherals/Memory/MappedMemory.cs>`  的内存的访问完全在 C 级别处理，所有其他作都通过 {risrc}`TranslationCPU.Read{Byte,Word,DoubleWord}FromBus </src/Emulator/Peripherals/Peripherals/CPU/TranslationCPU.cs#L578-L615>` /  
{risrc}`TranslationCPU.Write{Byte,Word,DoubleWord}ToBus </src/Emulator/Peripherals/Peripherals/CPU/TranslationCPU.cs#L617-L654>` 函数。

注意：可以将  {risrc}`MappedMemory </src/Emulator/Main/Peripherals/Memory/MappedMemory.cs>`  类型更改为{risrc}`ArrayMemory </src/Emulator/Peripherals/Peripherals/Memory/ArrayMemory.cs>`  来处理 C# 级别的所有内存作。请记住，这可能会导致性能显著下降。此外，无法从 {risrc}`ArrayMemory </src/Emulator/Peripherals/Peripherals/Memory/ArrayMemory.cs>`  执行代码。


    ┌──────────────┐  C to C   ┌─────────────┐
    │ MappedMemory │ ◀──────── │     CPU     │
    └──────────────┘           └─────────────┘
                                 │
                                 │ C to C#
                                 ▼
    ┌──────────────┐           ┌─────────────┐     ┌─────────────┐
    │ Peripheral1  │ ◀──────── │  SystemBus  │ ──▶ │ Peripheral2 │
    └──────────────┘           └─────────────┘     └─────────────┘
                                 │
                                 │
                                 ▼
                               ┌─────────────┐
                               │ ArrayMemory │
                               └─────────────┘

## 如果在给定偏移量处没有映射外设怎么办？

在写入的情况下，该作将被忽略，并在日志中生成一条警告消息。

在 read 的情况下，将返回默认值 0，并在日志中生成一条警告消息。

## 当外设没有实现给定的访问宽度时会发生什么情况？

默认情况下，这种情况被视为在给定偏移处没有 peripheral 映射。

但是，可以使用 {risrc}`AllowedTranslation </src/Emulator/Main/Peripherals/Bus/AllowedTranslationsAttribute.cs>`  属性在外围级别启用访问类型的自动转换 - 请参阅 {risrc}`示例 </src/Emulator/Peripherals/Peripherals/Timers/LiteX_Timer.cs#L18>`. 。
It is, however, possible to enable automatic translation of access type at the peripheral level using the  attribute - see an 

请注意，自动转换可能会在总线上产生更多访问，例如，每 1 次双字读取 4 字节读取，或者每 1 字节写入 1 次双字读取和 1 次双字写入。这可能会对某些 registers 产生意想不到的副作用，例如，自动递增 FIFO data register，发出 “read-to-clear” 行为或其他行为，具体取决于 registers 的语义。开发人员需要验证自动转换在给定 peripheral model 的上下文中是否安全。

## 用 C# 编写外设模型

如果 C# 类实现 {risrc}`IPeripheral </src/Emulator/Main/Peripherals/IPeripheral.cs>` 接口，则将其视为外围模型。

为了使外设可连接到系统总线，它必须至少实现一项（但可以实现一些）：
{risrc}`IBytePeripheral </src/Emulator/Main/Peripherals/Bus/IBytePeripheral.cs>`,
{risrc}`IWordPeripheral </src/Emulator/Main/Peripherals/Bus/IWordPeripheral.cs>`,
{risrc}`IDoubleWordPeripheral </src/Emulator/Main/Peripherals/Bus/IDoubleWordPeripheral.cs>` 接口，分别支持 8 位、16 位和 32 位访问。

双字总线外设必须至少实现三种方法：

* 用于读取（例如，{risrc}`ReadDoubleWord </src/Emulator/Main/Peripherals/Bus/IDoubleWordPeripheral.cs#L13>`  - 由系统总线调用，以便从外设读取值）
* 用于写入（例如，{risrc}`WriteDoubleWord </src/Emulator/Main/Peripherals/Bus/IDoubleWordPeripheral.cs#L14>` - 由系统总线调用，以便将值写入外设）
* 用于重置 （{risrc}`Reset </src/Emulator/Main/Peripherals/IPeripheral.cs#L19>`） - 由框架调用，以将外围设备的状态恢复到初始状态。

尽管从技术上讲可以以任何方式实现`读` / `写`方法，但首选方法是使用 Register Framework（{risrc}`the source code </src/Emulator/Main/Core/Structure/Registers>` ）。有关使用示例，请参阅 {risrc}`the LiteX UART </src/Emulator/Peripherals/Peripherals/UART/LiteX_UART.cs#L20-L52>` 。


您甚至可以使用基类 （） `Basic{Byte,Word,DoubleWord}Peripheral` 来简化代码 - 请参阅 {risrc}`example </src/Emulator/Peripherals/Peripherals/Timers/LiteX_CPUTimer.cs>` 。

以下部分介绍如何使用 Register Framework 设计外设。

## 注册建模指南

创建一个私有枚举，最好命名为 `Registers`，列出 peripheral 支持**的所有** registers。符合枚举命名约定（或使用  {risrc}`RegistersDescription </src/Emulator/Main/Peripherals/Bus/Wrappers/RegisterMapper.cs#L71>` 接口标记它）允许系统总线通过在 `sysbus LogPeripheralAccess` 生成的日志中包含寄存器名称来生成更好的日志消息。

使用人类可读的 PascalCase 编码名称（即 `InterruptEnable` 而不是 `IEN`），即使它们在文档中的引用不同。作为 enum 字段的值，使用从 peripheral 内存空间开头的偏移量（即相对于 peripheral 开头的偏移量， **而不是**绝对地址）。请记住，一个平台可以具有给定类型的多个外围设备。请在这里保持合理 - 有时会有外设具有太多的寄存器或形成  {risrc}`可重复的模式 </src/Emulator/Cores/RiscV/PlatformLevelInterruptController.cs#L446-L469>`  - 在这种情况下，鼓励采用创造性的方法。

不要实现所有 registers - 只**实现**那些软件实际使用的 registers ，因此可以进行测试。

对于每个寄存器 **，列出**所有字段，但仅**实现**必要的字段。未实现的字段应标记为 tags、reserved 或 ignored - 这将有助于生成更好的访问日志。

Registers Framework 中有不同类型的字段可用：


* flags - 单位字段（{risrc}`示例 </src/Emulator/Peripherals/Peripherals/Timers/LiteX_Timer.cs#L124>`），
* enum fields - 单个或多个位字段，其中的位模式对一些非数值进行编码（{risrc}`示例 </src/Emulator/Peripherals/Peripherals/SD/LiteSDCard.cs#L176>`），
* 值字段 - 对数值进行编码的单个或多个位字段（ ({risrc}`示例 </src/Emulator/Peripherals/Peripherals/SD/LiteSDCard.cs#L171>`）。

对于每个字段，您可以选择一种访问模式（默认为 Read&Write），该模式定义允许哪些作以及框架如何处理这些作。可能的基本值（可以与按位 `| 或`运算符）是：

* `Read`,
* `Write`,
* `Set` - writing `1` sets the bit, writing `0` has no effect,
* `Toggle` - writing `1` toggles the current value, writing `0` has no effect [this is most likely usable for flag fields only],
* `WriteOneToClear` - writing `1` clears the bit, writing `0` has no effect [this is most likely usable for fields flag only],
* `WriteZeroToClear` - writing `0` clears the bit, writing `1` has no effect [this is most likely usable for fields flag only],
* `ReadToClear` - the value is set to 0 after read.

将非零值写入只读字段将被忽略，并在日志中生成警告。从只写字段读取将返回默认值 0（但不会在日志中生成警告，因为无法推断读取了哪些字段）。

默认情况下，每个 register 都提供一个 automatic backing field。这意味着软件将读取先前写入的值（假设字段是可写和可读的）。可以访问 backing field 并从代码中修改其值。为此，请使用 `out` 参数 - 请参阅 {risrc}`示例 </src/Emulator/Peripherals/Peripherals/UART/LiteX_UART.cs#L42>`。

有用于生成 registers 组的辅助方法 - 请参阅 {risrc}`DefineMany </src/Emulator/Peripherals/Peripherals/Timers/LiteX_Timer.cs#L59-L67>` 用法示例。



还可以为字段为以下情况附加回调：

* written (with any value) ({risrc}`example </src/Emulator/Peripherals/Peripherals/UART/LiteX_UART.cs#L23>`),
* changed (written with a value different than the current one) ({risrc}`example </src/Emulator/Cores/X86/LAPIC.cs#L163>`),
* read (the value is taken from the backing field),
* read - value provider (the value is generated by the callback itself) ({risrc}`example </src/Emulator/Peripherals/Peripherals/UART/LiteX_UART.cs#L24>`). 

还有整个 register 的回调 - {risrc}`WriteCallback </src/Emulator/Peripherals/Peripherals/I2C/OpenCoresI2C.cs#L63>` 和{risrc}`ReadCallback </src/Emulator/Peripherals/Peripherals/Timers/EFR32_RTCC.cs#L66>`。当回调逻辑需要多个字段的值时，它们非常有用。

## 总线外设大小

在大多数情况下， bus 上 peripheral 的大小是明确定义的，并且可以包含在模型中。为此，该类必须实现 {risrc}`IKnownSize </src/Emulator/Main/Peripherals/IKnownSize.cs>` 接口。在 `Size` 属性中编码的大小以字节表示。

注意： **未**实现 `IKnownSize` 接口的外围设备也可以在 Renode 中使用，但每次 {rsrc}`注册设备 </platforms/cpus/stm32f746.repl#L13>`时都需要提供大小（在 repl 文件中）。

## 测试指南

对于要推送到 Renode 上游存储库的外围设备，需要提供测试，至少执行一个二进制文件。测试外设的首选方法是使用标准测试/样本（如果可用），例如 Zephyr 样本、驱动程序测试等，或提供自定义的特定二进制文件。所有测试二进制文件都应该可以从源构建。

测试用例应在 Robot Framework 中描述。请参阅简单测试 {rsrc}`示例 </tests/platforms>` 

## 外围设备示例

以下是可用作灵感的各种 Renode 外设模型的列表：

* {risrc}`UART </src/Emulator/Peripherals/Peripherals/UART/LiteX_UART.cs>`
* {risrc}`Timer </src/Emulator/Peripherals/Peripherals/Timers/LiteX_Timer.cs>`
* {risrc}`GPIO controller </src/Emulator/Peripherals/Peripherals/GPIOPort/MPFS_GPIO.cs>`
* {risrc}`I2C controller </src/Emulator/Peripherals/Peripherals/I2C/MPFS_I2C.cs>`
* {risrc}`SPI controller </src/Emulator/Peripherals/Peripherals/SPI/MPFS_SPI.cs>`
* {risrc}`I2C sensor </src/Emulator/Peripherals/Peripherals/Sensors/SI70xx.cs>`
* {risrc}`SPI sensor </src/Emulator/Peripherals/Peripherals/Sensors/TI_LM74.cs>`
