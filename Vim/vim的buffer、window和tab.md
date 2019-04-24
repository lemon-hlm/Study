
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [标签页(Tab)](#标签页tab)
* [Buffer](#buffer)

<!-- /code_chunk_output -->

Vim 中的 window 和 tab 非常具有迷惑性，跟我们平时所说的 “窗口” 和 “标签页” ，是完全不同的两个概念，请看 vimdoc 给出的定义:

```
A buffer is the in-memory text of a file.
A window is a viewport on a buffer.
A tab page is a collection of windows.
```

简单来说就是:

- buffer 可以看做是内存中的文本文件，在没写到磁盘上时，所有的修改都发生在内存中;
- window 用来显示 buffer，同一个 buffer 可以被多个 window 显示(一个 window 只能显示一个 buffer);
- tab page 包含了一系列的 window，其实叫 layout 更合适，看 [这里](http://stackoverflow.com/questions/102384/using-vims-tabs-like-buffers/103590#103590)

vim中的buffer就相当于一个文件，windows相当于一个窗口的frame（一个显示区，viewport），tab相当于一个窗口。也就是说，一个显示区（frame）显示一个文件（buffer），一个窗口（tab）可以有多个显示区（frame）。

来看 Vim 官网上的一幅图:

![config](images/windows_buffer_tab.png)

## 标签页(Tab)

Vim的标签（Tab）页，类似浏览器的标签页，一个标签页打开一个Vim的窗口，一个Vim的窗口可以支持N个分屏。
在Vim中新建一个标签的命令是：

```
:tabnew
```

如果要在新建标签页的同时打开一个文件，则可以在命令后面直接附带文件路径：

```
:tabnew filename
```

Vim中的每个标签页有一个唯一的数字序号，第一个标签页的序号是0，从左向右依次加一。关于标签页有一系列操作命令，简介如下：

```
:tabs                   查看所有打开的tab

:tN[ext]                跳转到上一个匹配的标签
:tabN[ext]              跳到上一个标签页
:tabc[lose]             关闭当前标签页
:tabdo                  为每个标签页执行命令
:tabe[dit]              在新标签页里编辑文件
:tabf[ind]              寻找 'path' 里的文件，在新标签页里编辑之
:tabfir[st]             转到第一个标签页
:tabl[ast]              转到最后一个标签页
:tabm[ove]  N           把标签页移到序号为N位置
:tabnew [filename]      在新标签页里编辑文件
:tabn[ext]              转到下一个标签页
:tabo[nly]              关闭所有除了当前标签页以外的所有标签页
:tabp[revious]          转到前一个标签页
:tabr[ewind]            转到第一个标签页
```

## Buffer

Buffer 是内存中的一块缓冲区域，用于临时存放Vim打开过的文件。用Vim打开文件后，文件就自动被加入到Buffer队列中，而且Buffer中永远是最新的版本，修改文件后还未保存时，改动就存在于Buffer中。打开过的文件会一直存在Buffer中，除非手动的删除（bw命令，不过很多时候没这个必要）。

```
:ls                     查看当前Buffer中的文件列表
:buffers                查看当前Buffer中的文件列表
:files                  查看当前Buffer中的文件列表

:bn(bnext)              切换到当前buffer的下一个buffer
:bp(bprevious)          切换当前buffer的前一个buffer
:bfirst                 切换到第一个buffer
:blast                  切换到最后一个buffer

:badd file              将文件file添加到buffer
:bd(bdelete)            关闭当前buffer，对应文件也随之关闭
:bd 2                   关闭buffer id为2的buffer(先要查看)，对应文件也随之关闭

:args                   查看当前打开的文件列表，当前正在编辑的文件会用[]括起来。
```

:ls等第一列是文件编号，第二列是缓冲文件的状态，第三列是文件的名称，第四列是上一次编辑的位置，即在不同文件之间切换的时候Vim会自动跳转到上一次光标所在的位置。 缓冲文件的状态有如下几种，仅供参考：

```
- （非活动的缓冲区）
a （当前被激活缓冲区）
h （隐藏的缓冲区）
% （当前的缓冲区）
# （交换缓冲区）
= （只读缓冲区）
+ （已经更改的缓冲区）
```

