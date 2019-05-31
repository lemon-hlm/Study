
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 初始化环境配置](#1-初始化环境配置)
* [2 启动虚拟机](#2-启动虚拟机)
* [3 配置eclipse](#3-配置eclipse)

<!-- /code_chunk_output -->

使用QEMU \+ GDB \+ Eclipse调试Linux内核

# 1 初始化环境配置

本地安装Eclipse for Cpp

本地checkout一份代码出来, 该代码和编译的kernel commit信息一致

# 2 启动虚拟机

```
qemu-system-x86_64 -smp 2 -m 1024 -kernel arch/x86/boot/bzImage -nographic -append "rdinit=/linuxrc loglevel=8 console=ttyS0" -S -s
```

# 3 配置eclipse

右上角"Debug", 选择"C/C\+\+"

![](./images/2019-05-31-12-51-02.png)

new → "Makefile Project with Existing Code"

