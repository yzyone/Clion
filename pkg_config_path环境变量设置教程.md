
# pkg_config_path 环境变量设置 教程 #

由于最近在瞎搞Kali Linux NetHunter，需要用上“pkg_config_path 环境变量设置”，担心有很多人跟我一样也在搞NetHunter而踩坑走不出来，我特意发表出来。

 

**一、查看 pkg_config_path 环境变量 命令**

    root@kali:~# echo $PKG_CONFIG_PATH

查看 pkg_config_path 环境变量

从上面可以清楚的看到我的“ pkg_config_path 环境变量”是空的。

 

**二、查看自己的 pkgconfig 路径在哪里？**

    root@kali:~# find / -name pkgconfig

查看自己的 pkgconfig 路径

可以清楚的看到有三个pkgconfig路径：/usr/share/pkgconfig、/usr/lib/pkgconfig、/usr/lib/x86_64-linux-gnu/pkgconfig，自己看着去选吧！我建议大家选择前两个任意一个。

 

**三、设置 pkg_config_path 环境变量 方法**

有两种方法都可以设置 pkg_config_path 环境变量 。

 

1、如果你只是想加上某库的pkg，只需要用如下命令：

    root@kali:~# export PKG_CONFIG_PATH=/usr/lib/pkgconfig/ 
 

2、如果你想覆盖掉原来的pkg，可选择用此方法。因为PKG_CONFIG_LIBDIR的优先级比 PKG_CONFIG_PATH 高，所以会覆盖PKG_CONFIG_PATH的设置。

    root@kali:~# export PKG_CONFIG_LIBDIR=/usr/lib/pkgconfig/
 

也可以使用如下命令，注意一定要放在PKG_CONFIG_PATH的前面，这样才能首先读取。

    root@kali:~# export PKG_CONFIG_PATH=/usr/lib/pkgconfig/:$PKG_CONFIG_PATH
 

pkg_config_path 环境变量设置”成功效果图

总结：最后只需要给大家看一下我的“pkg_config_path 环境变量设置”成功效果图，如上图。