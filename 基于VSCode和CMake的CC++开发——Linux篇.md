# 基于VSCode和CMake的C/C++开发——Linux篇

## 一、

GCC 编译器对程序的编译分为 4 个阶段：预处理（预编译）、编译和优化、汇编和链接。GCC 的编译器可以将这 4 个步骤合并成一个。

1、预处理: 在这个阶段主要做了三件事: 展开头文件 、宏替换 、去掉注释行
		这个阶段需要GCC调用预处理器来完成, 最终得到的还是源文件, 文本格式

```shell
# 1. 预处理, -o 指定生成的文件名
gcc -E test.c -o test.i
```

2、编译: 这个阶段需要GCC调用编译器对文件进行编译, 最终得到一个汇编文件

```shell
# 2. 编译, 得到汇编文件
gcc -S test.i -o test.s
```

3、汇编: 这个阶段需要GCC调用汇编器对文件进行汇编, 最终得到一个二进制文件

```shell
# 3. 汇编
gcc -c test.s -o test.o
```

4、链接: 这个阶段需要GCC调用链接器对程序需要调用的库进行链接, 最终得到一个可执行的二进制文件

```shell
# 4. 链接
gcc test.o -o test
```

## 二、g++重要参数

####  -g  

编译带调试信息的可执行文件，即告诉GCC产生能被GNU调试器使用的调试信息，以调试程序

```shell
  # 产生带调试信息的可执行文件test
  g++ -g test.cpp -o test   
```

####   -O[n] 

优化源代码，一般选择-O2就够了

```shell
  # 使用 -O2优化源代码，并输出可执行文件
  g++ -O2 test.cpp -o test
```

####   -l 和 -L

 指定库文件 | 指定库文件路径

```shell
  # -l参数就是用来指定程序要链接的库，-l参数紧接着的就是库名。注意：只有在/lib，/usr/lib和/usr/local/lib里面的库才可以直接用-l直接链接，其他目录下的需要使用-L指定
  # 链接glog库
  g++ -lglog test.cpp
  # 链接 home/jl55/myTest -lmytest 目录下的mytest库，
  g++ -L/home/jl55/myTest -lmytest test.cpp 
```

####   -I(大写的i）

指定头文件搜索目录

```shell
  # /usr/include目录一般是不需要指定的
  g++ -I/myinclude test.cpp
```

####  -Wall 

打印警告信息

####  -w 

关闭警告信息

```shell
  # 打印出gcc提供的警告信息
  g++ -Wall test.cpp
```

####   -std=c++11  

设置编译标准

```shell
  # 使用C++17 标准编译test.cpp
  g++ -std=c++17 test.cpp
```

####   -o 

指定输出文件名，不指定的话，默认输出是a.exe

```shell
  # 指定输出的可执行文件名为test，test.exe
  g++ test.cpp -o test
```

####   -D 

 定义宏  在使用gcc/g++编译的时候定义宏

```shell
  # 常用场景： -DDEBUG  定义DEBUG宏，可能文件中又DEBUG宏部分的相关信息，用这个DEBUG宏来选择开启或关闭DEBUG

  示例代码：
  // -Dname 定义宏name，默认定义内容为字符串“1”
  #include <stdio.h>
  int main()
  {
  	#ifdef DEBUG
  		printf("DEBUG LOG\n");
  	#endif
  		printf("\n");
  }
  // 在编译的时候，使用g++ -DDEBUG main.cpp  printf("DEBUG LOG\n");这句代码会被执行
```

## 三、制作库文件

// 这个测试目录现在被我移动了

GCC 编译器对程序的编译分为 4 个阶段：预处理（预编译）、编译和优化、汇编和链接。GCC 的编译器可以将这 4 个步骤合并成一个。

1、预处理: 在这个阶段主要做了三件事: 展开头文件 、宏替换 、去掉注释行
		这个阶段需要GCC调用预处理器来完成, 最终得到的还是源文件, 文本格式

```shell
# 1. 预处理, -o 指定生成的文件名
gcc -E test.c -o test.i
```

2、编译: 这个阶段需要GCC调用编译器对文件进行编译, 最终得到一个汇编文件

```shell
# 2. 编译, 得到汇编文件
gcc -S test.i -o test.s
```

3、汇编: 这个阶段需要GCC调用汇编器对文件进行汇编, 最终得到一个二进制文件

```shell
# 3. 汇编
gcc -c test.s -o test.o
```

4、链接: 这个阶段需要GCC调用链接器对程序需要调用的库进行链接, 最终得到一个可执行的二进制文件

```shell
# 4. 链接
gcc test.o -o test
```

#### 目录结构：

```shell
jl55@jl55-virtual-machine:~/Test$ tree
.
├── include
│   └── swap.h
├── main.cpp
└── src
    └── swap.cpp

3 directories, 5 files
```

### 3.1 制作静态库

```shell
# 进入src目录
cd src 

# 汇编，生成swap.o文件
g++ swap.cpp -c -I../include

# 生成静态库libswap.a
ar rs libswap.a swap.o

# 回到上级目录
cd ..

# 链接，生成可执行文件：static_main
g++ main.cpp -Iinclude -Lsrc -lswap -o static_main

# 执行完之后，生成一个static_main的可执行文件   ./static_main之后就可以执行
jl55@jl55-virtual-machine:~/Test$ ls
include  main.cpp  static_main  src
jl55@jl55-virtual-machine:~/Test$ ./static_main 
交换之前：
val1 = 10
val2 = 20
交换之后：
val1 = 20
val2 = 10
```

### 3.2 制作动态库

```shell
# 进入src目录
cd src

# 生成动态库libswap.so
g++ swap.cpp -I../include -fPIC -shared -o libswap.so

## 上面的命令等价于下面的两条命令  -fPIC 或 -fpic 参数的作用是使得 gcc 生成的代码是与位置无关的，也就是使用相对位置。
# gcc swap.cpp -I../include -c -fPIC
# gcc -shared -o libswap.so swap.o

# 回到上级目录
cd ..

# 链接，生成可执行文件 share_main
g++ main.cpp -Iinclude -Lsrc -lswap -o share_main
jl55@jl55-virtual-machine:~/Test$ g++ main.cpp -Iinclude -Lsrc -lswap -o share_main

# 执行完之后，生成一个sharemain 的文件 
jl55@jl55-virtual-machine:~/Test$ ls
include  main.cpp  share_main  src  static_main

# ./sharemain 不可以直接这样执行，他会找不到这个动态库。我们需要指定这个动态库的目录，如果这个动态库目录在usr下的话，可能就不会出错了，
jl55@jl55-virtual-machine:~/Test$ ./share_main 
./share_main: error while loading shared libraries: libswap.so: cannot open shared object file: No such file or directory
jl55@jl55-virtual-machine:~/Test$ LD_LIBRARY_PATH=src ./share_main 
交换之前：
val1 = 10
val2 = 20
交换之后：
val1 = 20
val2 = 10
```

## 四、GDB调试，常用命令

```shell
GDB调试，常用命令

# 开始GDB调试,test为可执行文件，且包含调试信息
gdb test
# 进入GDB命令
(gdb) help(h)    		# 查看命令帮助，具体命令查询在gdb中输入help+命令
(gdb) run(r)    		# 重新开始运行文件(run-text:加载文本文件，run-bin:加载二进制文件)
(gdb) start    			# 单步执行，运行程序，停在第一行执行语句
(gdb) list(l)			# 查看原代码(1ist-n,从第n行开始査看代码。1ist+ 函数名:查看具体函数)
(gdb) set 				# 设置变量的值
(gdb) next(n)			# 单步调试(逐过程，函数直接执行)
(gdb) step(s)			# 单步调试(逐语句:跳入自定义函数内部执行)
(gdb) backtrace(bt)		# 查看函数的调用的栈帧和层级关系
(gdb) frame(f) 			# 切换函数的栈帧
(gdb) info(i)			# 查看函数内部局部变量的数值
(gdb) finish			# 结束当前函数，返回到函数调用点
(gdb) continue(c)		# 继续运行
(gdb) print(p)			# 打印值及地址
(gdb) break+num(b)	# 在第num行设置断点
(gdb) info breakpoints	#查看当前设置的所有断点
(gdb) delete breakpoints num(d)		# 删除第num个断点I
(gdb) display			# 追踪查看具体变量值
(gdb) undisplay		#取消追踪观察变量
(gdb) watch			#被设置观察点的变量发生修改时，打印显示
(gdb) i watch			#显示观察点
(gdb) enable breakpoints	#启用断点
(gdb) disable breakpoints	#禁用断点
(gdb) x				#查看内存x/20xw 显示20个单元，16进制，4字节每单元
(gdb) run argv[1] argv[2]	# 调试时命令行传参
(gdb) set fo11ow-fork-mode child		#Makefi1e项目管理:选择跟踪父子进程(fork())
```

## 五、Linux VSCode 的使用

### 5.1 安装

```shell
# 安装
sudo snap install --classic code
# 打开，先cd到指定目录之后 code .  即可打开
code . 
```

### 5.2 插件安装

```shell
安装插件，
必要：C/C++， Cmake，Cmake Tools
选装：C/C++ Extension Pack，Chinese（Simplified）

后面这几个似乎也没什么必要
1、编程美化
Bracket Pair Colorizer
给匹配的括号上色，可以自定义配置
注意：该插件已经内置到 vscode，不用重复安装，
设置方法：setting 里搜索 editor.bracketPairColorization.enabled，设置为 true 即可生效
2、代码风格统一
EditorConfig for VS Code
3、自动格式化代码
Prettier - Code formatter
```

### 5.3 小tips:

```shell
1、字体不舒服设置，刚进来VSCode之后，字体可能不是太好，空格间距太小，还有就是回车换行之后只有两个空格的间距。
解决方法：进入设置（右上角的三个点->配置编辑器）-->常用设置-->Editor: Font Family->consolas, 'Courier New', monospace
将原来的格式删除替换成consolas, 'Courier New', monospace，字体看着也舒服，间隔也合适。
2、像VS那样，分号打上之后自动格式化代码，同上进入设置，搜索 Editor: Format On Type，打上对钩即可。
3、自动保存格式化代码：同2，进入设置，找到Format On Save，打钩即可。
不然，有时候你写代码没有手动保存，去编译会报错
4、引入自定义头文件的时候总是会报错，先Ctrl + S保存一下就好了。可能是因为json文件还没有配置好， 


写完之后之后打开终端，快捷键：Ctrl + `(tab键上方的那个按键）
```

## 六、CMake

### 6.1 基本语法

```cmake
基本语法格式：指令（参数1 参数2）
	参数使用小括号括起来，参数之间使用空格或者分号分开
	指令是大小写无关的，参数和变量是大小写相关的
	变量使用 ${} 的方式来取值的，但是在IF控制语句中是直接使用变量名的
```

### 6.2 CMake重要指令

#### 1、cmake_minimum_required——指定CMake的最小版本要求

```cmake
语法：cmake_minimum_required(VERSION versionNumber)

# CMake的最小版本要求为2.8.3
cmake_minimum_required(VERSION 2.8.3)
```

#### 2、project——定义工程名称，并可指定工程支持的语言

```cmake
语法：project(projectName [CXX][C][Java])

# 指定工程名字为HELLOWORLD
project(HELLOWORLD)
```

#### 3、set——显示的定义变量

```cmake
语法：set(变量名 变量1 变量2)

# 定义SRC变量，其值为sayHello.cpp hello.cpp
set(SRC sayHello.cpp hello.cpp)
```

#### 4、include_directories——向工程添加多个特定的头文件搜索路径（相当于指定g++编译器的 -I 参数）

```cmake
语法：include_directories(Path1 Path2)

# 将/usr/include/myInclude 和 ./include 添加到头文件的搜索路径
include_directories(/usr/include/myInclude ./include)
```

#### 5、link_directories——向工程添加多个特定的库文件搜索路径（相当于指定g++编译器的 -L 参数）

```cmake
语法：link_directories(Path1 Path2)

# 将/usr/lib/myLib 和 ./lib 添加到头文件的搜索路径
link_directories(/usr/lib/myLib ./lib)
```

#### 6、add_library——生成库文件

```cmake
语法：add_library(库名 动态库[SHARED]/静态库[STATIC] 源文件1 源文件2)

# 通过变量SRC 生成libhello.so的动态库
add_library(hello SHARED ${SRC})
```

#### 7、add_compile_options——添加编译参数

```cmake
语法：add_compile_options(<option>...)

# 添加编译参数 -Wall -std=c++11 -O2
add_compile_options(-Wall -std=c++11 -O2)
```

#### 8、add_executable——生成可执行文件

```cmake
语法：add_executable(可执行文件名 源文件1 源文件2...)

# 将main.cpp 生成可执行文件main
add_executable(main main.cpp)
```

#### 9、target_link_libraries——为target添加需要链接的共享库（相当于指定g++编译器的 -l 参数）

```cmake
语法：target_link_libraries(target library1 library2 Value)指定库文件或者变量也可以

# 将hello动态库链接到可执行文件main
target_link_libraries(main hello)
```

#### 10、add_subdirectory——向当前工程添加存放源文件的子目录，并可以指定中间二进制和目标二进制存放的位置

```cmake
语法：add_subdirectory(源文件 [二进制目录] )

# 添加src子目录，src中需要有一个CMakeLists.txt
add_subdirectory(src)
```

#### 11、aux_source_directory——发现一个目录下所有的源代码文件并将列表存储在一个变量中，这个指令临时被用来自动构建源文件列表4

```cmake
语法：aux_source_directory(目录 变量名)

# 定义SRC变量，其值为当前目录下所有源文件代码
aux_source_directory(. SRC)
# 编译SRC变量所代表的源文件，生成main可执行文件
add_executable(main ${SRC})
```

### 6.3 CMake 常用变量

#### 1、编译选项

```cmake
CMAKE_C_FLAGS   	   	gcc编译选项
CMAKE_CXX_FLAGS 		g++编译选项
```

#### 2、在CMAKE_CXX_FLAGS编译选项后追加-std=c++11
```cmake
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11)
```

#### 3、CMAKE_BUILD_TYPE  	编译类型（Debug，Release）

```cmake
# 设定编译类型为Debug，测试时需要选择Debug
set(CMAKE_BUILD_TYPE Debug)

# 设定编译类型为Release，测试时需要选择Release
set(CMAKE_BUILD_TYPE Release)
```

## 七、VSCode 调试——配置CMakeLists.txt文件和json文件

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

### 7.1 CMakeLists.txt文件：

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

### 7.2 launch.json文件

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

### 7.3 tasks.json文件

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

### 7.4 回到main.cpp文件，打断点，然后按F5开始调试

