
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 两个环境(远程调试)](#1-两个环境远程调试)
	* [1.1 初始化环境配置](#11-初始化环境配置)
	* [1.2 连接调试测试](#12-连接调试测试)
	* [1.3 配置eclipse](#13-配置eclipse)
		* [1.3.1 创建项目](#131-创建项目)
		* [1.3.2 配置debug选项](#132-配置debug选项)
			* [1.3.2.1 远程手动运行方式](#1321-远程手动运行方式)
			* [1.3.2.2 远程自动运行调试](#1322-远程自动运行调试)

<!-- /code_chunk_output -->

使用QEMU \+ GDB \+ Eclipse调试Linux内核

有两种情况, 一种是内核虚拟机和eclipse在同一个环境, 另一种分属两个环境

# 1 两个环境(远程调试)

## 1.1 初始化环境配置

本地安装Eclipse for Cpp, 这个版本自带了CDT，不用另行安装CDT插件。

本地checkout一份代码出来, 该代码和编译的调试用的kernel commit信息一致

本地安装gdb

远端的bzImage作为虚拟机启动镜像

本地vmlinux是带有调试信息的镜像, 与上面的对应

## 1.2 连接调试测试

启动远端gdbserver

```
qemu-system-x86_64 -smp 2 -m 1024 -kernel arch/x86/boot/bzImage -nographic -append "rdinit=/linuxrc loglevel=8 console=ttyS0" -S -s
```

或

```
qemu-system-x86_64 -smp 2 -m 1024 -kernel arch/x86/boot/bzImage -append "rdinit=/linuxrc loglevel=8" -S -s -daemonize
```

在当前节点

```
# telnet 10.1.194.104 1234

# gdb
(gdb) target remote 10.1.194.104:1234
(gdb) b start_kernel
(gdb) c
```

## 1.3 配置eclipse

preferences-->C/C++-->indexer 去掉勾选 Enable indexer。

preferences-->General-->Workspace 去掉勾选 Build Automatically。

### 1.3.1 创建项目

New → "Makefile Project with Existing Code", 这里代码目录选择上面说的与调试内核代码一致的目录, Toolchain选为None.

![](./images/2019-05-31-14-06-14.png)

右上角"Debug", 选择"C/C\+\+"

![](./images/2019-05-31-12-51-02.png)

### 1.3.2 配置debug选项

配置debug选项, "Run" → "Debug Configurations", 选择"C\/C\+\+ Remote Application(属于CDT插件)"

Eclipse的C/C++插件CDT已经很好的支持gdb在远程调试了。调试一个应用程序时，CDT有三种运行方式：

1. Automatic Remote Launcher ：**远程自动运行**，这是最方便好用的一种方式

2. Manual Remote Launcher : **远程手动运行**。**用户**自己在**目标板**上运行**gdbserver**，然后在**开发主机**上指定**远程连接的方式**（如IP地址和端口），连接到gdbserver

3. Remote Attach Launcher ：**远程依附运行**。类似于上一种，但它不是重新运行程序开启一个debug会话，而是**直接Attach到一个已经运行的程序**，然后调试

在Debug Configurations 对话框中，创建一个远程调试配置，这个配置在创建时会根据项目情况提供一个默认的配置，**默认**将使用第一种Automatic Remote Launcher方式，这在Main标签中下方“GDB (DSF) Automatic Remote Debugging Launcher”可以看出，点击右边的“**Select other**…”可以切换其它方式。

#### 1.3.2.1 远程手动运行方式

我们希望Eclipse**每次生成一个项目**之后，**自动**把生成出来的**二进制程序拷贝到目标板上**，这可以通过**NFS挂载共享目录来实现**，我们只需要配置项目属性（依次展开：C/C\+\+Build > Settings > Build Steps > Post\-build steps，在Command中输入“cp ProgramBin /mnt/share”）即可。

接下来配置CDT的Debug选项，步骤如下：

1. 选中项目→菜单栏 ”Run“→Debug Configurations…

2. 双击 C/C++ Remote Application 新建一个配置，Eclipse会根据当前选择的项目初始化大部分配置，Project选择我们的linux\-debug项目, application填上面的带有调试信息的vmlinux, 选中 Disable auto build

![](./images/2019-05-31-22-28-18.png)

3. 在右下方点击“**Select other**”，选择“**GDB(DSF) Manual Remote Debugging Launcher**”，确认

![](./images/2019-05-31-22-03-57.png)

4. 选择进入Debugger配置页，在**Main标签**中，GDB debugger 填写**gdb**，如果**未加入PATH**环境变量则应该**填入绝对路径**, Stop on startup at填"start\_kernel"

![](./images/2019-05-31-22-29-43.png)

5. 在Debugger配置页的Shared Libraries标签中，可以添加库路径，比如调试过程中要步入**外部函数**，就必须在这里给出**带调试信息的库文件路径**，否则会找不到该函数的定义

6. 在Debugger配置页的Connection标签中，Type选“TCP”，并填写目标板上gdbserver监听的IP和端口号

![](./images/2019-05-31-22-33-21.png)

所有配置完成后，点“**Apply**”保存配置，并关掉配置窗口, 不要直接"debug"

接下来在目标板上运行 gdbserver, 即上面的执行虚拟机命令

点击CDT工具栏中的绿色小虫子右边的箭头，选择Linux\-remote\-debug，如果有什么错误提示不用理会，继续proceed。 

连接成功后，控制台会有输出，也可以在控制台输入gdb调试指令，也可以在CDT上直接加断点。

这时你会发现在 start\_kernel处停了下来。F5 F6这些快捷键都可以用。

#### 1.3.2.2 远程自动运行调试

在开发主机上查看目标板（远端主机）的系统状况，如查看远程的文件，查看进程列表等，甚至打开终端运行远程机器上的程序。我们还希望这样一个工具软件能集成在Eclipse上，并且提供了API接口让Eclipse的插件能调用远程机器上的程序。

**TM/RSE** 就是这样一个工具，全称是 Target Management / Remote Systems Explorer，是Eclipse上的一个插件。Eclipse的远程管理插件有很多，但这个最常用，所以我们安装了 Eclipse 的C++版，就自带了这个插件。ARM的软件开发环境DS-5  也使用了这个插件，其实整个DS-5的基础都是Eclipse社区的开源软件搭建起来的。关于TM/RSE的更多介绍，可参考其官方资料：TM and RSE FAQ

我们可以在安装了TM/RSE的Eclipse上，点击右上角的“Open Perspective”按钮，打开“Remote System Explorer”，在 Remote System Explorer 上，你可以很方便的管理多个远程连接，查看远程文件和进程，运行远程程序等。

RSE 支持很多网络连接类型，常用的有：



file-->new-->c project-->project name填写你自己合意的，比如linux-kernel-study / 去掉use default location，location输入框中选择你的linux-4.4.19路径。 / project type 选择 Makefile project-Empty project / Toolschains选择Linux GCC-->next-->Advanced Setting-->C/C++ Build-->去掉勾选use default build command， build command输入框内写上 make CONFIG_DEBUG_SECTION_MISMATCH=y -j2，build directory选择你的linux-4.4.19路径。 / Behavior页签 Build(Increament build) 输入框中输入一个空格即可。然后完成工程创建即可。







选择"C/C\+\+ Attach to Application"双击, 新建一个配置, 起个名字

Main中

Project选择刚才创建的project, C/C\+\+ Application选择上面所说的vmlinux. 

![](./images/2019-05-31-14-37-13.png)

Debugger中

Debugger选择"gdbserver"

Main中, 如下

![](./images/2019-05-31-14-40-39.png)

connection中, 如下, 对端IP填写, 端口为1234

![](./images/2019-05-31-14-41-04.png)

点击右下的"debug"


