http://docs.ceph.org.cn/rbd/rbd-openstack/

通过 libvirt 你可以把 Ceph 块设备用于 OpenStack ，它配置了 QEMU 到 librbd 的接口。 Ceph 把块设备映像条带化为对象并分布到集群中，这意味着大容量的 Ceph 块设备映像其性能会比独立服务器更好。

要把 Ceph 块设备用于 OpenStack ，必须先安装 QEMU 、 libvirt 和 OpenStack 。我们建议用一台独立的物理主机安装 OpenStack ，此主机最少需 8GB 内存和一个 4 核 CPU 。下面的图表描述了 OpenStack/Ceph 技术栈。

# 1 创建存储池

```
ceph osd pool create volume 128
ceph osd pool create images 128
ceph osd pool create backups 128
ceph osd pool create vm 128

[lihaiwei@BJ-IDC1-10-10-31-26 ~]$ sudo ceph osd lspools
3 volume,4 vm,5 images,6 rbd,7 backups,
```

# 2 配置Openstack的ceph客户端

## 2.1 

运行着 glance\-api 、 cinder-volume 、 nova-compute 或 cinder-backup 的主机被当作 Ceph 客户端，它们都需要 ceph.conf 文件。

```
ssh {your-openstack-server} sudo tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf
```
