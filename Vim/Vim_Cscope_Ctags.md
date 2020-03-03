转载自 http://easwy.com/blog/archives/vim-cscope-ctags/


使用vim + cscope/ctags，就可以实现SourceInsight的功能，以后可以不再用盗版SouceInsight读代码了。

按照vim里cscope的参考手册(在vim中执行”:help cscope”命令)，把cscope功能加到.vimrc里后(需要你的vim在编译时选择了”–enable-cscope”选项，否则你需要重新编译vim)，配置就算完成了。然后用下面的命令生成代码的符号索引文件：

```
cscope -Rbkq
```

这个命令会生成三个文件：cscope.out, cscope.in.out, cscope.po.out。

其中cscope.out是基本的符号索引，后两个文件是使用”-q”选项生成的，可以加快cscope的索引速度。

上面所用到的命令参数，含义如下：

```
-R: 在生成索引文件时，搜索子目录树中的代码
-b: 只生成索引文件，不进入cscope的界面
-k: 在生成索引文件时，不搜索/usr/include目录
-q: 生成cscope.in.out和cscope.po.out文件，加快cscope的索引速度
```

接下来，就可以在vim里读代码了。

不过在使用过程中，发现无法找到C\+\+的类、函数定义、调用关系。仔细阅读了cscope的手册后发现，原来cscope在产生索引文件时，只搜索类型为C, lex和yacc的文件(后缀名为.c, .h, .l, .y)，C++的文件根本没有生成索引。不过按照手册上的说明，cscope支持c\+\+和Java语言的文件。

于是按照cscope手册上提供的方法，先产生一个文件列表，然后让cscope为这个列表中的每个文件都生成索引。

为了方便使用，编写了下面的脚本来更新cscope和ctags的索引文件：

```
#!/bin/sh

find . -name "*.h" -o -name "*.c" -o -name "*.cc" > cscope.files
cscope -bkq -i cscope.files
ctags -R
```

- 首先使用find命令，查找当前目录及子目录中所有后缀名为”.h”, “.c”和”.cc”的文件，并把查找结果重定向到文件cscope.files中。

- 然后cscope根据cscope.files中的所有文件，生成符号索引文件。

- 最后一条命令使用ctags命令，生成一个tags文件，在vim中执行”:help tags”命令查询它的用法。它可以和cscope一起使用。

目前只能在unix系列操作系统下使用cscope，虽然也有windows版本的cscope，不过还有很多bug。

cscope的主页在：http://cscope.sourceforge.net/

在vim的网站上，有很多和cscope相关的插件，可以去找一下你有没有所感兴趣的。