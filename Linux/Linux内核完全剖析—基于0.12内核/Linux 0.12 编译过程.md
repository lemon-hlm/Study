- 问题1：

```
[root@tsinghua-pcm boot]# as -o head.o head.s 
head.s: Assembler messages:
head.s:43: Error: unsupported instruction `mov'
head.s:47: Error: unsupported instruction `mov'
head.s:59: Error: unsupported instruction `mov'
head.s:61: Error: unsupported instruction `mov'
head.s:136: Error: invalid instruction suffix for `push'
head.s:137: Error: invalid instruction suffix for `push'
head.s:138: Error: invalid instruction suffix for `push'
head.s:139: Error: invalid instruction suffix for `push'
head.s:140: Error: invalid instruction suffix for `push'
head.s:151: Error: invalid instruction suffix for `push'
head.s:152: Error: invalid instruction suffix for `push'
head.s:153: Error: invalid instruction suffix for `push'
head.s:154: Error: operand type mismatch for `push'
head.s:155: Error: operand type mismatch for `push'
head.s:161: Error: invalid instruction suffix for `push'
head.s:163: Error: invalid instruction suffix for `pop'
head.s:165: Error: operand type mismatch for `pop'
head.s:166: Error: operand type mismatch for `pop'
head.s:167: Error: invalid instruction suffix for `pop'
head.s:168: Error: invalid instruction suffix for `pop'
head.s:169: Error: invalid instruction suffix for `pop'
head.s:214: Error: unsupported instruction `mov'
head.s:215: Error: unsupported instruction `mov'
head.s:217: Error: unsupported instruction `mov'
head.s:231: Error: alignment not a power of 2
```

出错原因：

这是由于在64位机器上编译的原因，需要告诉编译器，我们要编译32位的code，在所有Makefile的AS后面添加 --32，CFLAGS中加-m32

解决办法：

修改head.s，在文件前加.code32

- 问题2：

```
gas -c -o boot/head.o boot/head.s
make: gas: Command not found
make: *** [boot/head.o] Error 127
```

出错原因：

gas、gld 的名称已经过时，现在GNU assembler的名称是 as


解决办法：

修改主 Makefile 文件

```
将 AS =gas 修改为 AS =as
将 LD =gld 修改为 LD =ld
```

或者

```
连接gas到as 
gld到ld
```

- 问题3：

```
gas -c -o boot/head.o boot/head.s

boot/head.s: Assembler messages:
boot/head.s:232: Error: alignment not a power of 2
```

出错原因：

.align 2 是汇编语言指示符，其含义是指存储边界对齐调整；

“2”表示把随后的代码或数据的偏移位置调整到地址值最后2比特位为零的位置（2^2），即按4字节对齐内存地址。

不过现在GNU as直接是写出对齐的值而非2的次方值了。

```
.align 2 应该改为 .align 4
.align 3 应该改为 .align 8
```

解决方案：

修改head.s文件

- 问题4

```
-nostdinc -Iinclude -c -o init/main.o init/main.c
gcc: error: unrecognized command line option ‘-fcombine-regs’
gcc: error: unrecognized command line option ‘-mstring-insns’
make: *** [init/main.o] Error 1
```

解决办法：

修改 Makefile 文件

将 -fcombine-regs -mstring-insns 删除或者注释掉 

- 问题5
 
```
init/main.c:23:29: error: static declaration of ‘fork’ follows non-static declaration

 static inline _syscall0(int,fork)

init/main.c:24:29: error: static declaration of ‘pause’ follows non-static declaration
 static inline _syscall0(int,pause)

init/main.c:26:29: error: static declaration of ‘sync’ follows non-static declaration
 static inline _syscall0(int,sync)
```

解决办法：

修改 init/main.c 文件

```
将 static inline _syscall0(int,fork) 修改为 inline _syscall0(int,fork)
static inline _syscall0(int,pause) 修改为 inline _syscall0(int,pause)
static inline _syscall1(int,setup,void *,BIOS) 修改为 inline _syscall1(int,setup,void *,BIOS)
static inline _syscall0(int,sync) 修改为 inline _syscall0(int,sync)
```