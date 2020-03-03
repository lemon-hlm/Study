
```
1. 跳转相关
2. 文件相关
3. 编辑相关
    3.1 缩进
    3.2. 空格和tab
4. 显示相关
    4. 高亮
5. 文件目录查找字符串
```

## 1. 跳转相关

- 跳动(jump) 来说，\<c-o\> 相当于是后退，\<c-i\> 是前进。

- \`\`可以回跳到上一个位置，多次按\`\`会在两个位置间跳转.

- % 跳转到相配对的括号（光标必须在符号上面）

- [[ 跳转至上一个函数（要求代码块中\'{\'必须单独占一行）

- ]] 跳转至下一个函数（要求代码块中\'{\'必须单独占一行）

- mx 设置书签，x只能是a-z的26个字母，\`x 跳转到书签处（"`"是1左边的键）

## 2. 文件相关

- :e anotherFile                      新增一个编辑文件，
- :e#                                 返回之前的文件
- :e ftp://192.168.10.76/abc.txt      打开远程文件
- :e \\\\qadrive\test\1.txt           打开share file

- vim打开多个文件

```
vim a b c

先进入a文件
敲入:n，进入b文件
再敲入:n，进入c文件

如果嫌文件名太长的话,
可以
:b1  回到第1个文件
:b2  回到第2个文件
:b3  回到第3个文件

如果文件已经被修改,
需要加上!强制执行, 如:
:b!2

:n   切换到下一个文件
:N   切换到上一个文件
:n filename2    切换到文件filename2

:ls             列出vim打开的所有文件的信息，包括文件名，buffer id等
:bn             切换到当前buffer的下一个buffer
:bp             切换当前buffer的前一个buffer
:bd             关闭当前buffer，对应文件也随之关闭
:bd2            关闭buffer id为2的buffer，对应文件也随之关闭
:args           查看当前打开的文件列表，当前正在编辑的文件会用[]括起来。
```

## 3. 编辑相关

### 3.1 缩进

#### normal 模式下 

- \>\> 当前行增加缩进

- 3\> 当前行和下三行（共四行）缩进一次

- :10,100> 第10行至第100行缩进一次

- :10,100>> 第10行至第100行缩进两次

---

- \<\< 当前行减少缩进

- 3\< 当前行和下三行（共四行）缩进一次

- :20,80< 第20行至第80行反缩进一次

- :20,80<< 第20行至第80行反缩进两次

#### Visual 模式下 

选择好需要缩进的行后，按一次大于号’>’缩进一次，按’6>’缩进六次，按’<’回缩

#### INSERT 模式下

```
CTRL+SHIFT+T      当前行增加缩进 

CTRL+SHIFT+D      当前行减少缩进
```

#### 自动缩进

通常根据语言特征使用自动缩进排版：

在命令状态下对当前行用== （连按=两次）, 或对多行用n==（n是自然数）表示自动缩进从当前行起的下面n行。

你可以试试把代码缩进任意打乱再用n==排版，相当于一般IDE里的code format。


### 3.2. 空格和tab

#### 3.2..1 默认设置tab为4个空格

为了vim更好的支持python写代码,修改tab默认4个空格有两种设置方法

方法一：

```
set tabstop=4
set shiftwidth=4
```

方法二：

```
set tabstop=4
set expandtab
set autoindent
```

其中 tabstop 表示一个 tab 显示出来是多少个空格的长度，默认 8。

softtabstop 表示在编辑模式的时候按退格键的时候退回缩进的长度，当使用 expandtab 时特别有用。

shiftwidth 表示每一级缩进的长度，一般设置成跟 softtabstop 一样。

当设置成 expandtab 时，缩进用空格来表示，noexpandtab 则是用制表符表示一个缩进。

推荐使用第二种方法，按tab键时产生的是4个空格，这种方式具有最好的兼容性。

#### 3.2.2 修改已经保存的文件

#### tab替换为空格

```
:set ts=4
:set expandtab
:%retab!
```

#### 空格替换为TAB

```
:set ts=4
:set noexpandtab
:%retab!
```

加!是用于处理非空白字符之后的TAB，即所有的TAB，若不加!，则只处理行首的TAB。

## 4. 显示相关

### 4.1 高亮显示

#### 方法一：单高亮

用vim时，想高亮显示一个单词并查找的方发，将光标移动到所找单词。

- shift + "*"  向下查找并高亮显示
- shift + "#"  向上查找并高亮显示
- "g" + "d"    高亮显示光标所属单词，"n" 查找！

如果没有高亮使用命令“:set hls”。

取消搜索高亮使用命令“:nohl”

#### 方法二：多高亮

简单的高亮可以使用 :match 和 :hi[light]命令

运行:hi，在显示的列表中选择一个或多个高亮名称。

```
:match ErrorMsg /evil/
:2match WildMenu /VIM/
:3match Search /Main/
```

取消使用命令

```
:mat[ch] 
:2mat[ch] none
```

就能高亮多个，最多有三个，见:help match

#### 方法三：插件mark.vim

## 5. 文件目录查找字符串

```
vimgrep /匹配模式/[g][j]    要搜索的文件/范围 
```

g：没有参数g的话,则行只查找一次关键字。反之会查找所有的关键字  
j：没有参数j的话,查找后,VIM会跳转至第一个关键字所在的文件。反之,只更新结果列表(quickfix)

```
vimgrep /pattern/ %                 在当前打开文件中查找
vimgrep /pattern/ *                 在当前目录下查找所有
vimgrep /pattern/ **                在当前目录及子目录下查找所有
vimgrep /pattern/ *.c               查找当前目录下所有.c文件
vimgrep /pattern/ **/*              只查找子目录
:vimgrep /pattern/ ./includes/*.*   在当前目录中的"includes"目录里的所有文件中查找
```

```
cn                                 查找下一个
cp                                 查找上一个
copen                              打开quickfix
cw                                 打开quickfix
cclose                             关闭qucikfix
help vimgrep                       查看vimgrep帮助
```
