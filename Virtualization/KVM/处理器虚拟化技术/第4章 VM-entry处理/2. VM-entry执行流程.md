进行VM\-entry操作时处理器会执行严格的检查, 可以分为以下3个阶段.

第1阶段: 对VMLAUNCH或VMRESUME指令的执行进行基本检查

第2阶段: 对当前VMCS内的VM\-execution, VM\-exit以及VM\-entry控制区域和host\-state区域进行检查.

第3阶段: 对当前VMCS内的guest\-state区域进行检查, 并加载MSR.

所有检查通过后, 处理器从**guest\-state区域**里加载**处理器状态和执行环境信息**. 如果设置需要加载**MSR**, 则接着从**VM\-entry MSR\-load MSR列表区域里加载MSR**.

**VM\-entry操作**附加动作会**清由执行MONITOR指令而产生的地址监控(！！！**), 这个措施可**防止guest软件检测到自己处于虚拟机(！！！**)内.

在**成功完成VM\-entry**后, 如果**注入了一个向量事件**, 则通过**guest\-IDT**立即**deliver这个向量事件执行**. 如果存在pending debug exception事件, 则在**注入事件完成后deliver一个\#DB异常执行**.

注意: 在整个VM\-entry操作流程中, 如果VM\-entry失败可能产生下面三种结果:

(1) VMLAUNCH和VMRESUME指令产生异常, 从而执行相应的异常服务例程.

(2) VMLAUCH和VMRESUME指令产生VMfailValid或VMfailValid类型失败, 处理器接着执行下一条指令.

(3) VMLAUNCH和VMRESUME指令的执行由于检查guest\-state区域不通过, 或在加载MSR阶段失败而产生VM\-exit, 从而转入host\-RIP的入口点执行.

在前面所述的第1阶段检查里, 可能会产生第1种或第2中结果. 在第2阶段检查, 可能会产生第2种结果. 在第3阶段检查可能会产生第3种结果.