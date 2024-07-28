## 基本配置

### 配置使用xv6的 .gdbinit

- 问题描述
很多情况下，如果任由不设置，会出现每次都需要target remote，并且gdb调试看不到正在调试的代码
```bash
make qemu-gdb
warning: File "/home/darrenzhou/Code/xv6-public/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
```

- 解决方案
/home/daozhou下新建.gdbinit，添加
```bash
set auto-load safe-path /
```

- 跟踪变量变化：display /xw $eip