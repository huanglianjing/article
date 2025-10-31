# 1. 进程

**通过进程pid查看进程**

通过 top 或 ps 命令获取某个进程的 pid，如 12345。

```bash
# 程序所在目录
ls -l /proc/12345/

# 工作目录
ls -l /proc/12345/cwd

# 程序文件路径
ls -l /proc/12345/exe
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

