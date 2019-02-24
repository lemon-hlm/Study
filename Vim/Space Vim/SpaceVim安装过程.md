先确保安装了Git和Curl这两个工具

在特定终端(比如MobaXterm, Xshell等), 还需要设置终端的字体.

参照: https://github.com/Gabirel/Hack-SpaceVim/blob/master/zh_CN/installation/installation-for-linux.md#%E5%9C%A8linux%E4%B8%8A%E5%AE%89%E8%A3%85spacevim

# 1 安装Python2和3

```
./configure
make
make install > install.log
```

# 2 安装vim

1. 下载最新vim并安装

```
git clone https://github.com/vim/vim.git  vm8  
cd vim8/src
```

2. 编译

下面关于Python的参数可以查看python安装时候的日志查看

```
./configure --with-features=huge --enable-multibyte --enable-pythoninterp=yes --enable-python3interp=yes --enable-luainterp=yes --with-python-config-dir=/usr/lib64/python2.7/config/ --with-python3-config-dir=/usr/local/lib/python3.6/config-3.6m-x86_64-linux-gnu --enable-perlinterp=yes --enable-luainterp=yes --enable-cscope --prefix=/usr --enable-fail-if-missing --with-lua-prefix=/usr/local
```

编译中间任何的问题可以在./src/auto/config.log中有详细日志

其中要是没有lua.h文件, 需要源码重装 https://blog.csdn.net/feinifi/article/details/80078721 , 如果没有安装readline依赖包

lua解决办法：

```
yum install libtermcap-devel ncurses-devel libevent-devel readline-devel -y
make linux test
make linux install
```

其中

```
Python are sane... no: PYTHON DISABLED
```

查看详细日志

```
usr/bin/ld cannot find lpython2.7
```

安装

```
yum install python2-dev*
```

3. 删除之前的vim版本

```
sudo apt list --installed|grep vim                     
sudo apt remove vim-* 
```

4. 安装新版本的vim

```
sudo make install
vim --version
```

5. 清理当前vim配置

```
rm -rf ~/.vim
rm ~/.vim*
```

6. 安装spacevim

```
curl -sLf https://spacevim.org/install.sh | bash
```

spacevim的帮助信息:

```c
curl -sLf https://spacevim.org/cn/install.sh | bash -s -- -h
```

7. 安装Powerline font

SpaceVim 默认启用了Powerline 字体，默认的的字体文件是：[SourceCodePro Nerd Font Mono](https://github.com/ryanoasis/nerd-fonts/releases/download/v2.0.0/SourceCodePro.zip), 推荐个人也使用这个

当然如果终端不识别这个, 可以使用 DejaVu Sans Mono for PowerLine 字体

参考链接: https://spacevim.org/cn/documentation/

SourceCodePro和SauceCodePro一个意思

powerline字体的库: https://github.com/powerline/fonts

nerd-fonts字体的库: https://github.com/ryanoasis/nerd-fonts

```
# clone
git clone https://github.com/powerline/fonts.git
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```

下载上面的压缩包, 然后通过下面命令查看系统中所有的已安装的字体:

```
fc-list
```

从上面也能知道目前系统中的字体的存放路径, 然后将下载的字体拷贝到这个目录下, 然后使用下面命令

```
sudo fc-cache -f -v
```

然后就可以使用新字体了.

8. 如果是在windows平台的终端, 需要在windows平台安装Powerline字体

windows直接复制字体进相应目录即可

9. 在相应终端(Xshell, MobaXterm)配置Powerline字体
 
修改终端字体为xxx for Powerline

10. SpaceVim配置字体
 
必要时候在SpaceVim中配置for powerline的字体

11. 终端的真彩色

安装 SpaceVim 最理想的环境是 neovim + nerdfont + 一个支持真色的终端模拟器

通过色彩检测看是否支持

# 3 配置相关

通过rpmfind查找一个global的rpm安装