
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 微处理器发展模式](#1-微处理器发展模式)
	* [1.1 Tick\-Tock模式](#11-tick-tock模式)
	* [1.2 Process\-Architecture\-Optimization模式](#12-process-architecture-optimization模式)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 微处理器发展模式

## 1.1 Tick\-Tock模式

Tick-Tock是Intel公司发展微处理器芯片设计制造业务的一种发展战略模式，在2007年正式提出。

Intel指出，每一次处理器**微架构(microarchitecture**)的更新和每一次**芯片制程**的更新，它们的时机应该错开，使他们的微处理器芯片设计制造业务更有效率地发展。“Tick-Tock”的名称源于时钟秒针行走时所发出的声响。

Intel指出，

- 每一次“**Tick**”代表着一代微架构的**处理器芯片制程**的更新，意在处理器性能几近相同的情况下，缩小芯片面积、减小能耗和发热量, 有时候也会引入新的指令, 比如2014年末的Broadwell；
- 而每一次“**Tock**”代表着在上一次“Tick”的芯片制程的基础上，更新**微处理器架构**，提升性能, 原来是每次tock定义一个新的微体系架构。但在进入 14nm 时明显碰壁, 2014年时候, 在Intel以微架构的较小更新的形式创建了tock(当时是Haswell)的新概念"tock refresh(当时是Haswell Refresh"), 其以改进Haswell为主, 不将其认为是新一代架构. 

一般一次“Tick-Tock”的周期为两年，“Tick”占一年，“Tock”占一年。

此策略常被许多计算机玩家戏称“挤牙膏策略”，因为每一代新处理器性能和前一代处理器性能的差距很短，就好像Haswell的4790K和Skylake的6700K那样

## 1.2 Process\-Architecture\-Optimization模式

2016年3月22日，Intel在 [Form 10-K](https://www.wikiwand.com/en/Form_10-K) 报告中宣布, 弃用"Tick\-Tock"模式, 采用三步"Process\-Architecture\-Optimization"模式, 即"制程、架构、优化". 具体来讲, 将Tick Tock放缓至三年一循环，即增加优化环节，进一步减缓实际更新的速度。

在此模式下, 三代处理器都将是同一个制程流程下生产,   

举例, Process是Broadwell(代表14nm第一代), Architecture是Skylake(代表这个微架构的第一代, 但是制程还是14nm), 第一代优化(Optimization)是Kaby Lake(2016年8月30). 第二代优化是Coffee Lake(2017年10月5日), 在14nm上共生产了4代().




# 参考

- Intel官方Tick\-Tock模式主页: https://www.intel.com/content/www/us/en/silicon-innovations/intel-tick-tock-model-general.html
- Intel 处理器架构演进: http://jcf94.com/2018/02/13/2018-02-13-intel/
- Tick-Tock维基百科: https://www.wikiwand.com/en/Tick%E2%80%93tock_model
- 中文Tick-Tock维基: https://zh.wikipedia.org/wiki/Intel_Tick-Tock
- Intel微处理器列表: https://zh.wikipedia.org/wiki/%E8%8B%B1%E7%89%B9%E5%B0%94%E5%BE%AE%E5%A4%84%E7%90%86%E5%99%A8%E5%88%97%E8%A1%A8


