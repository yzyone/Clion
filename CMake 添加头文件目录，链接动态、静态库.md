
### CMake 添加头文件目录，链接动态、静态库（添加子文件夹）

---

CMake支持大写、小写、混合大小写的命令。

当编译一个需要第三方库的项目时，需要知道：

去哪找头文件（.h），-I（GCC）

	INCLUDE_DIRECTORIES()

去哪找库文件（.so/.dll/.lib/.dylib/...），-L（GCC）

	LINK_DIRECTORIES()

需要链接的库文件的名字：-l（GCC）

	LINK_LIBRARIES(库名称即可)

1. 添加头文件目录

	INCLUDE_DIRECTORIES

它相当于 g++ 选项中的 -I 参数的作用，也相当于环境变量中增加路径到 CPLUS_INCLUDE_PATH 变量的作用：

	include_directories(../../../thirdparty/comm/include)

2. 添加需要链接的库文件目录

	LINK_DIRECTORIES

它相当于 g++ 命令的 -L 选项的作用，也相当于环境变量中增加 LD_LIBRARY_PATH 的路径的作用

	link_directories("/home/server/third/lib")

3. 宏定义

CMakeLists.txt 之 多重判断宏定义

比如设置为 CPU_ONLY

	add_definitions(-DCPU_ONLY)

它相当于 g++ 命令的 -D 选项的作用（-DCPU_ONLY），用于宏定义。

4. 添加子文件夹

使用 add_subdirectory

	add_subdirectory(Foundation_Classes)
	add_subdirectory(Behavioral_Patterns)
	add_subdirectory(Creational_Patterns)
	add_subdirectory(Structural_Patterns)

References

[CMake学习-添加头文件路径，库路径，库](http://blog.csdn.net/snail_hunan/article/details/70238478)

[cmake 添加头文件目录，链接动态、静态库](http://www.cnblogs.com/binbinjx/p/5626916.html)

转载于:https://www.cnblogs.com/mtcnn/p/9422156.html