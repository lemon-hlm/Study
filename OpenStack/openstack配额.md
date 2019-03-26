# 1 计算资源

查看默认配额

```
$ nova quota-defaults
+----------------------+--------+
| Quota                | Limit  |
+----------------------+--------+
| instances            | 500    |
| cores                | 2000   |
| ram                  | 512000 |
| metadata_items       | 128000 |
| key_pairs            | 10000  |
| server_groups        | 10000  |
| server_group_members | 10000  |
+----------------------+--------+
```

修改默认配额

```
$ nova quota-class-update default key value
```

获取租户ID

```
$ openstack project list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 1a3a9bd31e2b49b4893286535c825b97 | admin   |
| 5cae732686624bfa8aa6c3db45818c7a | service |
+----------------------------------+---------+
```

查看租户的配额

```
nova quota-show --tenant 1a3a9bd31e2b49b4893286535c825b97
+----------------------+--------+
| Quota                | Limit  |
+----------------------+--------+
| instances            | 500    |
| cores                | 400    |
| ram                  | 960000 |
| metadata_items       | 128000 |
| key_pairs            | 10000  |
| server_groups        | 10000  |
| server_group_members | 10000  |
+----------------------+--------+
```

修改租户的配额

```
$ nova quota-update --instances 500 1a3a9bd31e2b49b4893286535c825b97
```

# 2 卷资源

查看租户默认配额

```
$ cinder quota-defaults 1a3a9bd31e2b49b4893286535c825b97
+----------------------+--------+
| Property             | Value  |
+----------------------+--------+
| backup_gigabytes     | 1000   |
| backups              | 10     |
| gigabytes            | 400000 |
| groups               | 10     |
| per_volume_gigabytes | -1     |
| snapshots            | 50000  |
| volumes              | 500    |
+----------------------+--------+
```

修改默认配额

```
cinder quota-class-update default --backups 50
```

查看租户的cinder配额

```
$ cinder quota-show 1a3a9bd31e2b49b4893286535c825b97
+----------------------+--------+
| Property             | Value  |
+----------------------+--------+
| backup_gigabytes     | 1000   |
| backups              | 500    |
| gigabytes            | 400000 |
| groups               | 500    |
| per_volume_gigabytes | -1     |
| snapshots            | 50000  |
| volumes              | 500    |
+----------------------+--------+
```

更新租户的配额

```
cinder quota-update --volumes 500 1a3a9bd31e2b49b4893286535c825b97
```

# 3 网络资源

查看租户默认配额

```
$ neutron quota-default-show 1a3a9bd31e2b49b4893286535c825b97
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+---------------------+-------+
| Field               | Value |
+---------------------+-------+
| floatingip          | 50    |
| network             | 100   |
| port                | 500   |
| rbac_policy         | 10    |
| router              | 10    |
| security_group      | 10    |
| security_group_rule | 100   |
| subnet              | 100   |
| subnetpool          | -1    |
+---------------------+-------+
```

修改租户默认配额

```

```

# 4 镜像资源



