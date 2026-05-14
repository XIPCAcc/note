vscode 调试环境

```bash
sudo apt update
sudo apt install -y build-essential cmake gdb git glibc-source

cd /usr/src/glibc
ls                      # 看有哪一版，比如 glibc-2.39.tar.xz
sudo tar -xvf glibc-2.39.tar.xz   # 解出来 /usr/src/glibc/glibc-2.39，可以调试到C库

git clone https://github.com/intel/uintr-ipc-bench.git
cd uintr-ipc-bench
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Debug ..
make -j"$(nproc)"
```

## 插件
在 VSCode 里安装这两个扩展：
C/C++（ms-vscode.cpptools）
CMake Tools（ms-vscode.cmake-tools）

## c_cpp_properties.json

在项目根目录创建 .vscode/c_cpp_properties.json
这样 VSCode 才能正确找到头文件、跳转符号。
```json
{
  "version": 4,
  "configurations": [
    {
      "name": "Linux",
      "includePath": [
        "${workspaceFolder}/**",
        "${workspaceFolder}/source/common",
        "${workspaceFolder}/source/uintrfd"
      ],
      "defines": [],
      "compilerPath": "/usr/bin/gcc",
      "cStandard": "gnu11",
      "cppStandard": "gnu++14",
      "intelliSenseMode": "linux-gcc-x64",
      "configurationProvider": "ms-vscode.cmake-tools"
    }
  ]
}
```

## tasks.json

在 .vscode/tasks.json 中写一个简单的 CMake 构建任务，后面调试可以自动先编译：

```rust
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "CMake Configure",
      "type": "shell",
      "command": "cmake",
      "args": [
        "-S",
        "${workspaceFolder}",
        "-B",
        "${workspaceFolder}/build",
        "-DCMAKE_BUILD_TYPE=Debug"
      ],
      "group": "build",
      "problemMatcher": []
    },
    {
      "label": "CMake Build",
      "type": "shell",
      "command": "cmake",
      "args": [
        "--build",
        "${workspaceFolder}/build",
        "--parallel"
      ],
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": []
    }
  ]
}
```

## launch.json
在 VSCode 左侧「运行和调试」面板里，点「创建 launch.json」，类型选「C++ (GDB/LLDB)」，

program 指向你想调试的可执行文件，比如：
build/source/shm/shm
build/source/uintrfd/uintrfd-uni
build/source/tcp/tcp
args 对应 README 里的参数：
-c 消息次数
-s 消息大小
preLaunchTask 会在每次按 F5 前自动执行一次构建，保证二进制是最新的。

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Signal",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/source/signal/signal",
      "args": [ "-c", "1" ],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "externalConsole": false,
      "MIMode": "gdb",
      // "preLaunchTask": "CMake Build",
      "setupCommands": [
        {
          "description": "启用 gdb pretty-printing",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        },
        {
          "description": "添加 glibc 源码目录（让 gdb 能找到 arch-fork.h 等）",
          "text": "directory /usr/src/glibc/glibc-2.39",
          "ignoreFailures": true
        },
        {
          "description": "fork 后父子进程都保持在调试中",
          "text": "set detach-on-fork off",
          "ignoreFailures": false
        }
      ]
    }
  ]
}
```

## 单步调试signal

一定要让signal发送方先发送信号，才能让接收方单步运行到sigwait，否则会导致整个调试线程阻塞

调试的时间也会算到最后的数据中，所以这个数据不准确。

## 汇编
ctrl + shift + P
Open Disassembly View
Debug: Open Disassembly View