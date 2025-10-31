# 1. 简介

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/container/docker_logo.png)

Docker 由 Docker 公司开发，是一个能把应用程序自动部署到容器的开源引擎。

官网：https://www.docker.com/

大多数 Docker 容器启动非常快速，同时一台宿主机也可以运行很多容器，使用户可以充分利用系统资源。

Docker 让开发人员和运维人员的职责分离，开发人员只需要关心容器中运行的程序，运维人员只需要关心如何管理容器。

Docker 还鼓励面向服务的架构和微服务架构，推荐单个容器只运行一个应用程序或进程，组成分布式的应用程序模型，使扩展和调试应用程序变得简单。

Docker 包含三个基本概念：

* 镜像（Image）：镜像相是一个 root 文件系统，用户基于镜像来运行容器。
* 容器（Container）：容器基于镜像启动，容器中可以运行一个或多个进程，是镜像运行的实体。
* 仓库（Registry/Repository）：Registry 是镜像仓库平台，保存用户构建的镜像，分为公有和私有，Docker 公司的公共 Registry 叫做 Docker Hub，用户可以下载和上传镜像，也可以架设自己的私有 Registry。而 Repository 指的是 Registry 中某个用户下的某个仓库，和 Github 中的仓库类似。

Docker Hub 是官方的 Registry，地址为：https://hub.docker.com/。也可以在本地运行自己的 Registry，对应的镜像为 registry。

## 1.1 架构

Docker 是客户端/服务器（C/S）架构的程序，客户端只需向服务器或守护进程发出请求，服务器或守护进程完成所有工作并返回结果。Docker 提供了一个命令行工具 docker 来与守护进程交互，客户端可以访问本机的守护进程，也可以连接到另一台宿主机上的守护进程。

Docker 结构如下图所示，客户端连接到守护进程，守护进程会访问在本地的镜像以及仓库中的镜像，并且创建和管理容器。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/container/docker_architecture.jpg)

## 1.2 镜像

Docker 镜像是由文件系统叠加而成的。镜像的最底端是一个引导文件系统 bootfs，往上一层是容器中的基础镜像，是某种操作系统如 Ubuntu 或 Debian 等，而上面还会有 0 到多层的镜像，每一层表示基于镜像的修改再次提交的改动叠加，最上面一层则是容器，是从镜像启动创建容器后的空间。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/container/docker_filesystem.jpg)

Docker 使用联合加载（union mount）技术，对于从内核到基础镜像到多层镜像的文件系统叠加到一起，最终封装为包含所有底层的文件和目录的镜像文件。除了容器这一层可以读写，下面的其他层都是只读的。

这样一层层的镜像堆叠，位于下面的镜像称为父镜像（parent image），依此类推知道镜像栈最底部的基础镜像（base image）。当从一个镜像启动容器时，Docker 会在该镜像的最顶层加载一个读写文件系统，执行我们指定的程序或命令。

**Dockerfile**

Dockerfile 是一个定义文件，它使用基于 DSL（Domain Specific Language）语法的指令，用来构建一个 Docker 镜像。

首先我们需要创建一个空目录，进入该目录，创建 Dockerfile 并在里面编辑。这个目录称为构建环境（build environment），这个环境称为上下文（context），Docker 将会在构建镜像时将上下文中的文件和目录上传到 Docker 守护进程，守护进程就能直接访问到用户想存储到镜像的任何代码、文件、数据。

```bash
mkdir my_image
cd my_image
touch Dockerfile
```

Dockerfile 指令大体上的流程是：从基础镜像运行一个容器，执行一条指令，对容器作出修改，然后提交一个新的镜像层，再基于提交的镜像运行一个新的容器，执行下一条指令，直到所有指令执行完毕。

如果 Dockerfile 某条指令由于某些原因失败了，用户将得到一个可以使用的镜像，这对调试很有帮助。当构建过程失败时，可以将生成的镜像用 docker run 来构建容器，然后再在运行的容器内执行失败的命令，从而发现问题，然后就可以修改 Dockerfile 并再次尝试构建。

Dockerfile 的每条指令都是大写，注释以 # 开头。

下面是一个 Dockerfile 的例子。

```dockerfile
# nginx server
FROM ubuntu:12.04
MAINTAINER moondo "a.mail.com"
RUN apt-get update && apt-get install -y nginx
RUN echo 'this is a new container' > /usr/share/nginx/html/index.html
EXPOSE 80
```

每个 Dockerfile 的第一条指令必须是 FROM 指令，它指定一个已存在的镜像作为基础镜像（base image）。

MAINTAINER 指令标识该镜像的作者和邮件地址。

RUN 指令在当前镜像运行指定命令，并创建一个新的镜像层。指令默认使用命令包装器 /bin/sh -c，使用 exec 格式的 RUN 指令可以避免不支持 shell 的平台或不希望使用 shell 运行。

```dockerfile
# 使用 /bin/sh -c 运行
RUN apt-get install -y nginx

# exec 格式，不使用默认命令包装器，Docker 推荐形式
RUN ["apt-get","install","-y","nginx"]
```

EXPOSE 指令告诉 Docker 该容器内的应用程序将使用的容器内端口，创建容器时并不会因此自动打开该端口，需要在创建容器时指定。

Dockerfile 常用指令如下：

```dockerfile
# FROM 指定基础镜像
FROM ubuntu:12.04

# MAINTAINER 标识该镜像的作者和邮件地址
MAINTAINER moondo "a.mail.com"

# ENV 设置环境变量
ENV PATH /root

# RUN 运行命令，并创建一个新的镜像层
RUN apt-get update && apt-get install -y nginx

# CMD 指定容器启动时运行的命令
# docker run 传入的命令可以覆盖 CMD 指令，不指定则会执行 CMD 指定的命令
# 一个 Dockerfile 只能指定一条 CMD 指令，如果有多条，只有最后一条会被使用
CMD /bin/bash -l

# ENTRYPOINT 指定容器启动时运行的命令，相比 CMD 不会被 docker run 的命令覆盖
# docker run 的 --entrypoint 可以覆盖该指令
ENTRYPOINT ["/usr/sbin/nginx"]

# EXPOSE 声明应用程序将使用的容器内端口
EXPOSE 80

# WORKDIR 在容器内部设置工作目录，CMD 和 ENTRYPOINT 将在该目录执行
WORKDIR /opt/myapp

# USER 指定运行时的用户，不指定则默认为 root
USER nginx

# VOLUME 向容器添加卷
# 一个卷可以存在于一个或多个容器的特定目录，在容器间共享，对卷的修改不影响镜像，卷会一直存在直到没有任何容器使用它
# 可以将数据、代码等内容添加到镜像，而不用提交到镜像，从而在多个容器间共享内容
VOLUME ["/opt/project"]

# ADD 将构建环境的文件和目录复制到镜像中
# 只能添加构建目录中的文件或目录，文件源也可以是 URL 格式
Add a.lic /opt/myapp/a.lic

# COPY 将构建环境的文件和目录复制到镜像中
# 只复制文件，不能使用 URL，也不会提取和解压文件
COPY conf.d/ /etc/nginx/

# STOPSIGNAL 设置容器停止时发送的系统信号，可以用信号的数字或名称
STOPSIGNAL SIGKILL

# ARG 定义可以在构建镜像时传递进来的变量
# 如在构建时使用 --build-arg env=prod，可以预先赋值作为默认值
ARG env
ARG env=prod

# ONBUILD 添加触发器，当镜像被其它镜像作为基础镜像时，触发执行指令
ONBUILD ADD . /app/src
```

CMD 指令会被 docker run 的指令覆盖，ENTRYPOINT 指令不会被覆盖，可以通过以下方式来实现默认参数和可选参数。当 docker run 不指定命令，则以 ENTRYPOINT 作为命令而以 CMD 中的指令为参数，如果指定了则还会使用 ENTRYPOINT 的指令，但是会覆盖 CMD 的指令。

```dockerfile
ENTRYPOINT ["/usr/sbin/nginx"]
CMD ["-h"]
```

# 2. 命令

## 2.1 服务器

查看 Docker 程序状态，该命令会返回所有容器和镜像的数量、驱动程序、存储驱动、以及基本配置。

```bash
docker info
```

## 2.2 仓库

**登陆仓库**

登陆到 Docker 镜像仓库，如果没有指定镜像仓库地址，默认为 Docker Hub。

```bash
docker login

# 登陆私有仓库
docker login -u <admin> -p <passwd> <url>
```

**退出登录仓库**

退出登录的仓库。

```bash
docker logout
```

## 2.3 镜像

**列出镜像**

列出当前机器上可用的镜像。

```bash
docker images

# 查看指定镜像
docker images <image>[:<tag>]
```

**查找镜像**

从镜像仓库查找可用镜像。

```bash
docker search <image>
```

**拉取镜像**

从镜像仓库中拉取镜像到本地。

每个镜像会带有标签，表示不同的版本，默认拉取的是 latest 的镜像，除非指定镜像标签。

```bash
docker pull <image>[:<tag>]

# 拉取 latest 镜像
docker pull ubuntu
docker pull ubuntu:latest

# 拉取指定标签的镜像
docker pull ubuntu:12.04
```

**构建镜像**

从已有镜像创建的容器修改后创建新的镜像。

选项：

* -m""：指定镜像的提交信息
* -a""：指定镜像的作者信息

```bash
docker commit <container> <image>[:<tag>]
```

使用 Dockerfile 构建镜像，比较推荐这一种方式。

选项：

* -t：指定仓库、名称和标签
* --build-arg env=prod：构建时传入变量

```bash
docker build -t <repository>/<image>[:<tag>] <dir>

# 从当前目录构建镜像
docker build -t huanglianjing/my_image:v1 .

# 从 Git 仓库构建镜像
docker build -t huanglianjing/my_image:v1 git@github.com:huanglianjing/my_image
```

**镜像层级**

查看镜像的每一层的指令。

```bash
docker history <image>
```

**镜像删除**

删除本地镜像。

对于镜像仓库上的镜像，需要在浏览器登录然后在页面上操作删除。

```bash
docker rmi <image>
```

**镜像导出**

导出镜像为本地文件。

```bash
docker save -o <image>.tar <image>
```

**加载本地镜像**

将本地镜像加载到 docker 中。

```bash
docker load -i <image>.tar
```

**镜像推送**

将镜像推送到镜像仓库。

```bash
docker push <image>
```

## 2.4 容器

**查看容器列表**

查看已有的容器。

选项：

* -a：查看所有运行中和已暂停的容器，不带改选项只会返回运行中的容器
* -q：只返回容器 ID

```bash
# 查看所有容器的列表
docker ps -a

# 查看运行中的容器的列表
docker ps
```

**创建容器**

指定镜像名称和标签，首先检查本地是否存在该标签的镜像，如果没有就从镜像仓库中查找并下载镜像。然后使用该镜像创建一个新的容器，并且执行指定的命令。

每个容器有它的容器 ID 和名称，创建容器时会自动分配一个随机的名称，也可以在创建时指定名称。

选项：

* --name \<container_name\>：为容器指定名称
* -i：开启容器的 STDIN
* -t：为创建的容器分配一个伪 tty 终端，一遍提供交互式 shell
* -d：让容器在后台运行，启动守护式的容器
* -p \<port\>：控制容器公开给宿主机的网络端口，-p p2 将容器内的 p2 端口随机映射到宿主机 32768~61000 的某个端口，-p p1:p2 将容器内的 p2 端口映射到宿主机的 p1 端口
* -P：对外公开所有 EXPOSE 指令的端口，绑定到宿主机随机端口上
* --rm：容器退出后将其删除
* -v file:file：映射本地文件或目录与容器中的文件或目录
* --log-driver=""：控制容器的日志驱动，默认为 json-file，即开启 docker logs 命令，syslog 将禁用 docker logs 命令并将容器日志输出重定向到 Syslog，none 将禁用所有容器的日志
* --restart=：设置容器错误停止时的重启条件，always 表示无论容器退出代码是什么都会自动重启，on-failure:5 表示退出代码非 0 才重启，可以设置最多重试 5 次
* --entrypoint：覆盖 ENTRYPOINT 指令
* --env：设置环境变量
* -w \<dir\>：覆盖 WORKDIR 目录

```bash
docker run <image>[:<tag>] <cmd>

# 创建容器并进入交互式终端，exit 命令将会暂停交互式容器
docker run -i -t ubuntu /bin/bash

# 创建守护式容器，进行端口映射，后台执行
docker run -p 8080:80 -d nginx
# 宿主机访问端口： curl 127.0.0.1:8000
```

只创建容器而不启动，具体选项和用法与 docker run 一样。

```bash
docker create <image>[:<tag>] <cmd>
```

**启动容器**

启动已经停止运行的容器，可以通过容器 ID 或容器名称来指定。

```bash
docker start <container>
```

**重启容器**

重启一个运行中的容器。

```bash
docker restart <container>
```

**附着容器**

附着在一个运行中的容器，进入交互式终端。

```bash
docker attach <container>
```

**停止容器**

停止运行中的容器。

```bash
# 向容器发送 SIGTERM 信号，对于守护式容器立刻停止，对于交互式容器将等待
docker stop <container>

# 向容器发送 SIGKILL 信号，立即退出
docker kill <container>
```

**启动新进程**

在容器内部额外启动新进程。

```bash
# 运行交互命令
docker exec -i -t <container> <cmd>

# 运行后台任务
docker exec -d <container> <cmd>
```

**容器日志**

获取容器的日志。

选项：

* -f：监控日志追加内容
* --tail \<n\>：显示最后 n 行
* -t：为每条日志加上时间戳

```bash
docker logs <container>

# 监控日志追加内容
docker logs -f <container>

# 获取日志最后 10 行内容
docker logs --tail 10 <container>

# 监控日志追加内容，不需要读取整个日志文件
docker logs --tail 0 -f <container>
```

**容器进程**

查看容器内部运行的进程。

```bash
docker top <container>
```

**容器端口**

查看容器端口映射情况，容器内的哪个端口映射到了宿主机的哪个端口。

```bash
docker port <container> [<port>]
```

**容器统计信息**

查看一个或多个容器实时更新的的统计信息，包括 CPU、内存、硬盘 IO、网络 IO。

```bash
docker stats <containers>
```

**获取更多容器信息**

获取更多容器信息，包括名称、命令、网络配置等信息。

```bash
docker inspect <containers>
```

**删除容器**

将一个容器删除。

```bash
docker rm <containers>
```

**导出容器**

将本地容器导出为本地文件。

```bash
docker export -o <file> <container>
docker export <container> > <file>
```

**导入容器**

从容器导出的文件导入为镜像。

```bash
docker import <file>|<url> <image>
cat <file> | docker import - <image>
```

# 3. 工具

## 3.1 docker-compose

Docker Compose 是一个用于定义和运行多容器 Docker 应用程序的工具。它使用 YAML 文件来配置应用服务，并通过一个命令同时管理这些服务。

**更新镜像**

```bash
# 所有镜像
docker-compose pull

# 指定镜像
docker-compose pull code-server
```

**运行容器**

```bash
# 所有容器
docker-compose up -d

# 指定容器
docker-compose up -d code-server
```

**列出已有容器**

```bash
docker-compose ps
```

**启动已经存在的容器**

```bash
docker-compose start
```

**停止容器**

```bash
docker-compose stop
```

**重启服务**

```bash
docker-compose restart
```

# 4. 参考

* [《第一本Docker书》](https://book.douban.com/subject/26780404/)
* [Docker — 从入门到实践](https://yeasy.gitbook.io/docker_practice)

