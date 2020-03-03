https://blog.csdn.net/pointfish/article/details/6529709?utm_source=copy

\_\_asm\_\_ \_\_volatile\_\_("": : :"memory")

memory强制**gcc编译器**假设**RAM所有内存单元**均**被汇编指令修改**，这样**cpu**中的**registers**和**cache**中已缓存的内存单元中的**数据将作废**。cpu将不得不在需要的时候**重新读取内存中的数据**。这就**阻止**了cpu又将**registers，cache中的数据**用于去**优化指令**，而避免去访问内存。

The GCC manual says using 'asm volatile' prevents the asm instruction from being 'movedsignificantly' and adding 'memory' to the clobber list prevents GCC from keeping memory values cached in registers across the assembler instruction, but also says 'GCC will perform some optimizations across a volatile asm instruction' but doesn't explain what.

1）\_\_asm\_\_用于指示编译器在此插入汇编语句。
2）\_\_**volatile**\_\_用于告诉编译器，**严禁**将此处的汇编语句与其它的语句重组合优化。即：**原原本本按原来的样子处理这这里的汇编**。
3）memory强制gcc编译器假设RAM所有内存单元均被汇编指令修改，这样cpu中的registers和cache中已缓存的内存单元中的数据将作废。cpu将不得不在需要的时候重新读取内存中的数据。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。
4）"":::表示这是个**空指令**。**barrier**()**不用在此插入一条串行化汇编指令**。