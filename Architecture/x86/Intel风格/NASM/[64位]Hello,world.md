
## 1. 代码

```
[bits 64]

global _start

section .data
message db "Hello, world!"

section .text 

_start:
    mov rax, 1
    mov rdx, 13
    mov rsi, message
    mov rdi, 1
    syscall

    mov rax, 60
    mov rdi, 0
    syscall
```

## 2. 可执行文件

### 2.1 生成目标文件

```
root@Gerry:/home/project/nasm# nasm -f elf64 -o hello-world.o  hello-world.asm
```

“-f elf64”说明让nasm生成一个elf64格式的目标文件。

**通过hexdump工具查看文件的机器码，适用于所有文件**

```
root@Gerry:/home/project/nasm# ll -h hello-world.o
-rw-r--r-- 1 root root 864 Nov 27 15:34 hello-world.o

root@Gerry:/home/project/nasm# file hello-world.o
hello-world.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

root@Gerry:/home/project/nasm# hexdump hello-world.o
0000000 457f 464c 0102 0001 0000 0000 0000 0000
0000010 0001 003e 0001 0000 0000 0000 0000 0000
0000020 0000 0000 0000 0000 0040 0000 0000 0000
0000030 0000 0000 0040 0000 0000 0040 0007 0003
0000040 0000 0000 0000 0000 0000 0000 0000 0000
*
0000080 0001 0000 0001 0000 0003 0000 0000 0000
0000090 0000 0000 0000 0000 0200 0000 0000 0000
00000a0 0011 0000 0000 0000 0000 0000 0000 0000
00000b0 0004 0000 0000 0000 0000 0000 0000 0000
···
0000340 000c 0000 0000 0000 0001 0000 0002 0000
0000350 0000 0000 0000 0000 0000 0000 0000 0000
0000360
```

通过hexdump查看文件的机器码可以看到一共有0x（360 - 0）= 864字节，和查出来的一致。

### 2.2 生成可执行文件

```
root@Gerry:/home/project/nasm# ld -o hello-world hello-world.o

root@Gerry:/home/project/nasm# file hello-world  
hello-world: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```

ld就是把目标文件组合，转换成可执行文件。

## 3. 理解寄存器

寄存器就是cpu内部的小容量的存储器，我们关心的主要是三类寄存器，数据寄存器，地址寄存器和通用寄存器。

- 数据寄存器保存数值，（比如整数和浮点值）
- 地址寄存器，保存内存中的地址
- 通用寄存器，既可以用过数据寄存器也可以用做地址寄存器。

汇编程序员大部分工作都是在操作这些寄存器。

## 4. 分析源码

```
[bits 64]
```

这是告诉nasm我们想要得到可以运行在64位处理器上的代码。

```
global _start
```

告诉nasm 用\_start 标记的代码段（section）应该被看成全局的，全局的部分通常允许其他的目标文件引用它，在我们的这个例子中，我们把\_start 段标记为全局的，以便让链接器知道我们的程序从哪里开始。

```
section .data
message db "Hello, World!"
```

告诉nasm后面跟着的代码是data段，data段包含全局和静态变量。

声明静态变量。message db "Hello, World!"，db被用来声明初始化数据，message是一个变量名，与"Hello, World!"关联。

```
section .text
```

告诉nasm把紧跟着的代码存到text段，text段有时候也叫做code段，它是包含可执行代码的目标文件的一部分。

最后，程序部分。

```
_start:
mov rax, 1
mov rdx, 13
mov rsi, message
mov rdi, 1
syscall

mov rax, 60
mov rdi, 0
syscall
```

第一行。\_start:，把它后面的代码和_start标记相关联起来。

```
mov rax, 1
mov rdx, 13
mov rsi, message
mov rdi, 1
```

上面这四行都是加载值到不同的寄存器里，RAX和RDX都是通用寄存器，我们使用他们分别保存1和13，RSI和RDI是源和目标数据索引寄存器，我们设置源寄存器RSI指向message，而目标寄存器指向1。

现在，当寄存器加载完毕后，我们有syscall指令，这是告诉计算机我们想要使用我们已经加载到寄存器的值执行一次系统调用，我们加载的第一个数也就是**RAX寄存器的值，告诉计算机我们想使用哪一个系统调用**，syscalls表和其对应的数字在文件/usr/include/x86\_64-linux-gnu/asm/unistd\_64.h中（32位在文件unistd\_32.h中）。

注意：上面针对的都是x86架构，功能号定义在文件/usr/include/bits/syscall.h中，然后针对不同架构有具体文件（如上面）

对比两个文件，会发现32位和64位的内容不同。

32位文件，会发现系统调用号1是退出，4是写
```
#ifndef _ASM_X86_UNISTD_32_H
#define _ASM_X86_UNISTD_32_H 1
 
#define __NR_restart_syscall 0
#define __NR_exit 1
#define __NR_fork 2
#define __NR_read 3
#define __NR_write 4
#define __NR_open 5
···
```

64位文件，会发现系统调用号1是写，60是退出

```
#ifndef _ASM_X86_UNISTD_64_H
#define _ASM_X86_UNISTD_64_H 1

#define __NR_read 0
#define __NR_write 1
#define __NR_open 2
#define __NR_close 3
#define __NR_stat 4
#define __NR_fstat 5
···
#define __NR_execve 59
#define __NR_exit 60
#define __NR_wait4 61
#define __NR_kill 62
#define __NR_uname 63
···
```

```
mov rax, 1
mov rdx, 13
mov rsi, message
mov rdi, 1
```

RAX里的1意味着我们想要调用write(int fd, const void* buf, size\_t bytes). 下一条指令 mov rdx,13，则是我们想要调用的函数write()的最后一个参数值，最后一个参数size\_t bytes指令了message的大小，此处 13也就是"Hello, World!"的长度。

下两条指令 mov rsi, message 和 mov rdi,1则分别作了其他两个参数，因此，当我们把他们放在一起，当要执行syscall的时候，我们是告诉计算机执行write(1, message, 13)，1就是标准输出stdout，因此本质上，我们是告诉计算机从message里取13个字节输出到stdout。其他使用默认参数。

```
mov rax, 60               
mov rdi, 0                
syscall
```

这个是退出，调用exit(0)

