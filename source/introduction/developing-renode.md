# 开发 Renode

Renode 具有许多内置功能来支持嵌入式软件的调试，例如 [使用 GDB 进行调试](../debugging/gdb.md) ，但有时您可能对调试 Renode 本身感兴趣，特别是如果您参与其开发

## 使用 GDB 进行调试

要使用 [GDB](https://www.sourceware.org/gdb/) 开始调试 Renode 及其组件，请执行以下步骤：

1. 在 debug 配置中构建 Renode。

    ```bash
    ./build.sh -d
    ```

1. 在启用调试器的情况下使用 Mono 启动 Renode。

    ```bash
    ./renode -d
    ```

1. 通过 Mono 调试器连接到 Renode，例如 [使用 VS Code](#vs-code-configurations)

1. 将 GDB 附加到正在运行的 Renode 进程，以调试内核的实现（通过 GDB 在 C 语言中）。由于 Mono 使用信号进行流控制，因此 GDB 需要特定的命令。最重要的命令是：

    ```
    handle SIGXCPU SIG33 SIG35 SIG36 SIG37 SIGPWR nostop noprint
    ```


(vs-code-configurations)=

## VS Code 配置

Visual Studio Code 广泛的插件生态系统为使用 Renode 代码库提供了良好的开发人员体验。您可以决定使用 Microsoft 的官方 [VS Code](https://code.visualstudio.com/) 应用程序，也可以使用 OSS 二进制版本 [VSCodium](https://github.com/VSCodium/vscodium) 或 [code-server](https://github.com/coder/code-server) 之一。官方版本和 OSS 版本的主要区别在于它们使用不同的扩展库，因此某些扩展可能不适用于每个版本。

在 VS Code 中启动 Renode 时，您可以使用几个现成的配置（如 [`launch.json` ](https://github.com/renode/renode/blob/master/.vscode/launch.json)文件中所定义）：

* `Launch - Release` - 相当于从控制台运行 `./build.sh` 和 `./renode`。
* `Launch - Debug` - 在 `Debug` 配置 （`./build.sh -d`） 中构建 Renode，并在 Mono 调试器下启动它。
* `(gdb) Tlib Attach` - 需要通过 GDB 连接到之前启动的 Renode 实例，以调试转换库（模拟内核的实现）。

这些配置需要以下 VS Code 扩展：

* [ms-dotnettools.csdevkit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)
* [ms-vscode.mono-debug](https://marketplace.visualstudio.com/items?itemName=ms-vscode.mono-debug)
* [ms-vscode.cpptools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)

### 启动 Renode 并调试

按照以下步骤，您将能够在 Renode 中添加常规断点，包括 C# 和 C 代码。

1. 使用 `Launch - Debug` 配置。
这可能需要一段时间，因为它会在 debug 配置中构建 Renode。

1. 当 Renode 启动时，使用要调试其代码的 CPU 加载平台。

1. 使用 `（gdb） Tlib Attach` 配置与 GDB 连接。

1. 启动后，将出现一个弹出窗口。在弹出窗口中，键入 `mono`。

1. 从下拉列表中，选择以 开头的选项 `/usr/bin/mono --debug --debuger-agent=...` ，如下面的屏幕截图中突出显示的那样。

:::{figure-md}
![VSCode Mono 下拉列表](img/mono.png)

VSCode Mono drop-down list
:::

### 有用的 VS Code 扩展

下面，您可以找到在使用 Renode 时可能有用的 VS Code 扩展列表：

* 描述
    * [Open VSX Registry](https://open-vsx.org/)
    * [Visual Studio Marketplace](https://marketplace.visualstudio.com/)
* C# 支持（代码完成和调试）
    * [muhammad-sammy.csharp](https://open-vsx.org/extension/muhammad-sammy/csharp)
    * [ms-dotnettools.csdevkit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)
* Mono debugger
    * [ms-vscode.mono-debug on Open VSX](https://open-vsx.org/extension/ms-vscode/mono-debug)
    * [ms-vscode.mono-debug on VS Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscode.mono-debug)
* C 支持（代码完成和调试）
    * [llvm-vs-code-extensions.vscode-clangd](https://open-vsx.org/extension/llvm-vs-code-extensions/vscode-clangd) and [vadimcn.vscode-lldb](https://open-vsx.org/extension/vadimcn/vscode-lldb)
    * [ms-vscode.cpptools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
* CMake 支持（用于独立的 Tlib 构建）
    * [ms-vscode.cmake-tools on Open VSX](https://open-vsx.org/extension/ms-vscode/cmake-tools)
    * [ms-vscode.cmake-tools on VS Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)
* Python 支持
    * [ms-python.python on Open VSX](https://open-vsx.org/extension/ms-python/python)
    * [ms-python.python on VS Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
* Robot Framework 语言服务器
    * [robocorp.robotframework-lsp on Open VSX](https://open-vsx.org/extension/robocorp/robotframework-lsp)
    * [robocorp.robotframework-lsp on VS Marketplace](https://marketplace.visualstudio.com/items?itemName=robocorp.robotframework-lsp)
* 使用 Renode 的 GDB 服务器调试嵌入式目标
    * [marus25.cortex-debug on Open VSX](https://open-vsx.org/extension/marus25/cortex-debug) and [webfreak.debug on Open VSX](https://open-vsx.org/extension/webfreak/debug)
    * [marus25.cortex-debug on VS Marketplace](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug) and [webfreak.debug on VS Marketplace](https://marketplace.visualstudio.com/items?itemName=webfreak.debug)

:::{note}

其中一些扩展可能需要额外的配置，具体取决于您计算机的设置。 [Tlib](https://github.com/antmicro/tlib/blob/master/CMakeLists.txt) 应该首先使用 CMake 构建，以生成 `clangd` 语言服务的 `compile_commands.json` 文件。

:::
