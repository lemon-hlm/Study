https://baike.baidu.com/item/CPUID/5559847

CPU ID 指用户计算机当今的信息**处理器的信息**。 信息包括型号，信息处理器家庭，高速缓存尺寸，钟速度和制造厂codename 等。 通过查询可以知道一些信息：晶体管数，针脚类型，尺寸等。

# 1 什么是cpuid指令

**CPUID指令**是intel IA32架构下**获得CPU信息**的汇编指令，可以得到CPU类型，型号，制造商信息，商标信息，序列号，缓存等一系列CPU相关的东西。

# 2 cpuid指令的使用

cpuid使用**eax作为输入参数**，**eax，ebx，ecx，edx作为输出参数**，举个例子:

```x86asm
__asm
{
mov eax, 1
cpuid
...
}
```

以上代码**以1为输入参数**，执行**cpuid**后，**所有寄存器的值都被返回值填充**。针对不同的输入参数eax的值，输出参数的意义都不相同。

# 3 获得CPU的制造商信息(Vender ID String)

把**eax = 0作为输入参数**，可以得到CPU的**制造商信息**。

cpuid指令执行以后，会返回一个**12字符的制造商信息**，前四个字符的ASC码按**低位到高位放在ebx**，**中间四个放在edx**，最后四个字符放在**ecx**。比如说，对于intel的cpu，会返回一个“GenuineIntel”的字符串，返回值的存储格式为:

```
31 23 15 07 00
EBX| u (75)| n (6E)| e (65)| G (47)
EDX| I (49)| e (65)| n (6E)| i (69)
ECX| l (6C)| e (65)| t (74)| n (6E)
```

# 4 获得CPU商标信息(Brand String)

由于商标的字符串很长(48个字符)，所以不能在一次cpuid指令执行时全部得到，所以intel把它分成了3个操作，eax的输入参数分别是0x80000002,0x80000003,0x80000004，每次返回的16个字符，按照从低位到高位的顺序依次放在eax, ebx, ecx, edx。因此，可以用循环的方式，每次执行完以后保存结果，然后执行下一次cpuid。

# 5. 检测CPU特性(CPU feature)

CPU的特性可以通过cpuid获得，参数是eax = 1，返回值放在edx和ecx，通过验证edx或者ecx的某一个bit，可以获得CPU的一个特性是否被支持。比如说，edx的bit 32代表是否支持MMX，edx的bit 28代表是否支持Hyper-Threading，ecx的bit 7代表是否支持speed step。