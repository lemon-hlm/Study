# Linux命令基础

 - commands not found的错误原因
     - 输错命令
     - 软件没有安装
     - 这个命令所在的目录，当前用户没有将它加入命令搜寻路径中，echo $PATH可以查看。
 - 系统的所有环境变量env可以查看
 - Linux在线求助：man page | info page  
 - 中文命令网站：http://man.linuxde.net/

# 1 man page

 - man page适用于所有Unix like系统。
 - man page的数字说明：
```
1：用户可以在shell环境中操作的命令或可执行文件
2：系统内核可调用的函数与工具
3：一些常用的函数（function）与函数库（library），大部分为C的函数库
4：设备文件的说明，通常在/dev/下面的文件
5：配置文件或某些文件的格式
6：游戏（games）
7：惯例与协议等，例如Linux文件系统、网络协议、ASCII code等说明
8：系统管理员可以使用的管理命令
9：与kernel相关的文件
    
注：man null会显示“NULL(4)”，可以看出null是一个设备文件
```
 - man page大致分为如下部分，见图  
 ![man page](images/man1.PNG "man page 说明")  
 - man page的数据也是存放在目录下面，不同distribution通常有点区别，通常在/usr/share/man目录，不过可以通过修改man page的查询路径。/etc/man.config(有些是man.conf或manpath.conf)，更多信息通过“man man”查询。
 - 查询出来的结果取决于配置文件（/etc/man.conf）的顺序，**先查询到的说明文件就先显示出来**。
 - man -f <命令或数据>：查询更多的说明文件。该命令的简略写法： whatis <命令或数据> 
 - man -k <命令或数据>：根据关键字查询说明文件。该命令的简略写法：apropos <命令或数据> 

> 注：这两个特殊命令（whatis、apropos）要可以使用，必须要创建whatis数据库才行，使用root账户，执行 makewhatis

总结：

> 不用背命令，记住常用的命令即可。根据man page查找命令并查看用法。

# 2 info page

 - Linux除了man，还提供的一套额外的在线求助方法，Linux独有的。
 - 目标数据的说明文件只有以info格式写成才能使用info的特殊功能（例如超链接）。而支持info命令的文件默认在/usr/share/info/目录下。

# 3 其他文件

 - 一般，命令或软件开发者都会将自己命令或软件的说明制作成“在线帮助文档”，会放置在/usr/share/doc这个目录下，包括bash信息等。