# Source Insight 4.0使用教程 #

创建工程,添加源程序

1.打开Source Insight4.0，工具栏选择Project -> New Project,将弹出下列框图:

![](./sourceinsight4/20190125100236518.png)

2.点击OK后,会弹出下列框图,填入工程的名称，以及工程文件生成的目录

![](./sourceinsight4/20190125100542813.jpg)

3.点击OK后,如下图:

![](./sourceinsight4/20190125100733858.jpg)

其中:                          

Add ：基本的文件添加操作

Add All ：添加整个工程所有的源文件（然后再选择是否递归添加子目录中的源文件,见上图对话框）

Add Tree ：添加指定的文件夹以及其子目录下的源代码文件

Remove Tree ：和Add Tree的功能相反

File Name可以不用写,完成点击close.

4.如没有出现下图,红色区域的栏目,按Ctrl + O即可

![](./sourceinsight4/20190125120542606.jpg)

5.将添加的代码进行同步，生成阅读代码的索引和辅助文件，方便对源代码进行阅读;点击Project--> synchronization file,会弹出下图框图:

![](./sourceinsight4/20190128121631854.png)

6.选择语言和后缀名的文件,点击Options-->File type Options,弹出下图框图：

![](./sourceinsight4/20190128122013503.jpg)

## 使用技巧 ##

1、选中字符，相同字符高亮

点击Options-->File type Options,弹出下图框图，标注项选上：

![](./sourceinsight4/dd7fdd8b76f3448c8ae8e4de86650e73.png)

效果如下：

![](./sourceinsight4/50b5cdab8cb74f6d98c65e42d315638d.png)

2、显示调用关系

方法1：

当选择某个函数时，显示函数调用流程，选择某个函数 -> 右键 -> Show in Relation Window，如下图所示。

![](./sourceinsight4/c86c2ab086f14138bb1cca80104ba812.png)

方法2：

![](./sourceinsight4/70733e9dbc2344d0ba31883628b4d7e8.png)

3、context window

context window 在看程序时很有用，当用户指向某个函数或变量时，在context window中都会有该变量或函数的定义。

![](./sourceinsight4/b6bf0e9dd96d46d1a15050cb79aaa5b6.png)

4、全局收索

![](./sourceinsight4/5fe36ba27f19409ab0f927ca1493bab0.png)

![](./sourceinsight4/fc81a18f83484b5b9ea5f52b908ed77d.png)

![](./sourceinsight4/43d4c20d762d455bb0645ce47183cc0e.png)

5、解决中文乱码

（1）单个文件

步骤：File -> Reload Enconding....

![](./sourceinsight4/b6fd036f138b47209d9dc69b80fd83e2.png)

（2）所有文件

步骤：Options > Preferences >File标签。

![](./sourceinsight4/2f8da181437b48c6b4dd74e0d792975f.png)

 
————————————————

版权声明：本文为CSDN博主「深深生生」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/qq_41960196/article/details/86636713