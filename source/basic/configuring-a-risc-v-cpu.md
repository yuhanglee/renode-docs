# 配置 RISC-V CPU

在 Renode 支持的架构中，RISC-V 是最突出的架构之一。Renode 支持 RISC-V 的 32 位和 64 位版本、各种版本的特权架构和广泛的扩展。

Renode 支持的最常见的基本 ISA 集是：

- RV32/64I Base (`I`)
- Integer Multiplication and Division (`M`)
- Atomic Instructions (`A`)
- Single-Precision Floating-Point (`F`)
- Double-Precision Floating-Point (`D`)
- Control and Status Register Instructions (`Zicsr`)
- Instruction-Fetch Fence (`Zifencei`)

以上所有内容都构成了 `G` 集。

Renode 还支持许多 ISA 扩展，例如：

- Compressed Instructions (`C`)
- Vector Operations (`V`)
- Bit-manipulation
  - Address generation instructions (`Zba`)
  - Basic bit-manipulation (`Zbb`)
  - Carry-less multiplication (`Zbc`)
  - Single-bit instructions (`Zbs`)
- Half-Precision Floating-Point (`Zfh`)
- Atomic Compare-and-Swap (CAS) (`Zacas`)
- PMP Enhancements for memory access and execution prevention in Machine mode (`Smepmp`)

最后，Renode 还支持自定义的非标准指令集。其中一些可以像其他指令集（例如 `Xandes`）一样进行选择，其他则可以直接在各自的核心类（例如 `CV32E40P`）中定义。

要在仿真中定义 RISC-V 内核，必须编辑 Renode 平台文件 （.repl） 并为 CPU 添加节点。

## 选择特定的 ISA 变体

首先添加 `CPU。RiscV32` 或 `CPU。RiscV64` 复制到 .repl 文件：
```
cpu: CPU.RiscV32 @ sysbus
```
然后使用 `cpuType` 参数指定 ISA 和所需的扩展。

根据架构宽度，从 “rv32” 或 “rv64” 开始，然后是已启用的 ISA 集列表。名称长度超过一个字符的扩展必须用下划线分隔。

例如：
```
cpu: CPU.RiscV32 @ sysbus
  cpuType: "rv32imaf_zicsr_zifencei"
```
它表示具有基本指令集 （I）、整数乘法和除法 （M）、原子指令 （A）、单精度浮点 （F）、CSR 指令 （Zicsr） 和指令获取栅栏 （Zifencei） 的 32 位 RISC-V。

## 自定义 CPU

在 .repl 文件中创建 CPU 时，您可以向 CPU 传递其他参数。所有这些都是可选的。

- `timeProvider` - 设置要用作 CPU 时间提供程序的外围设备，用于填充`时间` CSR。通常，您将提供 `clint` 中断控制器的实例
- `privilegedArchitecture` - 选择 CPU 应遵循的特权架构版本。默认值为 1.11。可用值为：
  - `PrivilegedArchitecture.Priv1_09`
  - `PrivilegedArchitecture.Priv1_10`
  - `PrivilegedArchitecture.Priv1_11`
  - `PrivilegedArchitecture.Priv1_12` - 目前对特权架构 v1.12 的支持是实验性的，并非所有支持都已实现。
- `endianness` - 指定 CPU 的字节序，默认为 little endian
- `nmiVectorAddress` 和 `nmiVectorLength` - 允许自定义不可屏蔽的中断向量（如果 CPU 支持）
- `allowUnalignedAccesses` - 定义每当软件对内存执行非对齐访问时是否应引发异常。默认为 `false`
- `interruptMode` - 允许您强制执行中断处理模式。默认为 auto。可用模式：
  - Auto （0） - 检查 `mtvec` 的 LSB 以检测模式
  - 直接 （1） - 所有异常都将 `PC` 设置为 `mtvec` 的 `BASE` 值
  - 矢量 （2） - 异步中断将 `PC` 设置为 `mtvec` 的 `BASE + 4 * 原因`
- `privilegeLevels` - 指定 CPU 的已实施权限级别。默认值为 Machine、Supervisor 和 User 模式。可用值为：
  - `PrivilegeLevels.MachineUser`
  - `PrivilegeLevels.MachineSupervisorUser`

## 添加自定义 RISC-V 指令

RISC-V 最重要的特点之一是它的可定制性，也带有非标准指令。

有几种方法可以将自定义指令添加到 Renode 中的 RISC-V CPU。

它们都围绕着两个论点
- Pattern - 一个位模式，指定要匹配哪些指令来执行您的自定义处理程序。字符 `1` 和 `0` 表示必须在该位置设置哪个位，而任何其他字符表示“任何值”。模式的长度必须为 64、32 或 16 个字符。
- Handler - 模式匹配时要执行的代码

### Python

您可以使用 {ref}`Python script <python-riscv>`通过 RISC-V CPU 上的 `InstallCustomInstructionHandlerFromString` or `InstallCustomInstructionHandlerFromFile` 方法处理自定义指令。

 例如：
```
sysbus.cpu InstallCustomInstructionHandlerFromString "10110011100011110000111110000010" "cpu.DebugLog('custom instruction executed!')"
```

Python 脚本具有 `instruction 变量 available`，其中包含所调用指令的作码。

### C#

您可以通过 `InstallCustomInstruction` 方法使用 C# 函数来处理自定义指令。它采用相同的 pattern 参数，但 handler 参数是 `Action<UInt64>`

```csharp
public class MyCustomRiscV : RiscV32
{
    public MyCustomRiscV(IMachine machine, IRiscVTimeProvider timeProvider = null, uint hartId = 0,
                    PrivilegedArchitecture privilegedArchitecture = PrivilegedArchitecture.Priv1_11,
                    Endianess endianness = Endianess.LittleEndian, string cpuType = "rv32imfc_zicsr_zifencei")
      : base(machine, cpuType, timeProvider, hartId, privilegedArchitecture, endianness, allowUnalignedAccesses : true)
    {
      // As mentioned, the pattern arguments takes a 16, 32, or 64 character long string.
      // Characters 0 and 1 specify the bits that should be set,
      // while all other characters mean that given bit can be either 1 or 0.
      // In this case we use F for the Imm field, B for rs1, and D for rD to make the pattern more readable
      //
      // FFFFFFFFFFFFBBBBB100DDDDD0001011
      //      |        |       +-- [ 7:11] - rD
      //      |        +---------- [15:19] - rs1
      //      +------------------- [20:31] - Imm
      //
      // We could've used "-----------------100-----0001011" as the pattern and it'd mean the same thing,
      // but suddenly it's not so clear where the different fields are inside the opcode we're matching.
      InstallCustomInstruction(pattern: "FFFFFFFFFFFFBBBBB100DDDDD0001011", handler: opcode =>
      {
        this.Log(LogLevel.Noisy, "(p.lbu rD, Imm(rs1!)) at PC={0:X}", PC.RawValue);
        // rD = Zext(Mem8(rs1))
        var rD = (int)BitHelper.GetValue(opcode, 7, 5); // Extract value from opcode starting at bit 7 and length of 5
        var rs1 = (int)BitHelper.GetValue(opcode, 15, 5); // Extract value from opcode starting at bit 15 and length of 5
        var rs1Value = (long)GetRegisterUnsafe(rs1).RawValue;
        SetRegisterUnsafe(rD, ReadByteFromBus((ulong)rs1Value));

        // rs1 += Imm[11:0]
        var imm = (int)BitHelper.SignExtend((uint)BitHelper.GetValue(opcode, 20, 12), 12); // Extract value from opcode starting at bit 20 and length of 12
                                                                                           // and sign-extend it to full int
        SetRegisterUnsafe(rs1, (ulong)(rs1Value + imm));
      });
    }
}
```


### 经过验证的自定义函数单元

您最多可以将 4 个 CFU 连接到在 Renode 中模拟的每个 RISC-V 内核。

使用协同仿真库编译 CFU 后（请参阅： [示例 CFU 项目](https://github.com/antmicro/renode-verilator-integration/blob/master/samples/cfu_mnv2/README.md) ），您可以将 CFU 附加到 CPU。

为此，请将此行添加到您的 .repl 中：
```
cfu0: CoSimulated.CoSimulatedCFU @ cpu 0
```
并将这一行添加到您的 .resc 中：
```
cpu.cfu0 SimulationFilePathLinux @<PATH_TO_COMPILED_CFU_BINARY>
```

如果您使用的是 Windows 而不是 Linux，则必须在 macOS 上使用 `SimulationFilePathWindows` 和 `SimulationFilePathMacOS`。然后，引用您的 CFU 的说明将被转发到 Verilated CFU。

所有 CFU 指令都遵循以下模式： `FFFFFFFAAAAABBBBBIIICCCCCNN01011`
- `N` - 将指令调用转发到的 CFU 编号
- `C` - 将放置结果值的寄存器
- `I`, `F` - 函数 ID。传递给 CFU 的最终 ID 是 `FFFFFFFIII`
- `B` - 源寄存器 1。其值将被读取并传递到 CFU 中
- `A` - 源寄存器 2。同上

## 添加自定义 RISC-V 控制和状态寄存器

与自定义指令类似，您还可以定义自定义 CSR。

### Python

要使用 {ref}`Python script <python-riscv>` 处理自定义 CSR，您可以使用 `RegisterCSRHandlerFromString` 和 `RegisterCSRHandlerFromFile`。第一个参数是 CSR ID，而第二个参数是指处理程序脚本。

该脚本将传递一个`请求`变量，该变量包含字段 `isRead`、`isWrite` 和 `value`。

```python
if request.isRead: # If the CPU tries to read your CSR, isRead will be True
    cpu.DebugLog('CSR read!')
elif request.isWrite: # Otherwise isWrite will be True and value will contain the value being written
    cpu.DebugLog('CSR written: {}!'.format(hex(request.value)))
```

### C#

要使用 C# 函数处理自定义 CSR，您可以使用 `RegisterCSR` 方法。同样，它将 ID 作为第一个参数，但随后按此顺序分别获取读取处理程序和写入处理程序。

让我们采用之前定义的自定义 RISC-V CPU 并向其添加自定义 CSR，每次读取时都会返回一个随机值。

```csharp
public class MyCustomRiscV : RiscV32
{
    public MyCustomRiscV(IMachine machine, IRiscVTimeProvider timeProvider = null, uint hartId = 0,
                    PrivilegedArchitecture privilegedArchitecture = PrivilegedArchitecture.Priv1_11,
                    Endianess endianness = Endianess.LittleEndian, string cpuType = "rv32imfc_zicsr_zifencei")
      : base(machine, cpuType, timeProvider, hartId, privilegedArchitecture, endianness, allowUnalignedAccesses : true)
    {
      this.random = EmulationManager.Instance.CurrentEmulation.RandomGenerator;

      InstallCustomInstruction(pattern: "FFFFFFFFFFFFBBBBB100DDDDD0001011", handler: opcode =>
      {
        // Custom Instruction implementation from before
      });

      RegisterCSR(
        (ulong)CustomCSR.Rnd,
        () => (ulong)random.Next(),           // Read handler
        value => { /* Ignore all writes */ }, // Write handler
        name: "RND"                           // Optionally add a name of your CSR
      );
    }

    private readonly PseudorandomNumberGenerator random;

    // Creating an enum isn't required, but it's cleaner
    // and allows to name otherwise arbitrary CSR IDs
    private enum CustomCSR : ulong
    {
      Rnd = 0xfc0,
    }
}
```
