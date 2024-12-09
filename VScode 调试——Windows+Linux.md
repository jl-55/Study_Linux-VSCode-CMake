# VScode 调试——Windows+Linux

# Windows：

## 一、单文件调试

### 1.1

创建好一个源文件，打好断点，然后直接按F5或者点击调试即可

如果操作成功了，最后会看到生成了一个叫做 .vscode的目录, 并且里边有两个文件launch.json和tasks.json

### 1.2、菜单栏->终端->新建终端

在得到的 Power Shell窗口中使用 gcc 编译源文件:这是生成可执行文件

```shell
# 编译c源文件
gcc xxx.c -o 可执行程序名

# 编译C++源文件
g++ xxx.cpp -o 可执行程序名

e.g:
g++ test.cpp -o test 
有可能要显示指定标志。添加 -mconsole 选项，告诉编译器这是一个控制台应用
g++ test.cpp -o test -mconsole  
```



## 二、多文件调试——基于CMake进行配置并调试

如果一个项目中有多个源文件，进行项目编译有两种方式：通过命令编译, 通过CMake编译，后者是全自动化也不需要 gcc 基础，推荐使用。

目录结构

```shell
Test2
	include
		max.h
	src
		max.cpp
	test.cpp
```

max.h

```c++
#ifndef MAX_H__
#define MAX_H__

#include <stdio.h>

int findMaxNum(int num1, int num2);


#endif // MAX_H__
```

max.cpp

```c++
#include "max.h"

int findMaxNum(int num1, int num2)
{
    return (num1 > num2 ? num1 : num2);
}
```

test.cpp

```c++
#include <stdio.h>
#include <iostream>
#include "include/max.h"

int main()
{
    int a = 10;
    printf("%d\n", a);
    int b = 20;
    printf("%d\n", b);
    int c = findMaxNum(a, b);
    printf("%d\n", c);

    
    return 0;
}
```

### 2.1 基于命令编译项目

首先打开一个终端（现有终端或者新的终端都可以），在 Test2目录执行如下命令编译源文件

```shell
# C++程序编译 多目录的话用空格隔开即可  
# 有时候可能需要在链接时添加 -mconsole 选项，指定生成一个控制台程序，而不是 GUI 程序。
g++ src/max.cpp test.cpp -o test -I ./include 
g++ src/max.cpp test.cpp -o test -I ./include -mconsole

# 如果第一次有报错的话，可能是没保存，去头文件和源文件下按一下Ctrl + S 之后再试试

# 编译 src 目录下所有的 .cpp 文件
g++ src/*.cpp test.cpp -o test -I ./include
```

这样就生成了可执行程序 test.exe。

### 2.2 基于CMake编译

首先在源文件所在的工作区目录中添加一个CMake文件叫做 CMakeLists.txt，在这个文件中添加两句话

```cmake
# project() ：设置项目名称，参数可以随意指定
# aux_source_directory(dir VAR): 搜索 dir 目录下所有的源文件，并将结果列表存储在变量 VAR 中
# add_executable(target src): 指定使用源文件src，生成可执行程序 target , ${变量名} 是取变量的值,可以多个目录，目录之间加空格
# include_directories(headDir): 设置包含的头文件目录,可以多个目录，目录之间加空格

cmake_minimum_required(VERSION 3.1)
project(TestMake)                          
aux_source_directory(src SRC_SUB)           
aux_source_directory(. SRC_CUR)      
include_directories(include)       
add_executable(test ${SRC_SUB} ${SRC_CUR}) 
```

在vscode中先配置cmake, 按快捷键 **ctrl+shift+p**，在窗口中搜索 **CMake configure**，选中这个配置项。

如果vscode 加载不到编译器，会弹出如下窗口，需要选择 MinGW 套件中的 GCC 编译器，不要选 VS里边的编译器。


CMake 配置完成之后, 会在vscode 工作区生成一个 build目录

然后，打开终端，执行下面三条命令：

```shell
进入到生成的build目录
cd build

通过cmake工具基于编写的CMakeLists.txt生成MakeFile文件
cmake ..
 
使用make工具通过生成的makefile文件构造项目。Linux中的构建工具叫make, 32位的MinGW中的构建工具叫做mingw32-make.exe, 64位的MinGW中的构建工具叫做mingw64-make.exe, 当前使用的构建工具是32位的。最终可执行程序就生成到build目录中了。
mingw32-make.exe
```

#### 现在的目录结构

```shell
Test2
	build
	include
		max.h
	src
		max.cpp
	Cmakelists.txt
	test.cpp
```

### 2.3 基于CMake进行配置并调试

# ！！！我试了几次都没成功，但是看博主又成功了，不知道是哪里出了问题。可直接使用第三种方式

#### 1、创建launch.json文件，示例：

```json
// 这个配置文件中需要修改的项不太多, 介绍一下需要修改的配置项:

// program: 要调试的可执行程序的路径，里边可以使用一些宏，宏的外部加 ${} 表示取值
//     ${fileDirname}：文件目录的名字，launch.json 对应的目录名就是 .vscode
//     ${fileBasenameNoExtension}：不带扩展名的文件名，文件名是main函数对应的那个文件
//     ${workspaceFolder}：工作区目录
// preLaunchTask：调试项目前要执行的任务，C/C++: g++.exe 生成活动文件是tasks.json中的一个任务
//     通过执行这个任务生成了program对应的可执行文件

{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++.exe - 生成和调试活动文件",
            "type": "cppdbg",
            "request": "launch",
            // 这个路径必须跟task.json下的"cwd"的路径相同
            // 可执行程序的名test.exe必须要跟CMakeLists文件里的 add_executable(test ${SRC_SUB} ${SRC_CUR})  命名一致
            "program": "${workspaceFolder}/build/test.exe",
            "args": [],
            "stopAtEntry": false,
            //"cwd": "D:\\Qt5.15.2\\Tools\\mingw810_64\\bin",
            "cwd": "${workspaceFolder}",
            "environment": [],
            //"console": "externalTerminal",
            //"console": "integratedTerminal",
            "MIMode": "gdb",
            "miDebuggerPath": "D:\\Qt5.15.2\\Tools\\mingw810_64\\bin\\gdb.exe",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "Build my project"  // 需要跟task.json文件的对应的Label名一致
        }
    ]
}


```

#### 2、创建tasks文件，示例：

```json
{
    "version": "2.0.0",
    "options": {
        // cwd：进入到工作区的子目录 build中
        // 这个路径必须要和launch.json文件下的"program"下的路径一致
        "cwd": "${workspaceFolder}/build/"
    },

    "tasks": [
        {
            // label: 标题, 可随意取名, 这里叫做 cmake
            "label": "cmake",
            "type": "shell",
            // command: 要执行的命令是 cmake .. , 作用是使用 cmake 生成 makefile 文件
            "command": "cmake",
            "args": [
                ".."
            ]
        },
        {
            // label: 标题, 可随意取名, 这里叫做 make
            "label": "make",
            "group":{
                "kind":"build",
                "isDefault":true
            },
            // command: 要执行的命令是 mingw32-make.exe作用是使用 MinGW 编译套件中的这个工具基于 makefile 构建当前项目
            "command": "mingw32-make.exe",
            "args":[
            ]
        },
        {
            // label: 标题, 可以随意取名, 这里叫 Build my project。该名字需要作为 launch.json中 preLaunchTask配置项的值
            "label":"Build my project",
			// dependsOn: 需要执行的命令对应的 label,，按顺序执行
			"dependsOrder": "sequence",
			"dependsOn": [
				"cmake",
				"make"
			]
        }
    ]
}
```

#### 3、最后，打断点，按F5进行调试即可



## 三、多文件调试——直接点击VSCode左下角的按钮

Windows试了很多次，就这样最简单且不容易出错，上述的方法在Windows下总是出错，无语了已经

首先就是写好头文件和源文件，最重要的是要写好**CMakeLists.txt文件**，然后打断点，最后直接点击下图所示的按钮置即可

目录结构

```shell
Test2
	include
		max.h
	src
		max.cpp
	test.cpp
	CMakeLists.txt
```

如下图所示

![image-20241118110914968](VScode%20%E8%B0%83%E8%AF%95%E2%80%94%E2%80%94Windows+Linux.assets/image-20241118110914968-1733218754391-2-1733218756446-4-1733219333405-1-1733219334476-3.png)

然后，VSCode会自己生成一个build文件夹，并自己执行cmake，make等操作，调试到断点位置，他会自己输出



# Linux:

## 一、单文件

同上Windows即可，差不多

## 二、多文件调试

首先，写好源文件和头文件，放到对应的目录下，然后写好CMakeLists.txt文件。

```shell
进入终端（Ctrl+`)
mkdir build
cd build
cmake ..
make 

目录结构：
.
├── build
├── CMakeLists.txt
├── include
│   └── *.h
├── main.cpp
└── src
    └── *.cpp

可以执行之后，开始配置json文件
1、点击左侧的“运行和调试”按钮（Ctrl_shift+D)，创建launch.json文件，点击右下角的“添加配置”按钮，生成一个模板文件，具体配置看下面
2、点击上面菜单栏的“终端”按钮，在下拉菜单中选择“配置默认生成任务”，选择自己的编译器啥的，创建tasks.json文件，具体配置看下面
```

### 2.1 CMakeLists.txt文件：

```cmake
# 指定CMake最低版本要求
cmake_minimum_required(VERSION 3.1)
# 指定一个工程名字，可随意
project(MySwap)
# 指定头文件搜索目录
include_directories(include)  
# 将src文件夹下的所有源文件都赋值给变量 SRC_SUB                          
aux_source_directory(src SRC_SUB)      
# 将当前文件夹下的所有源文件都赋值给变量 SRC_CUR     
aux_source_directory(. SRC_CUR)     
# 设置构建类型是Debug，这样才带有调试信息，才可以直接F5调试
set(CMAKE_BUILD_TYPE Debug) 
# 将 SRC_SUB 和 SRC_CUR 下的源文件生成一个可执行文件，名字为main_cmake
add_executable(main_cmake ${SRC_SUB} ${SRC_CUR}) 
```

### 2.2 launch.json文件

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) 启动",
            "type": "cppdbg",
            "request": "launch",
            // 这个路径必须跟task.json下的"cwd"的路径相同
            // 可执行程序的名main_cmake.exe必须要跟CMakeLists文件里的
            // add_executable(main_cmake ${SRC_SUB} ${SRC_CUR})  命名一致
            "program": "${workspaceFolder}/build/main_cmake",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
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
            ],
            // 需要跟task.json文件的对应的Label名一致
            "preLaunchTask": "Build my project",
            "miDebuggerPath": "/usr/bin/gdb"
        }

    ]
}
```

### 2.3 tasks.json文件

```json
{
    "version": "2.0.0",
    "options": {
        // cwd：进入到工作区的子目录 build中
        // 这个路径必须要和launch.json文件下的"program"下的路径一致
        "cwd": "${workspaceFolder}/build/"
    },

    "tasks": [
        {
            // label: 标题, 可随意取名, 这里叫做 cmake
            "label": "cmake",
            "type": "shell",
            // command: 要执行的命令是 cmake .. , 作用是使用 cmake 生成 makefile 文件
            "command": "cmake",
            "args": [
                ".."
            ]
        },
        {
            // label: 标题, 可随意取名, 这里叫做 make
            "label": "make",
            "group":{
                "kind":"build",
                "isDefault":true
            },
            // command: 要执行的命令是 make 作用是使用 MinGW 编译套件中的这个工具基于 makefile 构建当前项目
            "command": "make",
            "args":[
            ]
        },
        {
            // label: 标题, 可以随意取名, 这里叫 Build my project。该名字需要作为 launch.json中 preLaunchTask配置项的值
            "label":"Build my project",
            // dependsOn: 需要执行的命令对应的 label
            "dependsOrder": "sequence",
            "dependsOn":[
                "cmake",
                "make"                
            ]
        }
    ]
}
```

### 2.4 回到main.cpp文件，打断点，然后按F5开始调试

