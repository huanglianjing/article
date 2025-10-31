# 1. 简介

Hugo 是一个使用 Go 编写的静态网站生成器，很适合用来部署个人博客网站。

官网：https://gohugo.io/

# 2. 部署

## 2.1 在mac部署

安装 Hugo：

```bash
brew install hugo
```

创建站点：

```bash
mkdir hugo
cd hugo
hugo new site huanglianjing.com
cd huanglianjing.com
```

打开 Hugo 的主题列表挑选主题：https://themes.gohugo.io/

下载并设置主题，将会将主题文件下载到 themes 文件夹内：

```bash
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
git submodule add https://github.com/CaiJimmy/hugo-theme-stack.git themes/stack
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/papermod
git submodule add https://github.com/Vimux/Mainroad.git themes/mainroad

echo "theme = 'papermod'" >> hugo.toml
```

添加文章，将会在 content/posts 下创建文件，然后在创建的 markdown 文件的开头文本后面追加博文内容：

```bash
hugo new posts/xxx.md

# 将已有内容追加到markdown文件中
cat xxx.md >> content/posts/xxx.md
```

创建的文章默认是草稿，取消草稿只需在文章开头配置信息修改 draft 为 false。

网站配置：

```bash
vi hugo.toml
```

构建静态网站，并启动网站，网站启动后按 Ctrl + C 将会停止运行 Hugo：

```bash
# 不包含草稿
hugo server

# 包含草稿
hugo server -D

# 后台运行
nohup hugo server & >> nohup.log 2>&1
```

Hugo 启动默认端口号为1313，浏览器访问网站：

```
http://localhost:1313/
```

在 GitHub 上创建对应仓库，然后把本地仓库同步到 GitHub 上去：

```bash
git remote add origin https://github.com/huanglianjing/huanglianjing.com.git
git push -u origin master
```

## 2.2 在云主机部署

在云主机中部署可以将项目上传至 Github，然后通过云主机拉取下来，进行编译。也可以在本机编译好，将整个项目或者 public 目录传到云主机。两种方式二选一。

### 2.2.1 通过Github拉取并编译

安装 Hugo：

```bash
# 打开 https://github.com/gohugoio/hugo/releases 选择最新版本的对应平台版本

wget https://github.com/gohugoio/hugo/releases/download/v0.115.3/hugo_0.115.3_Linux-64bit.tar.gz
tar zxf hugo_0.115.3_Linux-64bit.tar.gz
mv hugo /usr/bin/
```

从 GitHub 上拉取仓库。

```bash
git clone https://github.com/huanglianjing/huanglianjing.com.git
```

在网站仓库对应的文件夹，使用 hugo 命令构建网站，网站将会保存在 public 文件夹中：

```bash
hugo
```

### 2.2.2 本地编译后上传

对于云主机无法拉取 Github 代码的情况，可以从本地压缩整个文件夹，通过rz上传上去再解压。

```bash
# 本机执行
hugo
cd ..
rm -f huanglianjing.com.tgz
tar zcf huanglianjing.com.tgz huanglianjing.com/

# 云主机执行
rm -rf huanglianjing.com/
rm -f huanglianjing.com.tgz
rz
tar zxf huanglianjing.com.tgz 2>> /dev/null
```

### 2.2.3 配置Nginx

在云主机安装 nginx，启动 nginx 并调整配置：

```bash
service nginx start

vi /etc/nginx/nginx.conf
# user: 改为 root
# http - server - root: 改为 public 文件夹对应路径

nginx -t
nginx -s reload
```

然后在浏览器打开网址 http://huanglianjing.com/，成功看到内容！！！

### 2.2.4 部署https

以上只是开启了http的网站部署，如果申请了SSL证书，可以进行https部署。

首先进入腾讯云控制台，对于腾讯云服务器，进入安全组，对于腾讯云轻量应用服务器，进入防火墙，配置打开https(443)端口。

从控制台下载SSL证书的压缩包，如果是不分服务器类型的，解压后进入nginx文件夹，如果分服务器类型，则选择nginx的下载，下载下来解压后，在里面找到crt格式和key格式的文件，我的文件名如下：

```
huanglianjing.com_bundle.crt
huanglianjing.com.key
```

将这两个文件上传到服务器的 /etc/nginx 中。

编辑 /etc/nginx/nginx.conf，如下是我的配置，需要改动的地方已用序号标记出来：

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

# 1. 用户改成root，否则可能没有权限
user root;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        # 2. 配置域名
        server_name  huanglianjing.com;
        # 3. 配置public文件夹的绝对路径
        root         /root/git/huanglianjing.com/public;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            root /root/git/huanglianjing.com/public;
            index index.html index.htm;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

    # 4. 取消下面https的注释
    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        # 5. 配置域名
        server_name  huanglianjing.com;
        # 6. 配置public文件夹的绝对路径
        root         /root/git/huanglianjing.com/public;

        # 6. 配置ssl证书路径
        ssl_certificate "/etc/nginx/huanglianjing.com_bundle.crt";
        ssl_certificate_key "/etc/nginx/huanglianjing.com.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

}

```

保存后重启Nginx：

```bash
nginx -t
nginx -s reload
```

然后在浏览器打开网址 https://huanglianjing.com/，成功打开网页。

# 3. 命令

查看 Hugo 版本：

```bash
hugo version
```

查看命令帮助：

```bash
hugo help
hugo server --help
```

编译站点：

```bash
hugo
```

文档状态被定义在每个 markdown 文章的开头：

* title：标题
* draft：草稿，默认为草稿
* date：文档日期
* publishDate：发布日期
* expiryDate：过期日期

运行 Hugo：

```bash
# 构建网站
hugo

# 构建网站并运行
hugo server
```

运行 Hugo 选择的模式：

```bash
# --buildDrafts or -D
# --buildExpired or -E
# --buildFuture or -F
# --navigateToChanged 编辑内容时自动重定向网页
```

构建站点后，将会将站点发布到 public 文件夹。

# 4. 文件

## 4.1 目录结构

在站点的文件夹中，文件结构如下：

```
sitename/
├── archetypes/    // 内容模板目录
│   └── default.md // hugo new 创建文档的默认开头内容模版
├── assets/        // 记录需要被处理的文件
├── content/       // 内容目录，存放网站文章的 Markdown 源文件
├── data/          // 数据目录，存储数据结构，文件格式可以是json/toml/yaml，用 .Site.Data.xxxx 来获取数据
├── layouts/       // 模板目录，以html文件存储模板，指定如何将源文件转为静态网页
├── public/        // 编译生成静态网站的所有文件
├── static/        // 静态文件目录，存放如图片、CSS、JavaScript等文件
├── themes/        // 存放主题文件
└── hugo.toml      // 默认配置文件
```

public 文件夹结构如下：

```
public/
├── categories/
│   ├── index.html
│   └── index.xml  <-- RSS feed for this section
├── post/
│   ├── my-first-post/
│   │   └── index.html
│   ├── index.html
│   └── index.xml  <-- RSS feed for this section
├── tags/
│   ├── index.html
│   └── index.xml  <-- RSS feed for this section
├── index.html
├── index.xml      <-- RSS feed for the site
└── sitemap.xml
```

## 4.2 配置文件

Hugo 默认的配置文件是文件根目录中的 hugo.toml，自 Hugo v0.110.0 开始默认配置文件从 config.toml 改为了 hugo.toml。

Hugo 支持的配置文件格式包括 hugo.toml hugo.yaml hugo.json，可以指定配置文件构建网站：

```bash
hugo --config a.toml
```

配置文件参数如下，部分参数可以在 hugo server 命令后面加上，以在运行时设定参数。

toml格式配置文件：

```toml
# 网站标题
title = "website title"

# 域名地址
baseURL = "http://example.com/"

# 主题名称
theme = "papermod"

# 网站的语言代码
languageCode = "en-us"

# 是否将相对URL变为绝对URL
canonifyURLs = false

# 生成静态站点的目录
publishDir = "public"

# 是否生成robots.txt文件
enableRobotsTXT = false

# 是否启用自动检测内容中的中文/日语/韩语，让.Summary和.WordCount对于CJK语言正确运行
hasCJKLanguage = false

# 摘要长度
summaryLength = 70

# 默认分页数
paginate = 10

# 启用.html后缀地址，默认URL为/filename/，启用为/filename.html
uglyurls = false

# 自定义参数，通过.Site.Params.xxxx获取参数
[params]
  postDir = "posts"
  layoutReverse = false
  copyright = "cprcpr"
  description = "我的网站"

# 菜单参数，通过.Site.Menus.main获取参数
# Name为菜单名称、Weight为菜单排序参数、URL为菜单名称
[Menus]
  main = [
      {Name = "Categories", Weight = 1, URL = "/categories/"},
      {Name = "Tags", Weight = 2, URL = "/tags/"},
      {Name = "Links", Weight = 3, URL = "/links/"},
      {Name = "About", Weight = 4, URL = "/about/"},
      {Name = "Feedback", Weight = 5, URL = "/feedback/"}
  ]

# 博客链接的路径格式
[permalinks]
  post = "/:year/:month/:title/"
  page = "/:slug"

# 顶部栏
[[menu.navbar]]
  name = "首页"
  url = "http://localhost:1313"

# 侧边栏，可以写多个
[[menu.sidebar]]
  name = "新浪"
  url = "https://www.sina.com"
[[menu.sidebar]]
  name = "Github"
  url = "https://github.com"

# 属性设置
[params]
  # Site author
  author = "作者名"

  # homepage 页描述信息
  description = "我的博客站点"

  # Show header (default: true)
  #header_visible = true

  # Format dates with Go's time formatting
  DateFormat = "2006-01-02"
```

yaml格式配置文件：

```yaml
# 网站标题
title: "website title"

# 域名地址
baseURL: "http://example.com/"

# 主题名称
theme: "papermod"

# 网站的语言代码
languageCode: "en-us"

# 首页配置，toml中的[[languages.en.params]]在yaml中表示为树形结构
languages:
  en:
    params:
      languageName: "English"
      weight: 1
      profileMode:
        enabled: true
        title: "huanglianjing's blog"
        subtitle:
        imageUrl: "img/DO_COOL_THINGS_THAT_MATTER_BLUE.png"
        imageTitle:
        imageWidth: 640
        imageHeight: 360
```

## 4.3 源文件

存放在 content 目录下的 Markdown 源文件，格式如下：

```
---
文章属性内容
---
Markdown 正文
```

前面部分存放这篇文章的属性，后面是文章的正文 Markdown 内容。创建文章时默认的文章属性定义在 archetypes/default.md 中，然后可以手动修改内容。

常用文章属性如下：

```
---
title: "文章标题"        # 文章标题
author: "作者"          # 文章作者
description: "描述信息" # 文章描述信息
date: 2015-09-28       # 文章编写日期
lastmod: 2015-04-06    # 文章修改日期
tags: [                # 文章所属标签
    "文章标签1",
    "文章标签2"
]
categories: [ # 文章所属标签
    "文章分类1",
    "文章分类2",
]
keywords = [ # 文章关键词
    "Hugo",
    "static",
    "generator",
]
next: /tutorials/github-pages-blog     # 下一篇博客地址
prev: /tutorials/automated-deployments # 上一篇博客地址
---
```

# 5. 网站改造

博客设置参考以下博客：https://www.sulvblog.cn

源码：[xyming108/sulv-hugo-papermod](https://github.com/xyming108/sulv-hugo-papermod)

# 6. 参考

* [Hugo](https://gohugo.io/)
* [HUGO 目录详解，创建自己的网站系统 · 回忆中的明天](https://ichochy.com/posts/20200810.html)
* [Hugo博客目录放在侧边 | PaperMod主题 | Sulv's Blog](https://www.sulvblog.cn/posts/blog/hugo_toc_side/)

