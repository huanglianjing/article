# 1. 进程

**查看进程proc**

通过 top 或 ps 命令获取某个进程的 pid，如 12345。

```bash
# 程序所在目录
ls -l /proc/12345/

# 工作目录
ls -l /proc/12345/cwd
pwdx 12345

# 程序文件路径
ls -l /proc/12345/exe

# 程序当前打开的文件描述符，包含读取的文件、数据库、日志、设备、管道、socket
ls -l /proc/12345/fd
```

# 2. 内存

**手动释放内存缓存**

执行前建议先执行 sync，减少脏页，尽可能释放更多内存缓存。

```bash
# 释放页缓存
echo 1 > /proc/sys/vm/drop_caches

# 释放目录、索引节点缓存
echo 2 > /proc/sys/vm/drop_caches

# 释放页、目录、索引节点缓存
echo 3 > /proc/sys/vm/drop_caches
```

参考文档：[kernel documents sysctl vm](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)

# 3. 系统

**查看系统负载**

分别查看系统整体的 CPU、内存、I/O 压力情况，也可以搭配 watch 命令持续监控：

```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

输出结果示例：

```
some avg10=1.00 avg60=1.00 avg300=1.00 total=2884247369
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

some 表示至少有一个任务因等待该资源而阻塞的时间百分比，full 表示所有非空闲任务都同时在等待该资源。

avg10、avg60、avg300 分别表示最近 10 秒、60 秒、300 秒的压力百分比，total 表示系统启动以来累计的阻塞总时间（单位为微秒）。

当 avg10 小于 0.05 表示状态健康，达到 0.1 时表示有明显任务等待，值越大表示负载越大。

查看具体服务的负载：

```bash
cat /sys/fs/cgroup/system.slice/<service name>/cpu.pressure
cat /sys/fs/cgroup/system.slice/<service name>/memory.pressure
cat /sys/fs/cgroup/system.slice/<service name>/io.pressure
```

