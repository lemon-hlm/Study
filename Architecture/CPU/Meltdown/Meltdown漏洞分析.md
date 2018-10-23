[TOC]

http://mp.weixin.qq.com/s?__biz=MzI3MTUxOTYyMA==&mid=2247484002&idx=1&sn=19393c4abd317aea0ab8d535559603b2&chksm=eac1d889ddb6519f2190f4f24d7679f34f443db342700332b4749e360c8011cb0d901d15023f&mpshare=1&scene=1&srcid=01274rIaA8vFPzDBWpMKvytq#rd

# 0 概述

![config](./images/2.png)

**Meltdown breaks the most fundamental isolation between user applications and the operating system**. This attack allows a program to access the memory, and thus also the secrets, of other programs and the operating system.

官网: https://meltdownattack.com/

![config](./images/3.png)

**Spectre breaks the isolation between different applications**. It allows an attacker to trick error-free programs, which follow best practices, into leaking their secrets. In fact, the safety checks of said best practices actually increase the attack surface and may make applications more susceptible to Spectre.

官网同上

一个形象的场景形容meltdown如下:

1. 三个人A, B和C去餐厅点餐, 排队, C排在B后面, B在A后面
2. A点了面条, 后厨听到开始做面条, 还没做完
3. B向点餐服务员喊说点饺子(指令顺序进入), 服务员说, **前面还有一位, 稍等下**. 但**后厨听到后开始做饺子**, 很快做好了, 这就是**指令预取**
4. A拿到面条走了, **轮到B点餐**了, 此时突然有事情, 没有吃饺子, 但是已经做好了, 然后被餐厅请出去了, 这就是**异常处理**
5. C排在B后面, 说: 饺子, 面条, 包子各来一份, 饿死了, 哪个快先给我哪个
6. C拿到了"饺子"
7. C知道了B点了饺子

5, 6, 7就是cache攻击

# 1 CPU体系结构

![config](./images/1.jpeg)


# 相关论文

- Meltdown, https://meltdownattack.com/
- Spectre Attacks: Exploiting Speculative Execution, https://meltdownattack.com/
- Google Zero Projecthttps://googleprojectzero.blogspot.com/ 
- MeltdownPrime and SpectrePrime: Automatically-Synthesized Attacks Exploiting Invalidation-Based Coherence ProtocolsKPTI, Submitted on 11 Feb 2018
- Negative Result:Reading Kernel Memory From User Mode https://cyber.wtf/2017/07/28/negative-result-reading-kernel-memory-from-usermode/