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
$ ceph osd pool create volume 32 32
pool 'volume' created
```

用于保存cinder的卷备份

```
$ ceph osd pool create backups 32 32
pool 'backups' created
```

### 1.1.3 创建虚拟机pool

用于保存虚拟机系统卷

```
$ ceph osd pool create vm 32 32
pool 'vm' created
```

### 1.1.4 查看pool

```
$ sudo ceph osd lspools
3 volume,4 vm,5 images,6 rbd,7 backups,
```

## 1.2 创建用户

### 1.2.1 创建Glance用户

创建glance用户, 并给images存储池访问权限

```
$ ceph auth get-or-create client.glance
[client.glance]
	key = AQC+3IVc37ZBEhAAS/F/FgSoGn5xYsclBs8bQg==

$ sudo ceph auth caps client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
updated caps for client.glance
```

查看并保存glance用户的keyring文件

```
$ ceph auth get client.glance
[client.glance]
	key = AQC+3IVc37ZBEhAAS/F/FgSoGn5xYsclBs8bQg==
	caps mon = "allow r"
	caps osd = "allow rwx pool=images"

$ ceph auth get client.glance -o /var/openstack/ceph/ceph.client.glance.keyring
exported keyring for client.glance
```

### 1.2.2 创建Cinder用户

创建cinder-volume用户, 并给volume存储池权限

```
$ ceph auth get-or-create client.cinder-volume
[client.cinder-volume]
	key = AQAf3oVcN2nMORAAl740sqdkcwE/8a/niSTIeg==

$ ceph auth caps client.cinder-volume mon 'allow r' osd 'allow rwx pool=volume'
updated caps for client.cinder-volume
```

查看并保存cinder-volume用户的keyring文件

```
$ sudo ceph auth get client.cinder-volume
exported keyring for client.cinder-volume
[client.cinder-volume]
	key = AQAf3oVcN2nMORAAl740sqdkcwE/8a/niSTIeg==
	caps mon = "allow r"
	caps osd = "allow rwx pool=volume"

$ ceph auth get client.cinder-volume -o /var/openstack/ceph/ceph.client.cinder-volume.keyring
exported keyring for client.cinder-volume
```

创建cinder-backup用户, 并给volume和backups存储池权限

```
$ ceph auth get-or-create client.cinder-backup
[client.cinder-backup]
	key = AQDH3oVcaAfVJxAAMvwYBYLKNP86OkT6lPNMRQ==

$ ceph auth caps client.cinder-backup mon 'allow r' osd 'allow rwx pool=volume, allow rwx pool=backups'
updated caps for client.cinder-backup
```

查看并保存cinder-backup用户的KeyRing文件

```
$ ceph auth get client.cinder-backup
exported keyring for client.cinder-backup
[client.cinder-backup]
	key = AQDH3oVcaAfVJxAAMvwYBYLKNP86OkT6lPNMRQ==
	caps mon = "allow r"
	caps osd = "allow rwx pool=volume, allow rwx pool=backups"

$ ceph auth get client.cinder-backup -o /var/openstack/ceph/ceph.client.cinder-backup.keyring
exported keyring for client.cinder-backup
```

### 1.2.3 创建Nova用户

创建nova用户, 并给vm存储池权限

```
$ ceph auth get-or-create client.nova
[client.nova]
	key = AQA334VczU4tOBAAdvUyAv2wsn02MdQiW4o8sg==

$ ceph auth caps client.nova mon 'allow r' osd 'allow rwx pool=vm'
updated caps for client.nova
```

查看并保存nova用户的keyring文件

```
$ sudo ceph auth get client.nova
exported keyring for client.nova
[client.nova]
	key = AQA334VczU4tOBAAdvUyAv2wsn02MdQiW4o8sg==
	caps mon = "allow r"
	caps osd = "allow rwx pool=vm"

$ ceph auth get client.nova -o /var/openstack/ceph/ceph.client.nova.keyring
exported keyring for client.nova
```

## 1.3 同步ceph配置文件以及用户keyring文件

```
$ ssh {部署节点} sudo tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf
$ scp /var/openstack/ceph/* root@{部署节点}:/etc/ceph/
```

# 2 配置Kolla-Ansible

按照AutoStack配置, 在部署节点, 其配置阶段做下面关于ceph的工作

## 2.1 配置服务

在配置阶段, 修改global.yml的下面配置项

```
# 禁止在当前节点部署ceph
enable_ceph: "no"

# 开启cinder服务
enable_cinder: "yes"

# 开启cinder glance和nova的后端ceph功能
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"

# cinder backup功能
enable_cinder_backup: "yes"
```

根据自己的情况, 修改global.yml文件

kolla网络相关参数

- network\_interface：为下面几种interface提供默认值
- api_interface: 用于management network,即openstack内部服务间通信，以及服务于数据库通信的网络。这是系统的风险点，所以，这个网络推荐使用内网，不能够接出到外网，默认值为：network_interface
- kolla_external_vip_interface: 这是一个公网网段，当你想要HAProxy Public endpoint暴漏在不同的网络中的时候，需要用到。当kolla_enables_tls_external设置为yes的时候，是个必选项。默认值为：network\_interface
- storage\_interface: 虚拟机与Ceph通信接口，这个接口负载较重，推荐放在10Gig网口上。默认值为：network\_interface
- cluster\_interface: 这是Ceph用到的另外一个接口，用于数据的replication，这个接口同样负载很重，当其成为bottleneck的时候，会影响数据的一致性和整个集群的性能。默认：network_interface
- tunnel_interface: 这个接口用于虚拟机与虚拟机之间的通信，通过 tunneled网路进行，比如：Vxlan，GRE等，默认为：network_interface
- neutron_external_interface: 这是Neutron提供VM对外网络的接口，Neutron会将其绑定在br-ex上，既可以是flat网络，也可以是tagged vlan网络，必须单独设置

使用ceph的话需要修改storage\_interface和cluster\_interface

## 2.2 配置Glance

配置glance使用glance用户以及images存储池

在kolla\-ansible配置目录下创建目录glance

```
$ mkdir -p /etc/kolla/config/glance

$ cat /etc/kolla/config/glance/glance-api.conf
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
```

新增glance的ceph客户端配置和glance用户的keyring文件, 参照ceph.conf文件内容

```
$ cat /etc/kolla/config/glance/ceph.conf
[global]
fsid = 9853760c-b976-4e3c-8228-9f7a76c336bc
mon_initial_members = BJ-IDC1-10-10-31-26
mon_host = 10.10.31.26
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

$ cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/ceph.client.glance.keyring
```

## 2.3 配置Cinder

配置Cinder卷服务使用Ceph的cinder-volume用户使用volume存储池，Cinder卷备份服务使用Ceph的cinder-backup用户使用backups存储池：

```
$ mkdir -p /etc/kolla/config/cinder

$ cat /etc/kolla/config/cinder/cinder-volume.conf
[DEFAULT]
enabled_backends=rbd-1

[rbd-1]
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=cinder-volume
backend_host=rbd:volume
rbd_pool=volume
volume_backend_name=rbd-1
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid = {{ cinder_rbd_secret_uuid }}

$ cat /etc/kolla/config/cinder/cinder-backup.conf
[DEFAULT]
backup_ceph_conf=/etc/ceph/ceph.conf
backup_ceph_user=cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool=backups
backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
```

新增Cinder的卷服务和卷备份服务的Ceph客户端配置和KeyRing文件：

```
$ cp /etc/kolla/config/glance/ceph.conf /etc/kolla/config/cinder/ceph.conf

$ mkdir -p /etc/kolla/config/cinder/cinder-backup/ /etc/kolla/config/cinder/cinder-volume/

$ cp -v /etc/ceph/ceph.client.cinder-volume.keyring /etc/kolla/config/cinder/cinder-backup/ceph.client.cinder-volume.keyring

$ cp -v /etc/ceph/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup/ceph.client.cinder-backup.keyring

$ cp -v /etc/ceph/ceph.client.cinder-volume.keyring /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder-volume.keyring
```

## 2.4 配置nova

配置Nova使用Ceph的nova用户使用vm存储池：

```
$ mkdir -p /etc/kolla/config/nova

$ cat /etc/kolla/config/nova/nova-compute.conf
[libvirt]
images_rbd_pool=vm
images_type=rbd
images_rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=nova
```

新增nova的客户端配置和keyring文件

```
$ cp -v /etc/kolla/config/glance/ceph.conf /etc/kolla/config/nova/ceph.conf

$ cp -v /etc/ceph/ceph.client.nova.keyring /etc/kolla/config/nova/ceph.client.nova.keyring

$ cp -v /etc/ceph/ceph.client.cinder-volume.keyring /etc/kolla/config/nova/ceph.client.cinder.keyring
```

注: 这里的nova使用cinder-volume的keyring文件必须改为cinder.keyring

# 3 部署

按照AutoStack的Deployment开始部署
