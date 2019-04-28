
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->



<!-- /code_chunk_output -->

IA32\_MCG\_STATUS MSR描述了发生Machine Check exception后处理器的当前状态.

![](./images/2019-04-28-11-53-49.png)

- bit 0: 
- bit 1:
- bit 2: 设置后说明生成机器检查异常。 软件可以设置或清除此标志。 设置MCIP时发生第二次机器检查事件将导致处理器进入关闭状态。 有关处于关闭状态的处理器行为的信息，请参阅第6章“中断和异常处理”中的说明：“中断8双故障异常（#DF）”。
- bit 3: 设置后说明生成本地machine\-check exception. 这表示当前的机器检查事件仅传递给此逻辑处理器。