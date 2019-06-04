
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->



<!-- /code_chunk_output -->


QEMU提供了一套面向对象编程的模型——QOM，即QEMU Object Module，几乎所有的设备如CPU、内存、总线等都是利用这一面向对象的模型来实现的。QOM模型的实现代码位于**qom/文件夹**下的文件中。对于开发者而言，只要知道如何利用QOM模型创建类和对象就可以了，但是开发者只有理解了QOM的相关数据结构，才能清楚如何利用QOM模型。因此本文先对QOM的必要性展开叙述，然后说明QOM的相关数据结构，在读者了解了QOM的数据结构的基础上，我们向读者阐述如何使用QOM模型创建新对象和新类，在下一篇中，介绍QOM是如何实现的。同时对于阅读代码的人来说，理解和掌握QOM是学习QEMU代码的重要一步。

为什么QEMU中要实现对象模型

各种架构CPU的模拟和实现

QEMU中要实现对各种CPU架构的模拟，而且对于一种架构的CPU，比如X86\_64架构的CPU，由于包含的特性不同，也会有不同的CPU模型。任何CPU中都有CPU通用的属性，同时也包含各自特有的属性。为了便于模拟这些CPU模型，面向对象的变成模型是必不可少的。

模拟device与bus的关系

在主板上，**一个device**会通过**bus**与其他的device相连接，一个device上可以通过不同的bus端口连接到其他的device，而其他的device也可以进一步通过bus与其他的设备连接，同时一个bus上也可以连接多个device，这种device连bus、bus连device的关系，qemu是需要模拟出来的。为了方便模拟设备的这种特性，面向对象的编程模型也是必不可少的。

