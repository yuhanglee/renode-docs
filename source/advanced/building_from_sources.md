# 从源码构建 Renode

本文档提供了有关如何准备构建环境，然后构建和测试 Renode 本身的详细信息。

## 先决条件

### 核心先决条件

::::{tab} Linux

以下说明已在 Ubuntu 22.04 上进行了测试，但是应该不会有任何重大问题阻止您使用其他（尤其是基于 Debian）的发行版。

:::{tab} Mono
首先，根据 [Mono 项目网站上](https://www.mono-project.com/download/stable/#download-lin)可找到的各种 Linux 发行版的安装说明安装 `mono-complete` 包。
:::

:::{tab} .NET
首先，按照安装说明安装 `.NET SDK` 包，该说明可在 [.NET 官方网站](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)上找到。
:::

要安装其余依赖项，请使用：

    sudo apt update
    sudo apt install git automake cmake autoconf libtool g++ coreutils policykit-1 \
                  libgtk-3-dev uml-utilities gtk-sharp3 python3 python3-pip

::::

::::{tab} macOS

在 macOS 上，可以使用 [Mono 项目网站上的下载链接](https://download.mono-project.com/archive/mdk-latest-stable.pkg)获取 Mono 包。

要安装其余的必备组件，请使用：

    brew install binutils gnu-sed coreutils dialog cmake
    xcode-select --install

:::{note}
   这需要在您的系统中安装 [homebrew](https://brew.sh/)。
:::

::::


::::{tab} Windows

在 Windows 上构建 Renode 使用 MinGW 和 Git Bash，需要你正确设置系统环境。

**Git**

1. 使用默认选项下载并安装  `git`  您可以从[官方网站](https://git-scm.com/downloads)获取它。
2. 确保安装目录（默认为 `C：\Program Files\Git`）位于系统 `PATH` 变量中。

:::{note}
在  *Windows* 上克隆存储库之前，必须适当配置 git。在 Git Bash 中运行以下命令以正确设置选项：

    git config --global core.autocrlf false
    git config --global core.symlinks true
:::

**Python 3**

1. 从 [Python 网站](https://www.python.org/downloads/)下载并安装 Python 3 框架的 Windows 版本。
2. 将二进制文件的位置添加到系统 `PATH` 变量中。安装程序可以为您执行此作。

**MinGW**

1. 从[下载站点](https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win64/Personal%20Builds/mingw-builds/8.1.0/threads-win32/sjlj/x86_64-8.1.0-release-win32-sjlj-rt_v6-rev0.7z)下载具有 `x86_64` 架构、`win32` 线程和 `sjlj` 异常处理的 `MinGW-w64 8.1.0`。
2. 解压缩下载的包，并将其 `mingw64\bin` 目录（例如 `C:\mingw-w64\x86_64-8.1.0-release-win32-sjlj-rt_v6-rev0\mingw64\bin` ）添加到系统 `PATH` 变量中。

**CMake**

1. 从 [CMake 网站](https://cmake.org/download/)下载 `CMake` 并安装 Windows CMake。
2. 确保安装目录在系统 `PATH` 中。安装程序将为你执行此操作。

**C# 生成工具**

:::{tab} .NET Framework

1. 下载 [VS Build Tools 2019](https://aka.ms/vs/16/release/vs_BuildTools.exe)。
2. 运行安装程序，选择  *Visual Studio Build Tools 2019* 产品，然后单击 *Install （安装 ）* 或  *Modify （修改 ）* 。
3. 切换到  *Individual components （单个组件）*  窗格，然后选择：

   *  .NET 部分中的 .NET Framework 4.6.2 目标包 ，
   * 代码工具部分中的 NuGet 目标和生成任务 。

4. 将二进制文件的位置（ `C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools\MSBuild\Current\Bin\amd64` 默认）添加到系统 `PATH` 变量中。

:::

:::{tab} .NET

有关如何安装 `.NET SDK` 的说明， [请参阅 .NET 官方网站](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) 。

:::

::::

## 下载源码

Renode 的源代码可在 GitHub 上找到：

    git clone https://github.com/renode/renode.git
    cd renode

子模块将在构建过程中自动初始化和下载，因此此时无需执行此作。

### 其他先决条件（用于 Robot 框架测试）

如果您按照上述说明作，则应在您的系统中安装 Python。安装 `pip` 包管理器和一些其他模块，以便能够使用 Robot 框架编写和运行测试用例：

    python3 -m pip install -r tests/requirements.txt

## 构建 Renode

:::{note}
在 Windows 上，本节中描述的构建过程只能在 Git Bash 中执行。
:::

:::{note}
使用 `.NET` 构建 Renode 时，请记住使用 `--net` 开关 （`./build.sh --net`）。
:::

要构建 Renode，请运行：

    ./build.sh

您可以使用一些可选标志：

    -c                                clean instead of building
    -d                                build in debug configuration
    -v                                verbose
    -p                                create packages after building
    -n                                create nightly packages after building
    -t                                create a portable package (experimental, Linux only)
    -s                                update submodules
    -b                                custom build properties file
    -o                                custom output directory
    --skip-fetch                      skip fetching submodules and additional resources
    --no-gui                          build with GUI disabled
    --force-net-framework-version     build against different version of .NET Framework than specified in the solution
    --net                             build with dotnet
    -B                                bundle target runtime (default value: linux-x64, requires --net, -t)
    -F                                select the target framework for which Renode should be built (default value: net8.0)
    --profile-build                   build optimized for tlib profiling
    --tlib-only                       only build tlib
    --tlib-arch                       build only single arch (implies --tlib-only)
    --tlib-export-compile-commands    build tlibs with 'compile_commands.json' (requires --tlib-arch)
    --host-arch                       build with a specific tcg host architecture (default: i386)
    --skip-dotnet-target-generation   don't generate 'Directory.Build.targets' file, useful when experimenting with different build settings


此外，您可以直接指定标志，这些标志将在 `--` 之后传递给构建系统。例如，如果要覆盖 `CompilerPath` 属性，可以使用：

    ./build.sh -- p:CompilerPath=/path/to/gcc

你也可以从 IDE 构建 `Renode.sln`（比如 MonoDevelop 或 Visual Studio），但 `build.sh` 脚本必须至少运行一次。

## 创建包

构建脚本只能创建本机软件包，即，您必须在 Windows 上运行它以创建 `.msi` 安装程序包，在 Linux 上运行它以创建 `.deb`、`.rpm` 和 `.pkg.tar.xz` 软件包，或者在 macOS 上运行它以创建 `.dmg` 映像。

还有一个用于创建 [Conda](https://docs.conda.io/en/latest/) 包的单独过程，如[专用 README](https://github.com/renode/renode/tree/master/tools/packaging/conda) 中所述。

### 先决条件

根据系统的不同，构建 Renode 包可能有一些先决条件。

:::{tab} Linux

运行：

    sudo apt-get install ruby ruby-dev rpm 'bsdtar|libarchive-tools'
    sudo gem install fpm

:::

:::{tab} macOS

macOS 没有其他先决条件。

:::

::::{tab} Windows

:::{note}

在 Windows 10 上，在安装 WiX 工具集之前在系统中启用 .NET 3.5 非常重要。

本节中描述的打包过程只能在 Git Bash 中执行。

:::

下载并安装 [WiX Toolset 安装程序](https://wixtoolset.org/releases/) （版本至少为 3.11）。

::::

### 构建

要构建二进制包，请运行：

    ./build.sh -p

包将分配一个版本，由 `tools/version` 文件的内容定义。

您还可以使用以下方法构建 nightly 包：

    ./build.sh -pn

这会将日期和提交 SHA 附加到输出文件。

### 软件包的位置

成功完成后，脚本将打印所创建文件的位置：

:::{tab} Linux

`renode/output/packages/renode_<version>.{deb|rpm|tar.gz}`

:::

:::{tab} macOS

`renode/output/packages/renode_<version>.dmg`

:::

:::{tab} Windows

`renode/output/packages/renode_<version>.msi`

:::
