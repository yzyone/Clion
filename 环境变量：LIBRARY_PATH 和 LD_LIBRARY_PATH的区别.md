
# 环境变量：LIBRARY_PATH 和 LD_LIBRARY_PATH的区别 #

添加环境变量：`https://www.cnblogs.com/lovychen/p/5583703.html`

PATH是可执行文件的环境变量。

LIBRARY_PATH 和 LD_LIBRARY_PATH 是Linux下的两个环境变量，二者的含义和作用分别如下：

两者的共同点是库，库是这两个路径和PATH路径的区别；

LIBRARY_PATH环境变量用于在程序编译期间查找动态链接库时指定查找共享库的路径；

例如，指定gcc编译需要用到的动态链接库的目录。设置方法如下（其中，LIBDIR1和LIBDIR2为两个库目录）：

export LIBRARY_PATH=LIBDIR1:LIBDIR2:$LIBRARY_PATH
LD_LIBRARY_PATH环境变量用于在程序加载运行期间查找动态链接库时指定除了系统默认路径之外的其他路径；

注意，LD_LIBRARY_PATH中指定的路径会在系统默认路径之前进行查找。

设置方法如下（其中，LIBDIR1和LIBDIR2为两个库目录）：

export LD_LIBRARY_PATH=LIBDIR1:LIBDIR2:$LD_LIBRARY_PATH
举个例子，我们开发一个程序，经常会需要使用某个或某些动态链接库，为了保证程序的可移植性，可以先将这些编译好的动态链接库放在自己指定的目录下，然后按照上述方式将这些目录加入到LD_LIBRARY_PATH环境变量中，这样自己的程序就可以动态链接后加载库文件运行了。

**区别与使用：**

开发时，设置LIBRARY_PATH，以便gcc能够找到编译时需要的动态链接库。

发布时，设置LD_LIBRARY_PATH，以便程序加载运行时能够自动找到需要的动态链接库。