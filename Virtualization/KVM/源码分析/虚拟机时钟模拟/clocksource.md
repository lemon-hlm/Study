1. 查看操作系统的clocksource：

在/sys/devices/system/clocksource/clocksource0目录下；

available_clocksource是当前所有可用的clocksource；

current_clocksource是当前正在使用的clocksource。

2. clocksource management

主要逻辑代码在linux-4.0.4/kernel/time/clocksource.c中实现：