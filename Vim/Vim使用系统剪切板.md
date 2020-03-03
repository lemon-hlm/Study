## windows:

配置文件中添加 

```
set clipboard=unnamed
```

添加到~/.vimrc, 注意只在gvim有效, 因为gvim有一个+寄存器, 可以与外部程序共享.

## Linux

可以下载源码重新编译，在configure的时候增加 --with-features=huge。