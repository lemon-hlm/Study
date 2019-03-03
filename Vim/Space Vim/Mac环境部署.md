[TOC]

# 1 Homebrew的安装和使用

## 1.1 安装Homebrew

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Homebrew安装成功后，会自动创建目录 /usr/local/Cellar 来存放Homebrew安装的程序。 这时你在命令行状态下面就可以使用 brew 命令了

更新Homebrew镜像源(以清华源示例)

参照 https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/ 或 https://lug.ustc.edu.cn/wiki/mirrors/help/brew.git

```
cd "$(brew --repo)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
brew update
```

## 1.2 Homebrew的使用

- 安装软件：brew install 软件名，例：brew install wget
- 搜索软件：brew search 软件名，例：brew search wget
- 卸载软件：brew uninstall 软件名，例：brew uninstall wget
- 更新所有软件：brew update

>通过 update 可以把包信息更新到最新，不过包更新是通过git命令，所以要先通过 brew install git 命令安装git。

- 更新具体软件：brew upgrade 软件名 ，例：brew upgrade git
- 显示已安装软件：brew list
- 查看软件信息：brew info／home 软件名 ，例：brew info git ／ brew home git

>brew home指令是用浏览器打开官方网页查看软件信息

- 查看哪些已经安装的程序需要更新: brew outdated
- 显示包依赖: brew reps

# 2 安装Python3和pip3

## 2.1 安装Python3

```
brew install python3
```

这一步会安装python3和pip3

## 2.2 测试

```
python3 -V

pip3 -V
```

# 3 安装pip2

mac自带了Python2但是不带pip2, 所以再次通过brew安装Python2就自然携带了pip2

## 3.1 安装Python2

```
brew install python@2
```

## 3.2 测试

```
python -V

pip2 -V
```

# 4 安装ipython2和ipython3

通过pip安装

## 4.1 安装ipython3

```
pip3 install ipython
```

## 4.2 测试ipython3是否安装成功

```
ipython3
```

## 4.3 安装ipython2

```
pip install ipython
```

## 4.4 测试ipython2是否安装成功

```
ipython
```

# 5 安装lua

```
brew install lua
```

# 6 安装vim

https://github.com/Gabirel/Hack-SpaceVim/blob/master/zh_CN/installation/installation-for-linux.md#%E5%9C%A8linux%E4%B8%8A%E5%AE%89%E8%A3%85spacevim

https://github.com/Valloric/YouCompleteMe/wiki/Building-Vim-from-source

## 6.1 检查依赖

```
git --version

lua -v

python -V
```

必须确保python默认是3版本

```
/usr/local/bin/python -> ../Cellar/python@2/2.7.15_3/bin/python
```

检查lua.h文件是否存在, 如果不存在需要源码重装lua

参照 https://www.jianshu.com/p/c46a1984e71a

```
brew uninstall lua

# 从 http://www.lua.org/ftp/ 下载

tar zxf lua-5.1.5.tar.gz

cd lua-5.1.5

make macosx test

make install
```

## 6.2 安装

检查现有的vim是否有+lua和python2和python3的支持

![config](./images/5.png)

也可以通过命令: echo has('lua')，echo has('python2')和echo has('python3')

- Lua 返回: 1
- Python 返回: 1
- Python3 返回: 1

所有的都要返回1

通过brew安装vim并做上面检查

```
brew install vim
```

如果没有就卸载掉

### 6.2.1 源码安装

参照 https://github.com/Valloric/YouCompleteMe/wiki/Building-Vim-from-source

```
git clone https://github.com/vim/vim.git

cd src

sudo ./configure --with-features=huge --enable-multibyte --enable-rubyinterp=yes --enable-pythoninterp=yes --with-python-config-dir=/usr/lib/python2.7/config --enable-python3interp=yes --with-python3-config-dir=/usr/local/lib/python3.6/config-3.6m-darwin --enable-perlinterp=yes --enable-luainterp=yes --enable-gui=gtk2 --enable-cscope --with-lua-prefix=/usr/local --prefix=/usr/local

sudo make -j12

sudo make install
```

## 6.3 清理当前vim配置

```
rm -rf ~/.vim
rm ~/.vim*
```

# 7 复制init.toml

复制并检查安装的模块的配置

https://spacevim.org/cn/layers/

global模块可能需要源码重装

# 8 安装字体

参考字体

# 9 创建新卷

创建新的卷, 大小写敏感的, Mac默认大小写不敏感