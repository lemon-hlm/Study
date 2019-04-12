openstack集群已经对接后端ceph了, 但是使用的是以太网IP, 想要切换到IB IP. 

```
# vim /etc/hosts
10.121.2.125 compute1
10.5.138.125 compute1
```

这中间涉及到:

- ceph集群更改ip并重启
- openstack这边需要更改配置

ceph集群更改后, ceph.conf文件肯定会变, 其它的比如client的key不变

openstack修改如下

修改/etc/kolla/global.yml

```
storage_interface: "ib0"
```

修改/etc/kolla/config/下的配置, 其实只需要修改ceph.conf即可

```
# ll /etc/kolla/config/*/ceph.conf
-rwxr-xr-x. 1 root root 267 3月  25 20:30 /etc/kolla/config/cinder/ceph.conf
-rwxr-xr-x. 1 root root 267 3月  25 20:30 /etc/kolla/config/glance/ceph.conf
-rwxr-xr-x. 1 root root 267 3月  25 20:30 /etc/kolla/config/nova/ceph.conf

# cat /etc/kolla/config/*/ceph.conf
[global]
fsid = f09aeb2a-0927-4bd1-8682-c43a1cddf826
mon_initial_members = SH-IDC1-10-5-8-126, SH-IDC1-10-5-8-127, SH-IDC1-10-5-8-128
mon_host = 10.5.8.126,10.5.8.127,10.5.8.128
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
[global]
fsid = f09aeb2a-0927-4bd1-8682-c43a1cddf826
mon_initial_members = SH-IDC1-10-5-8-126, SH-IDC1-10-5-8-127, SH-IDC1-10-5-8-128
mon_host = 10.5.8.126,10.5.8.127,10.5.8.128
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
[global]
fsid = f09aeb2a-0927-4bd1-8682-c43a1cddf826
mon_initial_members = SH-IDC1-10-5-8-126, SH-IDC1-10-5-8-127, SH-IDC1-10-5-8-128
mon_host = 10.5.8.126,10.5.8.127,10.5.8.128
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```

然后

修改/etc/kolla/config/需要执行

```
# kolla-ansible -i ../../multinode reconfigure 
```

reconfigure用来reconfigure OpenStack service.

修改/etc/kolla/global.yml

```
# kolla-ansible -i ../../multinode upgrade
# kolla-ansible -i ../../multinode post-deploy
```

upgrade用来upgrade现有openstack环境