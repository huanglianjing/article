# 1. 简介

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/prometheus_logo.png)

Prometheus 是一个开源的服务监控系统和时序数据库，提供了通用的数据模型和快捷的数据采集、存储和查询接口。Prometheus 将指标收集并存储为时间序列数据，即将指标信息、可选的 KV 标签值和记录时的时间戳一起存储。通过指标的收集和查询，我们可以更好地了解系统的运行状态，帮助排查定位异常问题的具体原因。

Prometheus 的主要特性有：

* 一个多维的数据模型，包含指标信息和 KV 标签的时序数据
* 使用 PromQL 来灵活地查询不同维度数据
* 使用单独服务器节点，不使用分布式存储
* 通过 HTTP 请求以拉的方式收集数据
* 通过中间网关支持推送时间序列
* 通过服务发现或静态配置来发现目标
* 支持多种图形和仪表板模式

Prometheus 适用于纯数字时间序列的记录，对多维数据收集和查询，系统注重可靠性，能帮助定位诊断问题，Prometheus 服务器独立不依赖其它网络存储或远程服务。

但需要注意，Prometheus 的统计不是 100% 准确的，收集的数据可能不够详细和完整，对这方面有要求的可以使用其它系统收集和分析精确数据，使用 Prometheus 来进行其余的监控。

Prometheus 对于监控数据的采集和计算，关注的是趋势和显著异常，因此记录并非百分百准确记录，计算结果存在误差，这是一种在高性能、低开销和可扩展性之间权衡的结果，误差来源有：

* 监控目标记录的数据保存在内存，重启会重置记录，未抓取的那段时间数据会丢失；
* 服务器以固定的时间间隔抓取数据，如 15秒或 1分钟一次，短暂波动如瞬时流量尖峰或错误可能无法被记录；
* 查询计算时，基于时间范围和外推法来估算速率或增量，会有精度偏差，且可能出现小数；

Prometheus 的使用场景：

* 云原生与容器化环境监控：发现并监控集群中的 Pod、Service、Node 等动态资源；
* 服务可用性和性能监控，如请求频率、成功率、接口时延等指标；
* 通过 Node Exporter 监控机器资源，如 CPU、内存、磁盘I/O、网络流量等；

不适合的场景：

* 要求完全精确的统计指标；
* 存储和查询长期历史数据；
* 日志记录和事件存储；

# 2. 架构

Prometheus 的架构如下图所示。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/prometheus_architecture.png)

Prometheus Server（服务器）作为核心组件，负责实现对监控数据的获取，存储以及查询。通过服务发现或配置发现目标，以拉的方式从中间推送网关（Pushgateway）或仪表器任务中拉取指标，将所有数据通过 TSDB 数据库保存在本地磁盘，通过 PromQL 在自带的 web UI 界面或如 Grafana 等数据可视化平台上聚合和查询数据，使用 Alertmanager 基于指标计算发送各种方式的告警。

Exporter 将监控数据采集的端点通过 HTTP 服务的形式暴露给 Prometheus Server，Prometheus Server 通过访问该 Exporter 提供的 Endpoint 端点，即可获取到需要采集的监控数据。Kubernetes、Etcd、Gokit 等服务直接内置了向 Prometheus 暴露监控数据的端点，或者我们可以使用 Prometheus 提供的各种目标的监控采集程序如 node_exporter、mysqld_exporter 等。

PushGateway 可以将无法通过拉取方式获得的内部网络的监控数据主动推送到 Gateway 当中，Prometheus Server 就可以采用拉取方式从 PushGateway 获取监控数据了。

AlertManager 支持基于 PromQL 的告警规则，集成了邮件、企业微信、Slack 等方式的通知。

## 2.1 TSDB

Prometheus 内置了一个专门为监控场景设计和优化的 TSDB 存储引擎。

核心特性有：

* 数据每 2 小时保存至磁盘的一个块（Block），每个块拥有自己的索引和元数据文件，后台进程通过XOR差分编码算法进一步压缩；
* 通过预写日志（WAL）机制，确保数据的持久性和崩溃恢复能力；
* 通过倒排索引实现数据快速查询；

但是内置的 TSDB 存储引擎设计为单节点本地存储，存储容量和持久性受单节点限制，不适合存储长期大量的监控数据。

## 2.2 数据模型

Prometheus 将所有数据存储为时间序列，即同一指标（metrics）和同一组标签（label）维度的基于时间戳流的值。通过对指标的查询，还可以生成临时的派生时间序列作为查询结果。Prometheus 会将所有采集到的样本数据以时间序列（time series）的方式保存在内存中的时序数据库 TSDB 中，并且定时保存到硬盘上。

指标（metrics）是随时间变化的数值测量。具体要测量的内容因应用程序而异，如对于 web 服务器是请求时间，而对数据库是活动连接和查询的数量。指标的名称可以包含大小写字母、数字、下划线和冒号，但是不能以数字开头，如 `http_requests_total`。

指标的命名有一些推荐建议：

* 指标名称前缀最好指代所属应用的方面，如 http_requests_total、api_request_seconds
* 应该包含基本单位，且是复数形式，如 seconds、requests、bytes 等
* Counter 类型的指标推荐使用 _total 作为后缀

标签（label）用于标记一个指标的特定维度，一个指标可以拥有多个标签，不同的标签值的组合都会创建新的时间序列。标签的名称可以包含大小写字母、数字和下划线，但是不能以数字开头，如 `method`。

```
<metric_name>{<label_name>=<label_value>, ...}
```

例如，一个指标具有如下三种不同组合的标签值，Prometheus 会分别为它们生成 3 个时间序列。

```
http_requests_total{method="GET", handler="/update"}
http_requests_total{method="GET", handler="/set"}
http_requests_total{method="POST", handler="/set"}
```

可以将时间序列理解为横轴为时间，竖轴为指标和标签唯一组合的数字矩阵，其中每一个点是一个样本（sample），每个样本由时间戳、指标和标签、样本值三部分组成。

```
  ^
  │   . . . . . . . . . . . . . . . . .   . .   metric{method="GET", handler="/update"}
  │     . . . . . . . . . . . . . . . . . . .   metric{method="GET", handler="/set"}
  │     . . . . . . . . . .   . . . . . . . .   metric{method="POST", handler="/set"}
  │     . . . . . . . . . . . . . . . .   . .  
  v
    <------------------ 时间 ---------------->
```

## 2.3 指标类型

Prometheus 支持四种指标类型，它们仅在客户端进行区分，服务器端实际上会把它们都展平为无类型的时间序列。

**Counter**

Counter（计数器）表示一个单调递增计数器，它只能增加，只会在重启时重置为零。它可以用于表示服务的请求数、已完成的任务数、错误数等只增不减的测量值。

**Gauge**

Gauge（仪表盘）表示可以任意上升或下降的单个数值。它用于表示内存使用情况、并发请求数等可能增加或减少的测量值。

**Histogram**

Histogram（直方图）对测量值进行采样并分配到自定义的几个桶中，以计算指标的平均值、加权平均数、分位数（P80、P90、P99 等）。它可以用于请求时间、响应结果大小等测量值，可以很方便地计算服务的平均请求时间、P90 请求时间等性能指标。

对于一个 Histogram 类型的指标 `<metric_name>` 来说，会在数据拉取时生成以下几个时间序列：

* `<metric_name>`_bucket{le="upper inclusive bound"}：根据指标定义时的设置，分为若干个计数器分桶。观测值根据分桶的分界值，不是在归属范围的某个桶累积计数，而是在最靠后的一或多个桶累积计数。例如有 `le="100"`、`le="200"`、`le="300"`、`le="+Inf"`几个桶，它们分别代表在 `(-Inf,100]`、`(-Inf,200]`、`(-Inf,300]`、`(-Inf,+Inf)` 这几个范围的计数，实际观测值 220，将会令后两个分桶计数加一
* `<metric_name>`_sum：所有观测值之和
* `<metric_name>`_count：已有的观测值数量，等同于 `m_bucket{le="+Inf"}`

一个例子，假设指标 request_latency_seconds 有 100 次观测值，每个观测值分布在 100 - 500 之间，则会在以下时间序列中记录有值：

```
request_latency_seconds_bucket{le="100"} 0.0
request_latency_seconds_bucket{le="200"} 31.0
request_latency_seconds_bucket{le="300"} 87.0
request_latency_seconds_bucket{le="+Inf"} 100.0
request_latency_seconds_count 100.0
request_latency_seconds_sum 32705.0
```

通过 histogram_quantile 函数来计算分位数时，其实是通过现有的各个分桶的指标值进行估算，来得出如 P90 的百分位的值，分桶的数量越多、分桶设置得越合理，就能更准确地预估百分位数。

源码中的计算公式如下：

```go
return bucketStart + (bucketEnd-bucketStart)*float64(rank/count)
```

**Summary**

Summary（摘要）类似于 Histogram，也会对观察值结果分桶拆样和统计观察值数量、观察值总和，但它还可以根据配置计算滑动时间窗口上的分位数。它可以用于请求时间、响应结果大小等测量值。

Summary 需要预定义要计算的分位数，在客户端计算好分位数，且只能查询预订好的分位数。只有用于监控单个实例且非常精确的分位数，才建议使用 Summary，否则应该避免使用。

对于一个 Summary 类型的指标 m 来说，会在数据拉取时生成以下几个时间序列：

* m{quantile="<φ>"}：观测值的 φ 分位数（0 ≤ φ ≤ 1）
* m_sum：所有观测值之和
* m_count：已有的观测值数量

## 2.4 作业和实例

实例（instance）是一个可以拉取指标数据的端点，通常是一个进程。

作业（job）是一组具有相同职责的实例的集合，是一个程序在不同容器的多个部署的集合。

例如，一个服务器程序具有四个实例：

```
job: api-server
	instance 1: 1.2.3.4:5670
	instance 2: 1.2.3.4:5671
	instance 3: 5.6.7.8:5670
	instance 4: 5.6.7.8:5671
```

Prometheus 在拉取数据时，会自动地添加一些标签用于识别区分拉取的目标：

* job：所属作业的名称
* instance：所属实例的名称，格式为 `<host>:<port>`

# 3. 安装和使用

## 3.1 服务器

Prometheus 服务器主要负责数据的收集、存储和查询支持，它本身也会上报一些相关的监控数据。

Prometheus 服务器默认端口为 9090。

可以在官方网站中下载并安装：https://prometheus.io/download/

对于 macOS 可以通过 brew 或者 docker 来安装软件。

```bash
# brew
brew install prometheus

# docker
docker pull prom/prometheus
```

Prometheus 使用基于 YAML 的配置文件，配置文件 prometheus.yml 示例：

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  # 采集服务器监控数据
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

配置中的 global 是全局配置，其中 scrape_interval 表示获取指标的频率，evaluation_interval 表示评估规则的频率，使用规则可以创建新的时间序列和生成告警。rule_files 是服务器需要加载的其它规则文件。scrape_configs 表示服务器监控的资源，在这里创建了一个 job 来监控本地机器的 9090 端口，指标通过 /metrics 路径访问。

对于其它的配置项可以参考：https://prometheus.io/docs/prometheus/latest/configuration/configuration/

启动服务器，会默认加载当前路径下的配置文件 prometheus.yml。

```bash
# 前台启动，默认加载当前路径下的配置文件 prometheus.yml
prometheus

# 前台启动，指定配置文件
prometheus --config.file=prometheus.yml

# brew，配置文件在 /usr/local/etc/prometheus.yml
brew services start prometheus

# docker
# 创建配置和数据目录
cd ~
mkdir prometheus
cd prometheus
mkdir data
touch prometheus.yml # 写配置文件
docker run -d -p 9090:9090 -v ~/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml -v ~/prometheus/data:/prometheus --network=host --name prometheus prom/prometheus
```

Prometheus Server 启动参数详情可参考：https://prometheus.io/docs/prometheus/latest/command-line/prometheus/

在浏览器打开 http://localhost:9090 可以使用指标的表达式，通过表格或图标的方式可视化地查询指标，或者可以访问 http://localhost:9090/metrics 获取所有指标的最新值。

## 3.2 Node Exporter

Prometheus 服务器主要负责数据的收集、存储和查询支持，我们需要用到 Exporter 来监控一些机器的监测数据如 CPU 使用率等。

node-exporter 服务默认端口为 9100。

可以在官网下载安装 node_exporter：https://prometheus.io/download/

对于 macOS 可以通过 brew 或者 docker 来安装软件。

```bash
# brew
brew install node_exporter

# docker
docker pull prom/node-exporter
```

由于我们在本机上安装 node_exporter 来监控机器数据，所以需要在 prometheus.yml 添加配置：

```yaml
scrape_configs:
  # 采集服务器监控数据
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  # 采集 node exporter 监控数据
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
```

启动服务：

```bash
# 前台启动
node_exporter

# brew 启动服务
brew services start node_exporter

# docker
docker run -d -p 9100:9100 --name node-exporter prom/node-exporter
```

然后重新启动 Prometheus 服务器。

访问 http://localhost:9100/metrics 获取 node exporter 上报的所有指标的最新值。想要通过可视化方式查看还是需要通过 Prometheus 服务器使用的端口在浏览器打开 http://localhost:9090。

在查询页面查询 up，可以看到 job 为 prometheus 和 node 各有一个，值都是 1，表示成功监听到了它们。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/prometheus_node_exporter_up.png)

node exporter 主要监控的指标有：

* node_boot_time_seconds：系统启动时间
* node_time_seconds：当前系统时间
* node_cpu_*：系统 CPU 使用量
* node_disk_*：磁盘 IO
* node_filesystem_*：文件系统用量
* node_load*：系统负载
* node_memory_*：内存使用量
* node_network_*：网络带宽
* go_*：node exporter 中 Go 相关指标

## 3.3 Grafana

虽然 Prometheus 提供了一个基础的 UI 界面网页，可以用于查询验证数据，但是有必要使用更成熟的数据可视化平台作为监控面板，这里推荐使用 Grafana。

Grafana 服务默认端口为 3000。

安装并运行 Grafana：

```bash
brew install grafana
brew services start grafana
```

浏览器访问 http://localhost:3000/ 进入 Grafana 界面，初始默认账号密码为 admin/admin。

在页面中选择添加 Data sources（数据源），选择 prometheus。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/grafana_add_data_source.png)

然后添加 Dashboards（仪表板）。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/grafana_add_dashboard.png)

然后就可以在页面上测试 PromQL 以及展示效果了，并且可以添加多个 Row（行）表示分类，添加不同的 Visualization（可视化）表示不同的监控。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/grafana_test_promql.png)

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/database/grafana_my_dashboard.png)

## 3.4 AlertManager

首先定义告警规则文件，test.rules 例子如下：

```
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
      description: description info
```

将一组相关的规则定义在一个 group 下，每个 group 中可以定义多个告警规则（rule），包含 alert（告警规则名称）、expr（告警触发表达式）、for（触发条件持续一定时间才告警，可选）、labels（自定义标签）、annotations（附加信息如告警描述文字等）。

然后在 Prometheus 配置文件 prometheus.yml 的 rule_files 中写入告警文件的路径，Prometheus 服务器会扫描路径加载告警规则文件。

```
rule_files:
  - /etc/prometheus/rules/*.rules
```

然后下载并安装 AlertManager：https://prometheus.io/download/

AlertManager 默认端口 9093，可以在浏览器访问 http://localhost:9093 查看运行状态。

具体部署方式参考：https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/install-alert-manager

# 4. PromQL

Prometheus 通过指标名称以及对应的一组标签唯一定义一条时间序列，指标名称反映了监控样本的基本标识，而标签则在这个基本特征上为采集到的数据提供了多种特征维度。用户可以基于这些特征维度过滤、聚合、统计，从而产生新的计算后的一条时间序列。

Prometheus 定义了它的查询语言 PromQL，通过 PromQL 用户可以非常方便地对监控样本数据进行统计分析。PromQL 支持常见的运算操作符，还提供了大量的内置函数可以实现对数据的高级处理。

## 4.1 查询时间序列

直接使用指标名称时，查询该指标下的所有时间序列。这时候花括号可以不写出来。

```
http_requests_total
http_requests_total{}
{__name__="http_requests_total"} # 还可以用 __name__ 来精确匹配或正则匹配若干个指标
```

将会返回这个指标所有标签对应的时间序列。

```
http_requests_total{method="GET", handler="/update"}
http_requests_total{method="GET", handler="/set"}
http_requests_total{method="POST", handler="/set"}
```

通过匹配标签来筛选过滤时间序列。

```
# 标签值等于
http_requests_total{method="GET"}

# 标签值不等于
http_requests_total{method!="GET"}

# 标签值匹配正则表达式
http_requests_total{method=~"GET|POST"}

# 标签值不匹配正则表达式
http_requests_total{method!~"GET|POST"}
```

以上查询的返回值只会包含时间序列的最新样本值，这种表达式称为瞬时向量表达式。

使用时间来获取过去一段时间的样本数据，需要在表达式后面加上已给时间，这称为区间向量表达式。支持时间格式有 s（秒）、m（分钟）、h（小时）、d（天）、w（周）、y（年），如 5m 代表过去 5 分钟，1d 代表过去 1 天。

```
# 最近 5 分钟内的样本数据
http_requests_total{}[5m]
```

之前是以当前时间作为标准，可以将这个时间往前位移。

```
# 20 分钟前的最新数据
http_request_total{} offset 20m

# 1 天前时刻的往前 3 小时内的数据
http_request_total{}[3h] offset 1d
```

## 4.2 操作符

指标间支持加减乘除（+-*/）、余数（%）、幂运算（^）的数学运算符。对于瞬时向量和标量的计算，会将瞬时向量的每个样本值与标量进行计算。对于两个瞬时向量的计算，会依次对双方标签完全一致的向量元素进行匹配和计算。

```
# 读写磁盘字节数之和
node_disk_writes_completed_total + node_disk_read_bytes_total

# 空闲内存的单位从字节转为 GB
node_memory_free_bytes / (1024 * 1024 * 1024)
```

此外还支持布尔运算有 ==、!=、>、<、>=、<=。将会对瞬时向量的各个样本数据进行计算，形成新的时间序列。

```
# 结果为 true 保留，结果为 false 丢弃
http_request_total > 100

# 结果为 true/false 转化为 1/0
http_request_total > true 100
```

支持集合运算 and（且）、or（或）、unless（排除）。

```
# 交集
metrics1 and metrics2

# 并集
metrics1 or metrics2

# metrics1 有而 metrics2 没有
metrics1 unless metrics2
```

## 4.3 聚合操作

聚合操作包含有：

* sum：求和
* min：最小值
* max：最大值
* avg：平均值
* stddev：标准差
* stdvar：标准方差
* count：计数
* count_values：对 value 进行计数
* topk：前 n 条时序
* bottomk：后 n 条时序
* quantile：分位数

聚合操作语法如下：

```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

只有 count_values、quantile、topk、bottomk 支持 parameter。

by 只保留列出的标签，without 移除列出的标签，然后按照保留的或者未移除的标签对数据进行聚合，不使用 by 或 without 会计算整个指标。

```
# 所有标签组合的值求和
sum(http_requests_total)

# 按 method 聚合求和
sum(http_requests_total) by (method)

# 值排序前 5 的时间序列
topk(5, http_requests_total)

# 大于 0.9 分位数的时间序列
quantile(0.9, http_requests_total)
```

## 4.4 内置函数

increase 函数用于 Counter 指标，对于时间序列，计算每个样本从样本时间的指定时间前到样本时间的增长量。

```
increase(node_memory_free_bytes[1m])

# 一段时间内的统计总数
sum(increase(node_memory_free_bytes[1m]))
```

rate 函数用于 Counter 指标，对于时间序列，计算每个样本从样本时间的指定时间前到样本时间的增长率。相当于 increase 函数的结果除以时间表示的秒数，如 1m 就除以 60。

```
rate(node_memory_free_bytes[1m])
```

increase 和 rate 函数计算样本的增长率有可能无法反应在时间窗口内样本数据的突发变化，如 CPU 使用量的突增被时间窗口掩盖无法反应出来。可以使用 irate 函数，它也用于计算区间向量的增长速率，通过区间向量中最后两个样本来计算增长速率的，有着更好的灵敏度。但 irate 更灵敏也有可能造成干扰，在长期趋势分析或告警规则中不建议使用。

```
irate(node_memory_free_bytes[1m])
```

predict_linear 函数用于 Gauge 指标，基于简单线性回归，预测时间序列在若干秒后的值，适用于告警阈值不固定的情况。

```
predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0
```

delta 函数用于 Gauge 指标，它计算时间序列的指定时间范围的第一个和最后一个样本的差值，和 increase 函数类似。

```
delta(go_memstats_alloc_bytes{job="prometheus"}[1h])
```

idelta 函数用于 Gauge 指标，它计算时间序列区间向量中最后两个样本来计算增长速率。

```
idelta(go_memstats_alloc_bytes{job="prometheus"}[1h])
```

histogram_quantile 函数用于 Histogram 指标，用于计算分位数。

```
# 计算中位数
histogram_quantile(0.5, http_request_duration_seconds_bucket)

# 计算 P90
histogram_quantile(0.9, http_request_duration_seconds_bucket)
```

更多内置函数可以参考：https://prometheus.io/docs/prometheus/latest/querying/functions/

# 5. 参考

* [Overview | Prometheus](https://prometheus.io/docs/introduction/overview/)
* [prometheus-book](https://yunlzheng.gitbook.io/prometheus-book)

