[TOC]

https://blog.51cto.com/11811268/1896308

有时候,一台服务器需要设置多个ip,但又不想添加多块网卡,那就需要设置虚拟网卡.这里介绍几种方式在Linux服务器上添加虚拟网卡. 

# 1 

# 2

# 3 创建tap

安装rpm包

```
yum install tunctl-1.5-12.el7.nux.x86_64
```

添加虚拟网桥

```
brctl addbr br0
```

激活虚拟网桥并查看网桥信息

```
ip link set br0 up
brctl show 
```

添加虚拟网卡tap

```
# tunctl -b
tap0 -------> 执行上面使命就会生成一个tap,后缀从0，1，2依次递增
```

激活创建的tap

```
ip link set tap0 up
```

将tap0虚拟网卡添加到指定网桥上。

```
brctl addif br0 tap0
```

给网桥配制ip地址

```
ifconfig br0 169.254.251.4 up
```

给br0网桥添加网卡eth6

```
brctl addif br0 eth6 
```

给tap0配置网络

```
ifconfig tap0 192.168.92.126 netmask 255.255.255.240 up
```

将virbr1网桥上绑定的网卡eth5解除

```
brctl delif virb1 eth5  
```
