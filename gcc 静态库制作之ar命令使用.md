
# gcc 静态库制作之ar命令使用 #

**前言**

我们通常把一些公用函数制作成函数库，供其他程序使用。函数库分为静态库和动态库两种。本文讲解如何制作属于自己的静态库。

**什么是静态库？**

通常来说，静态库以.a作为后缀，且以lib开头。类似于libxxx.a。静态库在程序编译时会被连接到目标代码中，程序运行时将不再需要该静态库。

**ar命令详解**

Linux ar命令用于创建或者操作静态库。

ar命令的参数如下:

参数	意义

- -r	将objfile文件插入静态库尾或者替换静态库中同名文件
- -x	从静态库文件中抽取文件objfile
- -t	打印静态库的成员文件列表
- -d	从静态库中删除文件objfile
- -s	重置静态库文件索引
- -v	显示详细信息
- -c	创建静态库文件

**制作静态库**

test.c

```
#include <stdio.h>
#include "test.h"


void test(){

	printf("This is a static library\n");

}
```

test.h

```
#define __TEST_H__
#ifndef __TEST_H__

void test();

#endif
```


编译成可重定位文件，即生成.o文件：

gcc -c test.c

为了制作成静态库，我们需要使用ar命令。

```
ar -rcs libtest.a test.o   #库名一般以.a为扩展名，以lib开头
ar -t libtest.a  #查看内容
test.o
```

输出信息可以看到，静态库以.a作为后缀，且以lib开头，这时候就制作好了自己的静态库了。

制作好了静态库，下面来使用它。

**静态库的使用**

编写一个main.c文件进行测试：


main.c

```
#include <stdio.h>
#include "test.h"

int main(int argc, char const *argv[])
{
	test();
	
	return 0;
}
```


出现信息为test未定义引用，原因是test已经编译成静态库。

解决办法为：


静态库的代码在编译时链接到应用程序中，因此编译时库文件必须存在，并且需要通过"-L"参数传递路径给编译器。

链接的库名为libtest.a，在链接的时候，去掉开头的lib和后缀.a，前面再加l，就变成了-ltest，其他库也是类似。

例如，你如果看到程序链接使用-lm，说明它使用了名为libm.a的库。可以参考这一篇在编译时时为什么要链接 -lm

总结

编译静态库时先使用-rcs选项，再利用ar工具产生，然后把一些文件可重定位文件打包在一起。

————————————————

版权声明：本文为CSDN博主「程序猿编码」的原创文章，遵循CC 4.0 BY-SA

版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/chen1415886044/article/details/104395351