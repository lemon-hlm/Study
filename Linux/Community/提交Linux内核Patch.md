https://consen.github.io/2018/01/19/submit-linux-kernel-patch/

之前使用QEMU和GDB调试Linux内核（详见Linux/Debug/相关文章），最后使用内核提供的GDB扩展功能获取当前运行进程，发现内核已经不再使用thread\_info获取当前进程，而是使用Per-CPU变量。而且从内核4.9版本开始，thread\_info也不再位于内核栈底部，然而内核提供的辅助调试函数lx\_thread\_info()仍然通过内核栈底地址获取thread\_info，很明显这是个Bug，于是决定将其修复并提交一个内核Patch，提交后很快就得到内核维护人员的回应，将Patch提交到了内核主分支。

Linux内核Patch提交还是采用邮件列表方式，不过提供了自动化工具。