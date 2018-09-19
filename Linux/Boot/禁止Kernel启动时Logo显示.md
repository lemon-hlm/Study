禁止Kernel启动时Logo显示

```
make ARCH=x86_64 menuconfig
    -> Device Drivers
        -> Graphics support
            [*] Bootup logo
```

将Bootup Logo特性关闭设置为N，默认是Y。