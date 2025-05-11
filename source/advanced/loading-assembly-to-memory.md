# 将 assembly 加载到内存中

Renode 允许用户以多种方式加载汇编代码：

* [LLVMAssembler](https://github.com/renode/renode/blob/791eec59cc66650893c0ff5dd3dc2909b7229bd5/tests/unit-tests/arm-ge-flag.robot#L25) 主要用于测试
* {rsrc}`直接编写 <tests/unit-tests/riscv-custom-instructions.robot#L140>` 可用于自定义指令的十六进制指令，
* 直接加载编译后的 ASM 代码，如下所述

```{note}
在继续之前，建议阅读 [使用 GDB 进行调试](https://renode-docs-chinese.readthedocs.io/en/latest/debugging/gdb.html) 和[使用 VSCode 进行调试](https://renode-docs-chinese.readthedocs.io/en/latest/debugging/vscode.html)章节。
```

## 先决条件

您需要拥有目标架构的 GNU 工具链，例如 [RISC-V](https://github.com/riscv-collab/riscv-gnu-toolchain) 或 [ARM](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain)。我们将使用 `riscv64-unknown-elf-*` 实用程序（对于 ARM，您将使用 `arm-none-eabi-*`）。

## 将汇编代码编译为可执行文件

让我们从简单的 RISC-V 程序集开始，该程序集以 `0x00064000` 的速度存储和加载内存：

```asm
.globl _start
_start:
    li a0, 0x00064000
    li a2, 0x5
    sw zero, (a0)
    lw a4, (a0)
    sw a2, (a0)
    lw a4, (a0)
loop:
    j loop
```

要编译代码，请使用 GNU 工具链的 `as` 和 `ld`：

```
riscv64-unknown-elf-as -march=rv64imac_zba_zbb asm.s -o asm.o
riscv64-unknown-elf-ld asm.o -o asm.elf
```

让我们将这个简单的 asm.resc 文件与 platform 一起使用：
```
using sysbus
machine LoadPlatformDescriptionFromString
"""
cpu: CPU.RiscV64 @ sysbus
    cpuType: "rv64imac_zba_zbb"
    privilegedArchitecture: PrivilegedArchitecture.Priv1_10
    timeProvider: empty

dram: Memory.MappedMemory @ sysbus 0x00
    size: 0x06400000
"""
machine StartGdbServer 3333
sysbus LoadELF @asm.elf
```

请注意，我们需要平台中的 CPU 类型和模型来匹配编译的 ASM 代码。这里我们有 CPU 模型 `。RiscV64` 和 `cpuType` 作为 `rv64imac_zba_zbb`。有关更多详细信息，请转到[配置 RISC-V CPU](https://renode-docs-chinese.readthedocs.io/en/latest/basic/configuring-a-risc-v-cpu.html)。

使用以下 .resc 文件启动 Renode：

```
$ renode asm.resc
(monitor) i $CWD/test.resc
(machine-0) 
```

现在在单独的终端中使用目标架构的 GDB：

```
$ riscv64-unknown-elf-gdb asm.elf
```

在 GDB 中，连接到 Renode 的 GDB 服务器：

```
(gdb) target remote :3333
Remote debugging using :3333
0x00000000000100b0 in _start ()
```

最好将 GDB 的 TUI 与 ASM 和 register 视图一起使用。为此，请执行以下作：

```
(gdb) tui enable
(gdb) layout asm
(gdb) layout regs
```

现在你可以看到 registers 和我们的 assembly 源。要单步执行，请使用 `si`。

```
┌─Register group: general────────────────────────────────────────────────────────────────────────────────────┐
│zero           0x0      0                             ra             0x0      0x0                           │
│sp             0x0      0x0                           gp             0x0      0x0                           │
│tp             0x0      0x0                           t0             0x0      0                             │
│t1             0x0      0                             t2             0x0      0                             │
│fp             0x0      0x0                           s1             0x0      0                             │
│a0             0x0      0                             a1             0x0      0                             │
│a2             0x0      0                             a3             0x0      0                             │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
│  >0x100b0 <_start>        lui     a0,0x64                                                                  │
│   0x100b4 <_start+4>      li      a2,5                                                                     │
│   0x100b6 <_start+6>      sw      zero,0(a0)                                                               │
│   0x100ba <_start+10>     lw      a4,0(a0)                                                                 │
│   0x100bc <_start+12>     sw      a2,0(a0)                                                                 │
│   0x100be <_start+14>     lw      a4,0(a0)                                                                 │
│   0x100c0 <loop>          j       0x100c0 <loop>                                                           │
│   0x100c2                 unimp                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
remote Thread 1 (asm) In: _start                                                            L??   PC: 0x100b0 
(gdb) 
```

请注意，有些指令是伪指令，您编写的源代码不一定与 GDB 中的汇编视图相同。生成的指令在 LLVMAssembler 和 GNU 之间也可能有所不同。

## 调试 tlib

```{note}
请记住使用 debug 标志 （`build.sh -d`， `renode -d`） 编译并启动 Renode，以便其工作！
```

Renode 还为您提供了查看说明如何翻译的选项。为此，我们将使用 VSCode。

启动 Renode 并附加 GDB 后，移动到 VSCode 并导航到 `src/Infrastructure/src/Emulator/Cores/tlib/arch/riscv/translate.c` 并将断点设置在 `case OPC_RISC_LUI：` 之前，然后使用提供的 `（gdb） Tlib Attach` 启动脚本。

搜索 Renode 进程，它应该是 `mono` 或 `dotnet` 类型，附加到它（您可能需要将权限提升到 root，对于无根调试，请检查 [YAMA 的 ptrace_scope](https://www.kernel.org/doc/html/latest/admin-guide/LSM/Yama.html#ptrace-scope)）。

然后在 GDB 中执行 `si`，这应该会让你的 VSCode 在之前设置的断点处停止。