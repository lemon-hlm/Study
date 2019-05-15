
1. 将当前系统打包成tar文件

```
tar -cvpf /home/system.tar --directory=/ --exclude=proc --exclude=sys --exclude=dev --exclude=run /
```

/proc、/sys、/run、/dev这几个目录都是系统启动时自动生成的！依赖与系统内核！

2. 导入镜像

