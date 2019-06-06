
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [evacuate实例](#evacuate实例)

<!-- /code_chunk_output -->

# evacuate实例

如果您需要把一个实例从一个有故障或已停止运行的 compute 节点上移到同一个环境中的其它主机服务器上时，可以使用 nova evacuate 命令对实例进行撤离（evacuate）。

- 撤离的操作只有在实例的磁盘在共享存储上，或实例的磁盘是块存储卷时才有效。因为在其它情况下，磁盘无法被新 compute 节点访问。

- 实例只有在它所在的服务器停止运行的情况下才可以被撤离；如果服务器没有被关闭，evacuate 命令会运行失败。



参考

- https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux_OpenStack_Platform/6/html/Administration_Guide/section-evacuation.html

- https://blog.fabian4.cn/2016/10/27/nova-evacuate/