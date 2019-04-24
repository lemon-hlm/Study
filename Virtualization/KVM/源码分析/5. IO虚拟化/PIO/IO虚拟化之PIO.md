
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1. PIO](#1-pio)
* [2. PIO在KVM中](#2-pio在kvm中)
* [3. PIO运行在QEMU](#3-pio运行在qemu)

<!-- /code_chunk_output -->

## 1. PIO

Port IO，所谓端口IO，x86上使用in、out指令进行访问。和内存的地址空间完全隔离。

80386的I/O指令使得处理器可以访问I/O端口，以便从外设输入数据，或者向外设发送数据。这些指令有一个指定I/O空间端口地址的操作数。有两类的I/O指令：

1、 在寄存器指定的地址传送一个数据（字节、字、双字）。

2、 传送指定内存中的一串数据（字节串、字串、双字串）。这些被称作为“串 I/O指令”或者说“块I/O指令”。

有IN/OUT INS/OUTS指令

cat /proc/ioports查看当前OS的所有的ioports：

```
[root@dell-cicada ~]# cat /proc/ioports 
0000-03bb : PCI Bus 0000:00
  0000-001f : dma1
  0020-0021 : pic1
  0040-0043 : timer0
  0050-0053 : timer1
  0060-0060 : keyboard
  0064-0064 : keyboard
  0070-0071 : rtc0
  0080-008f : dma page reg
  00a0-00a1 : pic2
  00b2-00b2 : APEI ERST
  00c0-00df : dma2
  00f0-00ff : fpu
  02f8-02ff : serial
03bc-03df : PCI Bus 0000:00
  03c0-03df : vga+
03e0-0cf7 : PCI Bus 0000:00
  03f8-03ff : serial
  0400-0403 : ACPI PM1a_EVT_BLK
  0404-0405 : ACPI PM1a_CNT_BLK
  0408-040b : ACPI PM_TMR
  0410-0415 : ACPI CPU throttle
  0420-042f : ACPI GPE0_BLK
  0430-0433 : iTCO_wdt.0.auto
    0430-0433 : iTCO_wdt
  0450-0450 : ACPI PM2_CNT_BLK
  0460-047f : iTCO_wdt.0.auto
    0460-047f : iTCO_wdt
  0500-053f : pnp 00:06
  0540-057f : pnp 00:06
  0600-061f : pnp 00:06
  0800-081f : pnp 00:06
  0880-0883 : pnp 00:06
  0ca0-0ca5 : pnp 00:06
  0ca8-0ca8 : pnp 00:09
    0ca8-0ca8 : ipmi_si
  0cac-0cac : pnp 00:09
    0cac-0cac : ipmi_si
0cf8-0cff : PCI conf1
1000-7fff : PCI Bus 0000:00
  2000-2fff : PCI Bus 0000:02
    2000-20ff : 0000:02:00.0
      2000-20ff : megasas: LSI
  3020-303f : 0000:00:1f.2
    3020-303f : ahci
  3040-305f : 0000:00:11.4
    3040-305f : ahci
  3060-3067 : 0000:00:1f.2
    3060-3067 : ahci
  3068-306f : 0000:00:1f.2
    3068-306f : ahci
  3070-3077 : 0000:00:11.4
    3070-3077 : ahci
  3078-307f : 0000:00:11.4
    3078-307f : ahci
  3080-3083 : 0000:00:1f.2
    3080-3083 : ahci
  3084-3087 : 0000:00:1f.2
    3084-3087 : ahci
  3088-308b : 0000:00:11.4
    3088-308b : ahci
  308c-308f : 0000:00:11.4
    308c-308f : ahci
8000-ffff : PCI Bus 0000:80
```

常见的port 40---timer，60---keyboard等等。这是业界习惯。

## 2. PIO在KVM中

Guest在使用in、out指令的时候，Host会感知。Host中会在arch/x86/kvm/emulate.c中处理：

```
static int em_in(struct x86_emulate_ctxt *ctxt)
{   
    if (!pio_in_emulated(ctxt, ctxt->dst.bytes, ctxt->src.val,
                 &ctxt->dst.val))
        return X86EMUL_IO_NEEDED;
    
    return X86EMUL_CONTINUE;
}   
    
static int em_out(struct x86_emulate_ctxt *ctxt)
{   
    ctxt->ops->pio_out_emulated(ctxt, ctxt->src.bytes, ctxt->dst.val,                                                                                                                                      
                    &ctxt->src.val, 1);
    /* Disable writeback. */
    ctxt->dst.type = OP_NONE;
    return X86EMUL_CONTINUE;
} 
```

以em\_in为例，它调用pio\_in\_emulated()，该函数定义如下(X86)：

```
static const struct x86_emulate_ops emulate_ops = {
    ······
    .pio_in_emulated     = emulator_pio_in_emulated,
    .pio_out_emulated    = emulator_pio_out_emulated,
    ······
};
```

通过追踪emulate\_ops调用关系，会发现是通过struct kvm\_x86\_ops vmx\_x86\_ops的vcpu\_create操作添加的映射

```
static struct kvm_x86_ops vmx_x86_ops = {
    ···
    .vcpu_create = vmx_create_vcpu, 
    ···
}
```     

通过查找vcpu\_create的调用关系，发现是函数kvm\_vm\_ioctl的case KVM\_CREATE\_VCPU调用

```
static long kvm_vm_ioctl(struct file *filp,
               unsigned int ioctl, unsigned long arg) 
{                 
    struct kvm *kvm = filp->private_data;
    void __user *argp = (void __user *)arg;
    int r;    
                  
    if (kvm->mm != current->mm)
        return -EIO;
    switch (ioctl) {
    case KVM_CREATE_VCPU:
        r = kvm_vm_ioctl_create_vcpu(kvm, arg);
        break;
}
```

所以是创建每个vcpu（不同CPU架构可能不同）时候根据架构不同进行的操作映射。

我们继续查看in指令的模拟。

## 3. PIO运行在QEMU

