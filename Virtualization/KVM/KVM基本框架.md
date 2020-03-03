
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [背景](#背景)
	* [开源全虚拟化方案](#开源全虚拟化方案)
	* [历史](#历史)
	* [云平台对KVM的支持](#云平台对kvm的支持)
* [KVM架构](#kvm架构)
	* [虚拟化管理接口逻辑分层（libvirt、qemu、kvm）](#虚拟化管理接口逻辑分层libvirt-qemu-kvm)
* [参考](#参考)

<!-- /code_chunk_output -->

# 背景

## 开源全虚拟化方案

- 支持体系结构：x86（32位、64位）、IA64、PowerPC、S390
- 依赖x86硬件支持：Intel VT-x/AMD-V
- 内核模块，使得Linux内核成为hypervisor

## 历史

- 2006年10月 以色列公司Qumranet发布KVM
- 2006年12月   KVM合入内核(Linux 2.6.20rc)
  - 2007年2月 Linux2.6.20正式版发布
- 2008年9月    Redhat以1.07亿美元收购Qumranet
- 2009年9月    RHEL5.4开始支持KVM（同时支持Xen）
- 2010年11月  RHEL6.0之后仅支持KVM

## 云平台对KVM的支持

- OpenStack、Eucalyptus、AbiCloud等同时支持KVM和Xen

# KVM架构

## 虚拟化管理接口逻辑分层（libvirt、qemu、kvm）


# 参考

虚拟化中KVM, Xen, Qemu的区别：http://yansu.org/2013/03/20/different-bewteen-kvm-xen-qemu.html

qemu,kvm,qemu-kvm,xen,libvirt的区别：https://my.oschina.net/qefarmer/blog/386843
