### 1. 错误信息

```
fatal: Out of memory, malloc failed (tried to allocate 364974473 bytes)
```

### 2. 解决办法

#### 2.1 网上解决办法

第一步，调内存

第二步，调配置

```
git config --global pack.threads 1
git config --global pack.deltaCacheSize = 128m
git config --global pack.windowMemory 50m
```

但是再一次失败了

#### 2.2 最终解决办法

原理：调节swap

创建一个2G文件：

```
qemu-img create /root/myswap 5G
dd if=/dev/zero of=/root/myswap bs=1M count=2048
```

修改权限

```
chmod 600 /root/myswap
```

使成为一个交换文件

```
mkswap /root/myswap
```

启用

```
swapon /root/myswap
```

即使重启也可用

```
vim /etc/fstab

/root/myswap  swap  swap  defaults 0  0
```

确认可用

```
#swapon -s
Filename                        Type            Size    Used    Priority
/dev/sda2                       partition       4192956 0       -1
/root/myswapfile                file            1048568 0       -2

#free -k
total       used       free     shared    buffers     cached
Mem:       3082356    3022364      59992          0      52056    2646472
-/+ buffers/cache:     323836    2758520
Swap:      5241524          0    5241524
```
