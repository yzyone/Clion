
# CMake 自动检测系统编译器是否支持C++11 #

在 CMakeLists.txt 中加入以下代码, 可以自动判断系统编译器是否支持c++11标准:

```
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()
```

作者：魏华祎

链接：https://www.jianshu.com/p/d71f300cca0a

来源：简书

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。