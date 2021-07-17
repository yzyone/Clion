Clion配置MSYS2通过Makefile编译工程

配置MSYS2

下载安装

MSYS2官方网站：`http://www.msys2.org/`

下载地址：`http://repo.msys2.org/distrib/x86_64/msys2-x86_64-20180531.exe`

安装没什么问题，按提示下一步即可。

pacman 修改软件源：

官方说明：`https://mirrors.tuna.tsinghua.edu.cn/help/msys2/`

编辑 `/etc/pacman.d/mirrorlist.mingw32` ，在文件开头添加：

    Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/i686

编辑 `/etc/pacman.d/mirrorlist.mingw64` ，在文件开头添加：

    Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/x86_64

编辑 /etc/pacman.d/mirrorlist.msys ，在文件开头添加：

    Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/msys/$arch

然后执行 `pacman -Sy` 刷新软件包数据即可。

执行    

    pacman-key --init

    pacman -Syu 

更新软件，更新过程中可能要求重新打开终端，另外更新的时候，pacman的软件源可能恢复到默认，需要手动再修改一次。

安装工具链

    pacman -S mingw-w64-x86_64-cmake mingw-w64-x86_64-extra-cmake-modules
    pacman -S mingw-w64-x86_64-make
    pacman -S mingw-w64-x86_64-gdb
    pacman -S mingw-w64-x86_64-toolchain

安装Clion

clion官网：`https://www.jetbrains.com/clion/`

安装没什么，根据提示下一步即可。

如需破解，请自行百度。

配置MingW环境


创建工程


main.c

```
#include <stdio.h>
 
int main() {
    printf("Hello, World!\n");
    return 0;
}
```

Makefile

```
all: MakeTest
 
MakeTest: main.c
	gcc main.c -o make_test
```

CMakeLists.txt

```
cmake_minimum_required(VERSION 3.10)
project(MakeTest C)
 
set(CMAKE_C_STANDARD 99)
 
#add_executable(MakeTest main.c)
 
 
add_custom_target(MakeTest COMMAND mingw32-make -C ${MakeTest_SOURCE_DIR})
```

编译和运行
