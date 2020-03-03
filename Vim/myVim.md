
参考：

https://github.com/Gerry-Lee/vim

VIM配置参照项目：https://github.com/SpaceVim/SpaceVim

### 1. 安装软件

```
apt-get install ctags cscope git wmctrl fonts-powerline
```

然后根据cscope的路径修改vimrc配置文件

### 2. 备份原有.vim和.vimrc

```
mv ~/.vim ~/.vim.orig
mv ~/.vimrc ~/.vimrc.orig
```

### 3. 添加配置文件

```
git clone https://github.com/Gerry-Lee/VimConfig.git ~/.vim
ln -s ~/.vim/vimrc ~/.vimrc
```

### 4. 安装插件

```
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/vundle
```

在vim执行“:BundleInstall”进行插件安装。

```
vim
:BundleInstall
```

参考当前目录下vimrc


### 6. 对库进行ctags\+cscope

```
cd /usr/include
ctags -R ./
find . -name "*.h" -o -name "*.c" -o -name "*.cc" -o -name "*.cpp" > cscope.files
cscope -bq -i cscope.files
```

### 7. 快捷键

- <F5> :TagbarToggle
- <F6> :NERDTreeToggle
- <F3> :GundoToggle
- <F4> :IndentGuidesToggle
- <C-F11> :!cscope -bRq
- <C-F12> :!ctags -R --c-kinds=+l+x+p --fields=+lS -I __THROW,__nonnull --extra=+ .

### 8.