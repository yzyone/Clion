CMake交叉编译简单教程

Shely2017 2018-09-07 16:21:18  18025  收藏 25
分类专栏： cmake
版权
首先要安装cmaek

然后安装交叉编译链

一、CMake简介：

CMake是一个跨平台的安装(编译)工具,可以通过简单的语句来描述所有平台的安装(编译过程)。他能够输出各种各样的makefile或者project文件。

 

二、CMake命令

CMake的语句都写在一个叫做CMakeLists.txt的文件里。常见的CMake内置变量和命令如下：

https://blog.csdn.net/wzzfeitian/article/details/40963457/

 

查看已安装好的cmake版本，我安装的是3.11.1版本



三、具体编译使用

(1)在atest/t1下写一个简单的main程序和对应的CMakeLists.txt文件。

Main.c内容如下：

    #include<stdio.h>
    
    int main()
    
    {
    printf("hello from t1 mian!\n");
    
    return 0;
    
    }

 

CMakeLists.txt内容如下：

    CMake_minimum_required(VERSION 3.11)
    
    PROJECT(HELLO)
    
    SET(SRC_LIST main.c)
    
    ADD_EXECUTABLE(hello ${SRC_LIST})

 

进行编译：

cmake .   //指定的是CMakeLists.txt所在目录

make

./hello

 



验证，结果正确。



 

此时t1文件夹中多了几个文件：



也可以查看hello的属性



 

(2)t1中默认生成的是x86，我们也可以自己添加语句来选择

建立atest/t2文件夹，main.c和CMakeLists.txt文件，main.c文件内容不变

CMakeLists.txt如下

 

    SET(CMAKE_SYSTEM_NAME Linux)
    
    SET(CMAKE_C_COMPILER /home/amm/software/arm-2014.05/bin/arm-none-linux-gnueabi-gcc)
    
    SET(CMAKE_CXX_COMPILER /home/amm/software/arm-2014.05/bin/arm-none-linux-gnueabi-c++)
    
     
    
    PROJECT(HELLO)
    
    SET(SRC_LIST main.c)
    
    ADD_EXECUTABLE(hello ${SRC_LIST})

 



进行编译

    cmake .
    
    make

 



此时文件夹内容



再次查看hello属性



 

(3)t1 和 t2 都是内部编译，我们其实是不希望编译文件与源文件混在一起的，故建立build文件夹，使编译都在build文件夹内进行

    #include<stdio.h>
    
//main.c
    
    int main()
    
    {
    printf("hello from t3 mian!\n");
    
    return 0;
    
    }

//CMakeLists.txt
    
    CMake_minimum_required(VERSION 3.11)
    
    PROJECT(HELLO)
    
    SET(SRC_LIST main.c)
    
    ADD_EXECUTABLE(hello ${SRC_LIST})



 

此时进入build目录下进行编译



此时目录情况



Build中：



达到目的。

 (4)当有多个文件时main.c hello.c hello.h

//main.c

    #include "hello.h"
    
    int main()
    
    {
    hello("World");
    
    return 0;
    
    }

// hello.c

    #include <stdio.h>
    
    #include "hello.h"
    
     
    
    void hello(const char* name)
    
    {
    printf("HELLO %s!\n",name);
    
    }

//hello.h

    #ifndef AMM_HELLO_
    
    #define AMM_HELLO_
    
    void hello(const char* name);
    
    #endif

//CMakeLists.txt

    project(A2)
    
    set(SRC_LIST main.c hello.c)
    
    add_executable(A2 ${SRC_LIST})



编译过程





(5)将hello生成库文件，只需要将CMakeLists.txt文件改一下：

    project(A3)
    
    set(LIB_SRC hello.c)
    
    set(APP_SRC main.c)
    
    add_library(hello ${LIB_SRC})
    
    add_executable(A3 ${APP_SRC})
    
    target_link_libraries(A3 hello)

编译过程一样，可以看到在build文件夹中多了.a静态库文件



(6)将源文件、库文件分别放入src和libhello不同文件夹中，并将生成的可执行文件放入build/bin中，生成的库文件放入build/lib中



需要写三个CMakeLists.txt文件，A5中，libhello中，src中

// A5中的CMakeLists.txt

    CMake_minimum_required(VERSION 3.11)
    
    project(A5)
    
     
    
    add_subdirectory(src)
    
    add_subdirectory(libhello)
    
    // libhello中的CMakeLists.txt
    
    set(LIB_SRC hello.c)
    
    add_library(libhello ${LIB_SRC})
    
    set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
    
    set_target_properties(libhello PROPERTIES OUTPUT_NAME "hello")
    
    // src中的CMakeLists.txt
    
    include_directories(${PROJECT_SOURCE_DIR}/libhello)
    
    set(APP_SRC main.c)
    
    set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
    
    add_executable(A5 ${APP_SRC})
    
    target_link_libraries(A5 libhello)

进行编译





观察后相应文件在相应设定的文件夹中



此时生成的是.a静态库，如果想生成动态库，则改动一下libhello中CMakeLists.txt文件



将第二行中加入SHARED就生成的是动态库了



四、其他

例如如何加入第三方库，没找到合适的例子



通过上边的语句应该可以达到目的。

 

后来补充：

例如：

有现成的lib库为libqueue.a和其对应的头文件queue.h



建立main文件，和写对应的CMakeLists.txt文件

//main.c
    
    #include "queue.h"
    #include <stdio.h>
    #include <stdlib.h>
     
     
    int main()
    {
    sp_queue q = init();
    void* x = (void*)"abc";
     
    //void* x = (void*)99;
     
     
    push(&q,x);
    printf("入队元素为：%s\n",x);
     
    printf("队列长度：%d\n",size(q));
     
     
    pop(&q, &x);//出队列
    printf("出队：%s\n", x);
     
     
    return 0;
    }

//CmakeLists.txt
    
    cmake_minimum_required(VERSION 3.11)
    
     
    
    project(testQ)
    
    set(INK_DIR /home/amm/atest/testQ)
    
    set(LINK_DIR /home/amm/atest/testQ)
    
    set(SOURCE_FILES main.c)
    
     
    
    include_directories(${INK_DIR})
    
    link_directories(${LINK_DIR})
    
    add_executable(testQ main.c)
    
    target_link_libraries(testQ libqueue.a)

 

2.进入终端，在空文件夹build下：

    Cmake ..
    
    Make
    
    ./testQ

 



可以发现，成功。