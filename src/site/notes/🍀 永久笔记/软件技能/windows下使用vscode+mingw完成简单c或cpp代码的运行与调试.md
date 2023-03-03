---
{"dg-publish":true,"permalink":"/🍀 永久笔记/软件技能/windows下使用vscode+mingw完成简单c或cpp代码的运行与调试/"}
---


[国内镜像链接](https://blog.csdn.net/qq_33698226/article/details/129031241)

## 参考

1. [ 基于 VS Code + MinGW-w64 的 C 语言/C++简单环境配置，专致小白](https://zhuanlan.zhihu.com/p/77074009)

## 安装

环境：`Win10 21H2 19044.1381` + `mingw` + [vscode 1.74.3](https://code.visualstudio.com/docs/?dv=win) + `C/C++ Extension Pack` 插件

国内镜像盘: [链接](https://gonglja.lanzouy.com/b00ql2fza)  密码:1dcs

**简单来说共分为 4 步**

1. 下载 `mingw`，`vscode` 和 `插件` 并安装
2. 添加 `mingw` 编译链路径到 **系统环境** 变量 `Path`
3. 拷贝 `tasks.json`、`launch.json` 到要运行的代码根目录 `.vscode` 文件夹中
4. 修改 `tasks.json`、`launch.json` 中 `gcc`、`g++`、`gdb` 路径

### 安装 vscode 和插件

- 下载 [vscode](https://code.visualstudio.com/docs/?dv=win) 并安装
- 安装 `C/C++ Extension Pack` 插件

### 配置 mingw

#### 下载 mingw 编译链

在 [此处](https://sourceforge.net/projects/mingw-w64/files/mingw-w64/mingw-w64-release/) 下载 windows 编译链，编译链命名方式如：系统 - 线程模型 - 异常处理模式

- 系统
	- **x84_64** 64 位可执行文件
	- i686 32 位可执行文件
- 线程模型
	- win32
	- posix
- 异常处理方式
	- ...

根据自己的系统选择，推荐下载最新的，我这里是 64 位系统，线程模型选择 win32，异常模式随便选

选择 [x86_64-win32-seh](https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win64/Personal%20Builds/mingw-builds/8.1.0/threads-win32/seh/x86_64-8.1.0-release-win32-seh-rt_v6-rev0.7z)

![Pasted image 20230114180958.png](/img/user/Resources/Images/Pasted%20image%2020230114180958.png)

**需注意：win 下如线程模式使用 posix，会造成 powershell 下无法通过编译链生成 a.exe，推荐使用如下图中工具链**

![Pasted image 20230116100458.png](/img/user/Resources/Images/Pasted%20image%2020230116100458.png)

#### 配置环境变量

将在 [[🍀 永久笔记/软件技能/windows下使用vscode+mingw完成简单c或cpp代码的运行与调试# 下载 mingw 编译链\|# 下载 mingw 编译链]] 下载的编译链解压到路径 xxx 下，记住该路径

`windows 键 + R` 打开 **运行**，输入 `sysdm.cpl`，回车进入

![Pasted image 20230114182219.png](/img/user/Resources/Images/Pasted%20image%2020230114182219.png)

点击 **高级**

![Pasted image 20230114182315.png](/img/user/Resources/Images/Pasted%20image%2020230114182315.png)

点击 **环境变量** 添加 mingw 工具链路径到 `Path` 下

![Pasted image 20230114182359.png](/img/user/Resources/Images/Pasted%20image%2020230114182359.png)

下载后解压的工具链路径：

![Pasted image 20230116090634.png](/img/user/Resources/Images/Pasted%20image%2020230116090634.png)

推荐在 **系统变量** 中的 **Path** 下添加刚刚下载的编译链的路径 xxx + `\bin`

比如你下载后解压的路径为 `C:\1\mingw64`，那么在此处就要添加 `C:\1\mingw64\bin`

![Pasted image 20230114182525.png](/img/user/Resources/Images/Pasted%20image%2020230114182525.png)

环境变量配置：**添加后不要忘记点确定**

![Pasted image 20230116090611.png](/img/user/Resources/Images/Pasted%20image%2020230116090611.png)

### 配置 vscode 环境

将下述代码保存为 `launch.json` 或者直接使用镜像中的 `.vscode` 中的 `launch.json`，修改配置文件中 `launch.json` 的 `gcc`、`g++`、`gdb` 路径

将 `launch.json` 中 `miDebuggerPath` 修改为你本地 `mingw` 中的 `gdb` 路径

```json
{
    "configurations": [
        {
            "name": "(gdb) 启动",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}\\${fileBasenameNoExtension}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "C:\\1\\mingw64\\bin\\gdb.exe",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ]
        }
    ],
    "version": "2.0.0"
}
```

将下述代码保存为 `tasks.json` 或者直接使用镜像中的 `.vscode` 中的 `tasks.json`,

将 `tasks.json` 中的 `cbuild` 中 `command` 改为你本地 `gcc` 的路径，`cppbuild` 中 `command` 改为你本地 `g++` 的路径

```json
{
    "tasks": [
        {
            "type": "cbuild",
            "label": "C/C++: gcc.exe 生成活动文件",
            "command": "C:\\1\\mingw64\\bin\\gcc.exe",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "${file}",
                "-o",
                "${fileDirname}\\${fileBasenameNoExtension}.exe"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "gdb调试器生成的任务。"
        },
        {
            "type": "cppbuild",
            "label": "C/C++: g++.exe 生成活动文件",
            "command": "C:\\1\\mingw64\\bin\\g++.exe",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "${file}",
                "-o",
                "${fileDirname}\\${fileBasenameNoExtension}.exe"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$g++"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "gdb调试器生成的任务。"
        }
    ],
    "version": "2.0.0"
}
```

最终 Vscode 工程目录结构如下，测试代码来自 [hello-algo](https://www.hello-algo.com/)，在此感谢大佬 [@Krahets](https://leetcode.cn/u/jyd/)

````shell
```shell
C:\Users\Administrator\Desktop\TEST\code>tree /F
文件夹 PATH 列表
卷序列号为 DE24-5DAC
C:.
├─.vscode
│      launch.json
│      tasks.json
│
└─codes
    ├─c
    │  │  CMakeLists.txt
    │  │
    │  ├─chapter_array_and_linkedlist
    │  │      array.c
    │  │      CMakeLists.txt
    │  │      linked_list.c
    │  │
    │  ├─chapter_computational_complexity
    │  │      CMakeLists.txt
    │  │      time_complexity.c
    │  │      worst_best_time_complexity.c
    │  │
    │  ├─chapter_sorting
    │  │      bubble_sort.c
    │  │      CMakeLists.txt
    │  │      insertion_sort.c
    │  │
    │  ├─chapter_stack_and_queue
    │  │
    │  ├─chapter_tree
    │  │      binary_search_tree.c
    │  │      binary_tree.c
    │  │      binary_tree_bfs.c
    │  │      binary_tree_dfs.c
    │  │      CMakeLists.txt
    │  │
    │  └─include
    │          CMakeLists.txt
    │          include.h
    │          include_test.c
    │          list_node.h
    │          print_util.h
    │          tree_node.h
    │
    ├─cpp
    │  │  .gitignore
    │  │  CMakeLists.txt
    │  │  Makefile
    │  │
    │  ├─chapter_array_and_linkedlist
    │  │      array.cpp
    │  │      linked_list.cpp
    │  │      list.cpp
    │  │      my_list.cpp
    │  │
    │  ├─chapter_computational_complexity
    │  │      leetcode_two_sum.cpp
    │  │      space_complexity.cpp
    │  │      time_complexity.cpp
    │  │      worst_best_time_complexity.cpp
    │  │
    │  ├─chapter_hashing
    │  │      array_hash_map.cpp
    │  │      hash_map.cpp
    │  │
    │  ├─chapter_searching
    │  │      binary_search.cpp
    │  │      hashing_search.cpp
    │  │      linear_search.cpp
    │  │
    │  ├─chapter_sorting
    │  │      bubble_sort.cpp
    │  │      insertion_sort.cpp
    │  │      merge_sort.cpp
    │  │      quick_sort.cpp
    │  │
    │  ├─chapter_stack_and_queue
    │  │      array_queue.cpp
    │  │      array_stack.cpp
    │  │      deque.cpp
    │  │      linkedlist_queue.cpp
    │  │      linkedlist_stack.cpp
    │  │      queue.cpp
    │  │      stack.cpp
    │  │
    │  ├─chapter_tree
    │  │      binary_search_tree.cpp
    │  │      binary_tree.cpp
    │  │      binary_tree_bfs.cpp
    │  │      binary_tree_dfs.cpp
    │  │
    │  └─include
    │          include.hpp
    │          ListNode.hpp
    │          PrintUtil.hpp
    │          TreeNode.hpp
    │  ...
    │
    ├─java
    │ ...
    ├─python
    │ ...
    ├─rust
    │ ...
    ├─swift
    │ ...

C:\Users\Administrator\Desktop\TEST\code>
````

## 测试

首先，通过 Vscode 打开工程（含有 `.vscode` 文件夹的目录），在打开 `c` 文件，根据右上角选择 **调试** 还是 **运行**，也可以通过快捷键调试（F5）

![Pasted image 20230120054846.png](/img/user/Resources/Images/Pasted%20image%2020230120054846.png)

如下为调试过程中的截图：

`gdb` 调试 `c` 文件，如下图

![Pasted image 20230116101007.png](/img/user/Resources/Images/Pasted%20image%2020230116101007.png)

`gdb` 调试 `cpp` 文件，如下图

![Pasted image 20230116101401.png](/img/user/Resources/Images/Pasted%20image%2020230116101401.png)
