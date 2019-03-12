OpenStack 里有三个地方可以和 Ceph 块设备结合：

- Images： OpenStack 的 Glance 管理着 VM 的 image 。Image 相对恒定， OpenStack 把它们当作二进制文件、并以此格式下载。
- Volumes： Volume 是块设备， OpenStack 用它们引导虚拟机、或挂载到运行中的虚拟机上。 OpenStack 用 Cinder 服务管理 Volumes 。
- Guest Disks: Guest disks 是装有客户操作系统的磁盘。默认情况下，启动一台虚拟机时，它的系统盘表现为 hypervisor 文件系统的一个文件（通常位于 /var/lib/nova/instances/<uuid>/）。在 Openstack Havana 版本前，在 Ceph 中启动虚拟机的唯一方式是使用 Cinder 的 boot-from-volume 功能. 不过，现在能够在 Ceph 中直接启动虚拟机而不用依赖于 Cinder，这一点是十分有益的，因为可以通过热迁移更方便地进行维护操作。另外，如果你的 hypervisor 挂掉了，也可以很方便地触发 nova evacuate ，并且几乎可以无缝迁移虚拟机到其他地方。

# 1 配置ceph

## 1.1 创建存储池

### 1.1.1 创建镜像pool

用于保存Glance镜像

```
$ ceph osd pool create images 32 32
pool 'images' created
```

### 1.1.2 创建卷pool

用于保存cinder的卷

```
$ ceph osd pool create volumes 32 32
pool 'volumes' created
```

用于保存cinder的卷备份

```
$ ceph osd pool create backups 32 32
pool 'backups' created
```

### 1.1.3 创建虚拟机pool

用于保存虚拟机系统卷

```
$ ceph osd pool create vms 32 32
pool 'vms' created
```

### 1.1.4 查看pool

```
$ sudo ceph osd lspools
3 volumes,4 vms,5 images,6 rbd,7 backups,
```

## 1.2 创建用户

### 1.2.1 创建Glance用户

创建glance用户, 并给images存储池