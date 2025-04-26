# 使用 Visual Studio Code 调试软件

VS Code 用户也可以使用 Renode 中使用 GDB 模拟的计算机上的调试软件。

为了在 Renode 中启用单击调试，您需要适当配置 VS Code。

Renode 可以运行任何软件，但在本章中，我们将使用示例配置在为 nRF52840 板构建的 Zephyr RTOS 上运行 TFLite Micro 演示。

此类配置由 4 个文件组成：
* .vscode/launch.json
* .vscode/tasks.json
* platform.resc
* debug.conf

示例配置文件位于 Renode 存储库的 `/tools/vscode_config/` 目录中。

所有这些文件都包含带有附加说明的内嵌注释。

```{note}
请注意，JSON 格式不支持注释，但 VSCode 可以毫无问题地处理它们。
```

对于提供的配置文件，VSCode 工作区根目录是运行 “west init” 的目录，即包含 Zephyr、模块、工具等的目录 - 所有路径都是相对于该位置的。

将配置文件放在工作区根目录中，或相应地调整脚本中的路径。

```{note}
.vscode 目录可能隐藏在某些文件资源管理器中。
```

## 启动配置

“launch.json”文件描述了一个名为“Debug application in Renode”的启动配置。如果需要，请调整 ELF 文件的路径和用于调试的 GDB 版本。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug application in Renode",
            "type": "cppdbg",
            "request": "launch",
            "preLaunchTask": "Run Renode",
            "postDebugTask": "Close Renode",
            "miDebuggerServerAddress": "localhost:3333",
            "cwd": "${workspaceRoot}",
            "miDebuggerPath": "arm-zephyr-eabi-gdb",
            "program": "${workspaceRoot}/build/zephyr/zephyr.elf"
        }
    ]
}
```

## 定义任务

示例 `tasks.json` 文件定义了三个任务：

1. “Build application” 是针对此特定场景的示例 build 任务。请根据您的需求进行调整。请注意，生成的二进制文件（需要是 ELF 文件才能进行有效调试）在 “platform.resc” 和 “launch.json” 中引用。

1. “Run Renode” 在 “Build application” 完成后运行。它启动 “platform.resc” 脚本并等待 GDB 服务器启动。它需要指向 resc 脚本的正确路径。

1. “Close Renode” 是关闭调试会话时触发的任务。它会关闭 Renode，是否要使用它是个人喜好的问题。您可以通过删除/注释掉 “postDebugTask” 行，在 “launch.json” 中禁用它。

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build application",
            "type": "shell",
            "command": "west",
            "args": [
                "build",
                "--pristine",
                "--board",
                "nrf52840dk_nrf52840",
                "zephyr/samples/modules/tflite-micro/hello_world",
                "--",
                "-DEXTRA_CONF_FILE=${workspaceFolder}/debug.conf"
            ],
            "problemMatcher": [
                "$gcc"
            ],
            "group": "build",
            "presentation": {
                "reveal": "always",
                "panel": "dedicated"
            }
        },
        {
            "label": "Run Renode",
            "type": "shell",
            "command": "renode",
            "args": [
                "${workspaceFolder}/platform.resc"
            ],
            "dependsOn": [
                "Build application"
            ],
            "isBackground": true,
            "problemMatcher": {
                "source": "Renode",
                "pattern": {
                    "regexp": ""
                },
                "background": {
                    "activeOnStart": true,
                    "beginsPattern": "Renode, version .*",
                    "endsPattern": ".*GDB server with all CPUs started on port.*"
                }
            },
            "group": "build",
            "presentation": {
                "reveal": "always",
                "panel": "dedicated"
            }
        },
        {
            "label": "Close Renode",
            "command": "echo ${input:terminate}",
            "type": "shell",
            "problemMatcher": []
        }
    ],
    "inputs": [
        {
            "id": "terminate",
            "type": "command",
            "command": "workbench.action.tasks.terminate",
            "args": "terminateAll"
        }
    ]
}
```

## Renode 模拟脚本

“platform.resc” 文件是一个示例 Renode 模拟脚本：

```
$bin ?= $CWD/build/zephyr/zephyr.elf
include @scripts/single-node/nrf52840.resc

machine StartGdbServer 3333
```

它包含两个重要部分：
- 指定 ELF 文件
- 启动 GDB 作为脚本中的最后一个命令。您需要根据自己的特定需求调整文件。

您可以使用此脚本来设置整个仿真，也可以依赖预定义的脚本。大多数 Renode 脚本使用 `bin` 变量来指定要运行的可执行文件。它还应与 `launch.json` 中的配置匹配。

## debug.conf

“debug.conf” 文件是特定于 Zephyr 的。它用于在编译期间禁用优化，因为某些优化可能会阻碍调试过程。您可以完全删除此文件，具体取决于您选择优化的方式。
