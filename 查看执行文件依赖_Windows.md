用 dumpbin （这个是visual studio 提供的工具，当然，你也可以安装 Dependency Walker 这个独立小巧的工具来看）来查看 DLL 依赖关系：

用 MinGW64 的编译环境，得到 a_mingw.exe

	d:\hello_world>dumpbin /dependents a_mingw.exe

看到

	Microsoft (R) COFF/PE Dumper Version 9.00.21022.08
	Copyright (C) Microsoft Corporation.  All rights reserved.


	Dump of file a_mingw.exe

	File Type: EXECUTABLE IMAGE

	  Image has the following dependencies:

	    KERNEL32.dll
	    msvcrt.dll