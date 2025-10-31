# 1. apt

apt（Advanced Packaging Tool）是在 Ubuntu 和 Debian 的软件包管理器。

apt 是 apt-get 的更新替代品，更加简单直观和友好，推荐使用 apt，在一些旧版本系统则要使用 apt-get。

apt 命令执行需要 root 权限。

更新软件包列表

```bash
apt update
```

搜索软件包

```bash
apt search <关键词>
```

查看软件包详情

```bash
apt show <包名>
```

列出软件包

```bash
# 所有软件包
apt list

# 已安装
apt list --installed

# 可升级
apt list --upgradable
```

安装软件包

```bash
apt install <包名>

# 安装特定版本
apt install <包名>=<版本号>
```

升级已安装的软件包

```bash
apt upgrade

# 仅升级修复安全漏洞的包
apt --only-upgrade install <包名>
```

卸载软件包

```bash
# 保留配置文件
apt remove <包名>

# 删除配置文件
apt purge <包名>
```

清理无用文件

```bash
# 删除已下载的旧版本软件包
apt autoclean

# 删除所有已下载软件包
apt clean

# 移除无用依赖包
apt autoremove
```

更换软件源为国内镜像，提高下载速度

```bash
# 备份当前的源
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cp -r /etc/apt/sources.list.d /etc/apt/sources.list.d.bak

# 打开源列表文件
vi /etc/apt/sources.list

# 不同系统版本对应的源配置不同，需要在网上找到对应版本的替换进去
```

# 2. yum

yum（Yellowdog Updater, Modified）是 RHEL、CentOS 和 Fedora 使用的软件包管理工具。

yum 命令执行需要 root 权限。

列出软件包

```bash
# 已安装
yum list installed

# 可升级
yum list updates

# 所有可用的
yum list available
```

搜索软件包

```bash
# 关键词匹配
yum search <关键词>

# 查看软件包详情
yum info <包名>

# 查看依赖关系
yum deplist <包名>
```

升级软件包

```bash
# 更新升级所有包
yum update

# 升级包
yum update <包名>

# 仅升级安全补丁
yum update --security

# 检查可升级的包
yum check-update
```

安装软件包

```bash
yum install <包名>

# 安装本地rpm文件，并自动解决依赖
yum localinstall <rpm文件>
```

卸载软件包

```bash
yum remove <包名>
```

清理缓存

```bash
# 删除下载的软件包缓存
yum clean all

# 删除旧缓存
yum clean packages
```

列出历史记录

```bash
yum history
```

# 3. rpm

rpm（Red Hat Package Manager）是 RHEL、CentOS 和 Fedora 使用的 rpm 软件包管理工具，它无法自动解决依赖，只能操作本地 rpm 文件。

rpm 命令执行需要 root 权限。

参数：

* -v：显示详细信息
* -h：显示进度条

查看已安装的包

```bash
rpm -q <包名>

# 查询文件属于哪个rpm包
rpm -qf <文件>
```

安装软件包

```bash
rpm -ivh <包名>.rpm
```

升级软件包

```bash
rpm -Uvh <包名>.rpm
```

卸载软件包

```bash
rpm -e <包名>
```

