32位汇编参见《Linux 汇编入门 - 32位》

## 一、不同

1. 系统调用号不同。比如x86中sys_write是4，sys_exit是1;而x86_64中sys_write是1, sys_exit是60。linux系统调用号实际上定义在/usr/include/asm/unistd_32.h和/usr/include/asm/unistd_64.h中。

2. 系统调用所使用的寄存器不同，x86_64中使用与eax对应的rax传递系统调用号，但是 x86_64中分别使用rdi/rsi/rdx传递前三个参数，而不是x86中的ebx/ecx/edx。

- 对于32位程序，应调用int $0x80进入系统调用，将系统调用号传入eax，各个参数按照ebx、ecx、edx的顺序传递到寄存器中，系统调用返回值储存到eax寄存器。

- 对于64位程序，应调用syscall进入系统调用，将系统调用号传入rax，各个参数按照rdi、rsi、rdx的顺序传递到寄存器中，系统调用返回值储存到rax寄存器。

3. 系统调用使用“syscall”而不是“int 80”

4. 32位和64位程序的地址空间范围不同。

例子：

32位

```
global _start  
_start:  
        jmp short ender  
starter:  
        xor eax, eax    ;clean up the registers  
        xor ebx, ebx  
        xor edx, edx  
        xor ecx, ecx  
  
        mov al, 4       ;syscall write  
        mov bl, 1       ;stdout is 1  
        pop ecx         ;get the address of the string from the stack  
        mov dl, 5       ;length of the string  
        int 0x80  
  
        xor eax, eax  
        mov al, 1       ;exit the shellcode  
        xor ebx,ebx  
        int 0x80  
ender:  
        call starter    ;put the address of the string on the stack  
        db 'hello',0x0a  
```

64位

```
global _start           ; global entry point export for ld  
_start:  
    jump short string   ; get message addr  
code:  
    ; sys_write(stdout, message, length)  
    pop     rsi         ; message address  
    mov     rax, 1      ; sys_write  
    mov     rdi, 1      ; stdout  
    mov     rdx, 13     ; message string length + 0x0a  
    syscall  
  
    ; sys_exit(return_code)  
    mov     rax, 60     ; sys_exit  
    mov     rdi, 0      ; return 0 (success)  
    syscall  
  
string:  
    call    code  
    db 'Hello!',0x0a    ; message and newline
```



## 二、兼容

由于硬件指令的兼容，32位的程序在用户态不受任何影响的运行，由于**内核保留了0x80号中断作为32位程序的系统调用服务**，因此32位程序可以安全触发0x80号中断使用系统调用，由于内核为0x80中断安排了另一套全新的系统调用表，因此可以安全地转换数据类型成一致的64位类型，再加上应用级别提供了两套c库，可以使64位和32位程序链接不同的库。

## 三、内联汇编

### 1. 程序代码和问题

首先看如下一段简单的C程序（test.c）

```
#include <unistd.h>
int main(){
    char str[] = "Hello\n";
    write(0, str, 6);
    return 0;
}
```

这段程序调用了write函数，其接口为：

```
int write(int fd /*输出位置句柄*/, const char* src /*输出首地址*/， int len /*长度*/)
```

fd为0则表示输出到控制台。因此上述程序的执行结果为：向控制台输出一个长度为6的字符串"Hello\n"。 

在控制台调用gcc test.c，可以正确输出。

为了更好地理解在汇编代码下的系统调用过程，可把上述代码改写成内联汇编的格式（参照《用asm内联汇编实现系统调用》）

```
test_asm_A.c
int main(){
    char str[] = "Hello\n";
    asm volatile(
        "int $0x80\n\t"
        :
        :"a"(4), "b"(0), "c"(str), "d"(6)
        );
    return 0;
}
```

其中，4是write函数的系统调用号，ebx/ecx/edx是系统调用的前三个参数。 

然而，执行gcc test\_asm\_A.c编译后，再运行程序，发现程序没有任何输出。一个很奇怪的问题是，如果采用如下test\_asm\_B.c的写法，则程序可以正常地输出：

```
test_asm_B.c
#include <stdlib.h>

int main(){
    char *str = (char*)malloc(7 * sizeof(char));
    strcpy(str, "Hello\n");
    asm volatile(
        "int $0x80\n\t"
        :
        :"a"(4), "b"(0), "c"(str), "d"(6)
        );
    free(str);
    return 0;
}
```

输出如下：

```
[root@tsinghua-pcm C]# gcc test_asm_B.c
test_asm_B.c: 在函数‘main’中:
test_asm_B.c:4:2: 警告：隐式声明与内建函数‘strcpy’不兼容 [默认启用]
  strcpy(str, "Hello\n");
  ^
[root@tsinghua-pcm C]# ./a.out 
Hello
[root@tsinghua-pcm C]# cp inline_assembly.c test_asm_A.c
[root@tsinghua-pcm C]# gcc test_asm_A.c
[root@tsinghua-pcm C]# ./a.out 
[root@tsinghua-pcm C]# 
```

两段代码唯一的区别，是test\_asm\_A.c中的str存储在栈空间，而test\_asm\_B.c中的str存储在堆空间。 

那么，为什么存储位置的不同会造成完全不同的结果呢？

### 2. 原因分析

将上述代码用32位的方式编译，即gcc test\_asm\_A.c -m32和gcc test\_asm\_B.c -m32，可以发现两段代码都能正确输出。这说明，上述代码按32位编译，可以得到正确的结果。 

```
[root@tsinghua-pcm C]# gcc test_asm_A.c -m32
[root@tsinghua-pcm C]# ./a.out 
Hello
[root@tsinghua-pcm C]# gcc test_asm_B.c -m32
test_asm_B.c: 在函数‘main’中:
test_asm_B.c:4:2: 警告：隐式声明与内建函数‘strcpy’不兼容 [默认启用]
  strcpy(str, "Hello\n");
  ^
[root@tsinghua-pcm C]# ./a.out 
Hello
```

如果没有-m32标志，则gcc默认按照64位方式编译。32位和64位程序在编译时有如下区别（上面已经提到）：

- 32位和64位程序的地址空间范围不同。

- 32位和64位程序的系统调用号不同，如本例中的write，在32位系统中调用号为4，在64位系统中则为1。

- 对于32位程序，应调用int $0x80进入系统调用，将系统调用号传入eax，各个参数按照ebx、ecx、edx的顺序传递到寄存器中，系统调用返回值储存到eax寄存器。

- 对于64位程序，应调用syscall进入系统调用，将系统调用号传入rax，各个参数按照rdi、rsi、rdx的顺序传递到寄存器中，系统调用返回值储存到rax寄存器。

再看上面两段代码，它们都是调用int $0x80进入系统调用，却**按照64位方式编译**，则会出现如下不正常情形：

- 程序的地址空间是64位地址空间。

- 0x80号中断进入的是32位系统调用函数，因此仍按照32位的方式来解释系统调用，即所有寄存器只考虑低32位的值。
 
再看程序中传入的各个参数，系统调用号（4），第1个和第3个参数（0和6）都是32位以内的，但是str的地址是64位地址，在0x80系统调用中只有低32位会被考虑。 

这样，test\_asm\_A.c不能正确执行，而test\_asm\_B.c可以正确执行的原因就很明确了：

- 在test\_asm\_A.c中，str存储在栈空间中，**而栈空间在系统的高位开始**，只取低32位地址，得到的是错误地址。

- 在test\_asm\_B.c中，str存储在堆空间中，**而堆空间在系统的低位开始**，在这样一个小程序中，str地址的高32位为0，只有低32位存在非零值，因此不会出现截断错误。

对于存储器堆栈，可以查看 Computer_Architecture/x86/指令系统/ 下内容

可见，test\_asm\_B.c正确执行只是一个假象。由于堆空间从低位开始，如果开辟空间过多，堆空间也进入高位的时候，这段代码同样可能出错。

### 3. 64位系统的系统调用代码

给出64位系统下可正确输出的asm系统调用代码：

```
//test_asm_C.c
int main(){
    char str[] = "Hello\n";
    //注意：64位系统调用中，write函数调用号为1
    asm volatile(
        "mov %2, %%rsi\n\t"
        "syscall"
        :
        :"a"(1), "D"(0), "b"(str), "d"(6)
        );
    return 0;
}
```