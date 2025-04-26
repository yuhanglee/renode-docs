# 生成覆盖率报告

Renode 提供了许多执行跟踪功能，如[执行跟踪](./execution-tracing.md#execution-tracing)部分所述，并允许您将二进制文件的整个执行转储到单个文件中。使用正确构建的二进制文件和专用脚本的执行跟踪，您可以创建代码覆盖率报告。

[该脚本](https://github.com/renode/renode/blob/master/tools/execution_tracer/execution_tracer/execution_tracer_reader.py)将 [DWARF 格式](https://dwarfstd.org/)的数据（通常用于将软件加载到 Renode 的 ELF 文件中可用的调试信息）与 Renode 计数的每条指令的执行次数相结合。以下教程将引导您完成在 Renode 中生成此类报告所需的步骤。

## 先决条件

要生成报告，您需要安装 Renode 以及[执行跟踪器脚本](https://github.com/renode/renode/tree/master/tools/execution_tracer) 。在 Linux 上，您可以按照[这些说明](https://github.com/renode/renode/blob/master/README.md#using-the-linux-portable-release)简单地下载便携式软件包。

假设您在 Renode 存储库的根目录中执行其余命令。否则，您可能需要相应地调整它们。

要运行该脚本，您可以将其安装为 CLI 实用程序，例如使用 [pipx](https://github.com/pypa/pipx)：

```sh
pipx install tools/execution_tracer/
```

然后，您将能够使用 call：

```sh
renode-retracer --help
```
查看可用命令的列表

或者，可以直接从 Renode 目录调用脚本。假设变量 `RENODE_PATH` 指向您的 Renode 安装或克隆的存储库。
首先，初始化虚拟环境并安装依赖项：

```sh
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r ${RENODE_PATH}/tools/execution_tracer/requirements.txt
```

然后，您应该能够运行：

```sh
python3 ${RENODE_PATH}/tools/execution_tracer/execution_tracer_reader.py --help
```
以查看可用命令的列表。


## 构建用于覆盖率收集的二进制文件

Renode 可以跟踪以任何方式构建的二进制文件的执行情况，但要提取覆盖率信息，我们需要一个 DWARF 格式的带有调试信息的二进制文件。为了提供最佳结果，建议在编译过程中使用 `-g` 和 `-O0`（或 `-Og`）标志（对于 GCC）。请注意，不同的编译器可能需要不同的选项。

`-g` 开关将调试信息添加到二进制文件中，特别是每个机器指令的代码行号。覆盖率功能需要该信息，如果 ELF 文件中缺少该信息，您将看到错误。建议使用最高可用的调试级别（目前为 `-g3`）。

强烈建议使用 `-O0` 开关禁用所有优化。`-Og` 开关会禁用大多数优化，旨在简化调试过程。

编译器所做的任何优化都可能阻止执行冗余的代码行。编译器还可能在程序中的多个地址之间复制相同的代码行，或者对执行流重新排序，这将使无法获得合法的结果。为了生成覆盖率报告，与调试相同，建议执行代码而不进行任何优化。为优化的二进制文件生成报告可能会产生不精确甚至意外的结果。

## 收集示例程序的覆盖率

在本教程中，我们准备了一个由两个文件组成的示例程序。本教程介绍了如何收集在仿真 [Kendryte K210 RISC-V 平台](https://github.com/renode/renode/blob/c5cd6fe6fa33ccbfc2f1f62ff0c87252fd2e5259/platforms/cpus/kendryte_k210.repl)上运行的该程序的执行指标。

```{note}
预构建的示例和源代码可[在此处](https://dl.antmicro.com/projects/renode/coverage-sample-riscv64.tar)下载。
```

该程序的源代码如下所示：

::::{tab} `main.c`
```c
const int buf_size = 100;

void funB(int *buf, int b) {
    for (int i = 0; i < buf_size; i++) {
        buf[i] -= b;
    }
}

void funA(int *buf, int b);
void funC();

void buf_increment_all(int *buf) {
    for (int i = 0; i < buf_size; i++) {
        buf[i] += 1;
    }
}

int main() {
    int buf[buf_size];
    for (int i = 0; i < buf_size; i++) {
        buf[i] = i;
    }

    buf_increment_all(buf);

    for (int i = 0; i < buf_size; i += 1) {
        if (buf[i] % 3 == 0) {
            funA(buf, i);
        }
        if (buf[i] % 5 == 0) {
            funB(buf, i);
        }
    }

    int i = 11;

    if (i < 10) {
        funC();
    }

    buf_increment_all(buf);

    while (1) ; // Don't let the program run-off!

    return 0;
}
```
::::

::::{tab} `additional.c`
```c
extern const int buf_size;

int funC() {
        return 42;
}

void funA(int *buf, int b) {
        for (int i = 0; i < buf_size; i++) {
                buf[i] += b;
        }
}
```
::::

要编译它，首先你需要获取一个 `riscv64` 工具链。请参阅 OS 发行版的软件包存储库，或从[源代码](https://github.com/riscv-collab/riscv-gnu-toolchain)自行编译。

```{note}
提示：您可以使用 [Zephyr SDK](https://docs.zephyrproject.org/latest/develop/toolchains/zephyr_sdk.html#zephyr-sdk-installation) 附带的预构建工具链。
```

然后，使用 （根据需要调整编译器名称） 编译程序：

```sh
riscv64-unknown-elf-gcc -O0 -g3 main.c additional.c --freestanding -nostdlib -Wl,-emain -o coverage-sample.elf
```

附加标志通知编译器我们在裸机环境中运行，因此我们不想与标准库链接。它们对覆盖率报告没有影响。

### 平台和跟踪执行

要跟踪上一节中二进制文件的执行情况，您只需使用[预先准备好的 RESC 脚本](https://github.com/renode/renode/blob/b509e7c85265e6d203dc632bcd29319ac758308a/scripts/complex/coverage/kendryte_k210_coverage.resc)即可。

```{note}
该脚本运行为 Kendryte K210 平台预先构建的二进制文件，但跟踪机制本身 （`CreateExecutionTracing`） 是通用的，适用于所有平台。
```

该脚本在功能上类似于此脚本：
```none
$bin=$CWD/coverage-sample.elf
include @scripts/single-node/kendryte_k210.resc
cpu2 IsHalted true
cpu1 SP 0x1000

cpu1 CreateExecutionTracing "trace" $CWD/trace.bin.gz PC isBinary=True compress=True
```

我们专注于单个内核并禁用另一个内核。我们还手动设置了 Stack Pointer，以简化软件。

您可以通过键入以下内容在 Renode 中加载脚本：

```none
include @scripts/complex/coverage/kendryte_k210_coverage.resc

emulation RunFor "0.003" # Run 300 000 instructions (default performance is 100MIPS)
quit
```

上述命令收集 guest 时间的 0.003 秒跟踪。执行完成后，使用 `quit` 关闭 Renode 实例。跟踪将以二进制格式保存到名为 `trace.bin.gz` 的文件中。该文件还被压缩以减小大小。

`CreateExecutionTracing` 命令初始化执行跟踪。 [的 Execution tracing](../execution-tracing/execution-tracing.md#execution-tracing) 部分包含有关配置命令的详细信息。生成覆盖率报告的脚本至少需要 PC（已执行指令的地址）才能在跟踪文件中可用。

```{note}
跟踪其他类型的数据可能会导致输出文件大小增加。此外，还建议通过将第四个和第五个参数设置为 `True` 来使用二进制格式和压缩，如上所示。
```

### 生成报告

此时，您可以运行生成报告的脚本：

```sh
renode-retracer coverage trace.bin.gz \
  --binary coverage-sample.elf \
  --sources main.c additional.c \
  --output coverage.zip \
  --export-for-coverview
```

```{note}
如果未使用 `--sources` 参数提供任何源，则脚本将尝试根据从二进制文件中提取的 DWARF 数据自动发现其位置。

如果源的位置与构建二进制文件（或二进制文件是在另一台计算机上构建）相比发生了变化，则可能需要执行路径替换。

为此，您可以通过为要替换的每对路径提供 `old_path：new_path` 来使用 `--sub-source-path` 参数;此参数可以多次提供。
```

通过将 `--export-for-coverview` 添加到命令行，该工具可以打包一个存档，以供 [Coverview](#coverview-integration)（一种用于生成覆盖率仪表板的工具）处理。对于 `main.c` 文件，输出将类似于如下所示。

![Coverage in Coverview](img/coverview-simple-app.png)

也可以使用[与 LCOV 兼容的](#report-formats)格式 （`.info`）。虽然对人类来说不那么容易阅读，但它更容易被自动化脚本和工具处理。

## 报告格式

通常，脚本将以与 [LCOV](https://github.com/linux-test-project/lcov) 兼容的格式 （`.info`） 输出数据。格式在 `geninfo` （`man geninfo`） 的手册页中进行了介绍。 `renode-retracer` 支持生成线路覆盖信息 （`DA）。` 此格式稍后可以与各种第三方工具（例如 `genhtml`）一起使用，以显示覆盖率数据。

通过使用 `--export-for-coverview` 开关，数据将被打包到一个存档中，准备由 [Coverview](#coverview-integration) 处理。

`renode-retracer` 还支持传统的基于文本的报告格式，以便于结果解释，如[上一节](#legacy-report-mode)所示。

## Coverview 集成

执行跟踪器可以将数据导出为可以直接加载到 Coverview 中的格式。为此，请使用 `--export-for-coverview` 开关，并记住使用 `--output filename.zip` 选项提供输出文件名。

[Coverview](https://github.com/antmicro/coverview) 是一种用于生成覆盖率仪表板的工具。该工具完全在客户端运行，您可以通过 `npm` 在本地构建它，也可以访问[部署在 GitHub Pages 上的](https://antmicro.github.io/coverview/index.html)页面。您可以上传从本章中介绍的场景获取的档案，并调查线路覆盖率。服务器上不会存储任何上传的数据。

然后，您可以将存档加载到控制面板中，并逐个文件浏览它。

## 收集 Zephyr 的覆盖率

此示例使用脚本收集在模拟 [Nucleo H753ZI](https://docs.zephyrproject.org/latest/boards/st/nucleo_h753zi/doc/index.html) 上运行的 Zephyr 应用程序的覆盖率。请按照以下步骤准备测试场景，并查看测试覆盖了多少代码。

首先，下载 Zephyr 并按照[官方指南中的](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)说明设置您的环境。

按照 `入门` 部分中描述的步骤作后，构建 shell 示例只需要一个命令：

```sh
west build -b nucleo_h753zi zephyr/samples/subsys/shell/shell_module/ -- -DCONFIG_NO_OPTIMIZATIONS=y
```

请注意，优化被完全禁用，以提高收集的覆盖率信息的质量。或者，您也可以在仅启用调试优化的情况下构建示例 （ `-DCONFIG_DEBUG_OPTIMIZATIONS=y` ）。向`西`运行后，您可以在 `build/zephyr/` 目录中找到编译后的 `zephyr.elf` 文件 - 想要获取覆盖率信息的 shell 应用程序。

下面是一个确定性的 Robot 测试文件，用于测试我们案例的覆盖范围：

```robotframework
*** Test Cases ***
Should Report Shell Coverage
    Execute Command           include @platforms/boards/nucleo_h753zi.repl
    Execute Command           sysbus LoadELF $CWD/build/zephyr/zephyr.elf
    Execute Command           cpu CreateExecutionTracing "trace" @${CURDIR}/trace.bin.gz PC True True
    Create Terminal Tester    sysbus.usart3  defaultPauseEmulation=True

    Wait For Prompt On Uart   uart:~$
    Write Line To Uart        demo board
    Wait For Line On Uart     nucleo_h753zi
    Wait For Prompt On Uart   uart:~$
```

下面是一个确定性的 Robot 测试文件，用于测试我们案例的覆盖范围：

下面是一个确定性的 Robot 测试文件，用于测试我们案例的覆盖范围：

跟踪将保存到 `trace.bin.gz` 文件中，该文件与 Robot 测试文件位于同一目录中。

要创建覆盖率报告，请执行以下命令：

```sh
renode-retracer coverage trace.bin.gz \
  --binary build/zephyr/zephyr.elf \
  --output coverage.zip \
  --export-for-coverview
```

故意不提供来源。该脚本将自动发现并加载它们，前提是它们仍然存在于进行构建的当前计算机上。否则，您可能需要使用 `--sub-source-path` 调整路径。

生成覆盖率数据可能需要一些时间来处理，具体取决于应用程序的代码库大小和程序的执行时间（跟踪的大小）。

之后，可以将存档加载到 [Coverview](#coverview-integration) 中并以交互方式浏览。对于负责打印板名称 （ `samples/subsys/shell/shell_module/src/main.c` ） 的代码，它可能如下所示：

![image](img/coverview-main-screen.png)

## 旧版报告模式

也可以使用简单的基于文本的模式来调查覆盖率。为此，请传递 `--legacy` 开关。例如，要从上述[示例应用程序](#gathering-coverage-for-a-sample-program)获取 `main.c` 的覆盖率数据，可以执行以下作：

```sh
renode-retracer coverage trace.bin.gz \
  --binary coverage-sample.elf \
  --sources main.c \
  --legacy
```

输出应以行开头，如下所示。冒号前的数字表示每行的执行次数。

```c
     0:   const int buf_size = 100;
     0:   
    28:   void funB(int *buf, int b) {
  2828:       for (int i = 0; i < buf_size; i++) {
  2800:           buf[i] -= b;
     0:       }
    28:   }
     0:   
     0:   void funA(int *buf, int b);
     0:   void funC();
     0:   
     2:   void buf_increment_all(int *buf) {
   202:       for (int i = 0; i < buf_size; i++) {
   200:           buf[i] += 1;
     0:       }
     2:   }
...
```
