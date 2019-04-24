
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [图形小抄（Graphical cheat sheet）](#图形小抄graphical-cheat-sheet)
* [图形化教程（Graphical cheat sheet based tutorial）](#图形化教程graphical-cheat-sheet-based-tutorial)
		* [lession 1:](#lession-1)
		* [lession 2:](#lession-2)
		* [lession 3:](#lession-3)
		* [lession 4:](#lession-4)
		* [lession 5:](#lession-5)
		* [lession 6:](#lession-6)
		* [lession 7:](#lession-7)
* [vim 命令图解](#vim-命令图解)
* [Vim特性图解](#vim特性图解)

<!-- /code_chunk_output -->

```
翻译自
http://www.viemu.com/a_vi_vim_graphical_cheat_sheet_tutorial.html
```

## 图形小抄（Graphical cheat sheet）

下图描述了完全的vi/vim输入模式、所有按键的功能以及所有主要特性。你可以把它当作一个压缩版的vi/vim手册。

![Graphical cheat sheet](images/vi-vim-cheat-sheet.gif)

![Graphical cheat sheet](images/vim-cheat-sheet-cn.png)

图中没有关于查找和替换的，应该用下面的。


- 自上而下的查找操作  /word小写的n和N
- 自下而上的查找操作  ?word小写的n和N
- 普通替换操作    :s/old/new
- 当前行进行匹配和替换、命令替换当前行中第一个匹配的字符行内全部替换操作s/old/new/g
- 当前行替换所有匹配的字符串在行区域内进行替换操作:#,#s/old/new/g
- 在整个文件内的替换操作:%s/old/new/g
- 在整个文档中进行替换操作的命令使用替换的确认功能:s/old/new/c:s/old/new/gc:#,#s/old/new/gc:%s/old/new/gc


## 图形化教程（Graphical cheat sheet based tutorial）

这个教程由7张图片组成，覆盖了vi/vim的主要命令。该教程是循序渐进的，首先是最简单以及最常用的部分，然后是一些高级特性。

#### lession 1:

![vi-vim-tutorial-1](images/vi-vim-tutorial-1.gif)

#### lession 2:

![vi-vim-tutorial-2](images/vi-vim-tutorial-2.gif)

#### lession 3:

![vi-vim-tutorial-3](images/vi-vim-tutorial-3.gif)

#### lession 4:

![vi-vim-tutorial-4](images/vi-vim-tutorial-4.gif)

#### lession 5:

![vi-vim-tutorial-5](images/vi-vim-tutorial-5.gif)

#### lession 6:

![vi-vim-tutorial-6](images/vi-vim-tutorial-6.gif)

#### lession 7:

![vi-vim-tutorial-7](images/vi-vim-tutorial-7.gif)

## vim 命令图解

![vim命令图解](images/Vim_Commands.png)

## Vim特性图解

![vim feature](images/vim-study.png)