## PromQL 基础

### 理解时间序列

在通过 node_exporter 暴露的HTTP服务，Prometheus可以采集到当前主机所有监控指标的样本数据。例如：

```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.  # HELP <metrics_name> <doc_string>
# TYPE node_cpu_seconds_total counter                               # TYPE <metrics_name> <metrics_type>
node_cpu_seconds_total{cpu="0",mode="idle"} 175210.6
node_cpu_seconds_total{cpu="0",mode="iowait"} 20.15
```

第一个 # 行是监控指标的说明，第二个 # 行是该样本的类型，剩余非 # 开头的行为采集到的监控样本。

### 样本

Prometheus会将所有采集到的样本数据以时间序列（time-series）的方式保存在内存数据库中，并且定时保存到硬盘上。time-series是按照时间戳和值的序列顺序存放的，我们称之为向量(vector). 每条time-series通过指标名称(metrics name)和一组标签集(labelset)命名。在time-series中的每一个点称为一个样本（sample），样本由以下三部分组成：

- 指标(metric)：metric name和描述当前样本特征的labelsets;
- 时间戳(timestamp)：一个精确到毫秒的时间戳;
- 样本值(value)： 一个float64的浮点型数据表示当前样本的值。

## 指标(Metric)

Prometheus定义了4中不同的指标类型(metric type)：Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）、Summary（摘要）。在形式上，Metrics 由 metric name 和 label name 组成：

```
<metric name>{<label name>=<label value>, ...}
```

### Counter：只增不减的计数器

Counter类型的指标其工作方式和计数器一样，只增不减（除非系统发生重置）。一般在定义Counter类型指标的名称时推荐使用_total作为后缀。

通过rate()函数获取HTTP请求量的增长率：

```
rate(prometheus_http_requests_total[5m]) 
```

查询当前系统中，访问量前10的HTTP地址：

```
topk(10, prometheus_http_requests_total)
```

### Gauge：可增可减的计量器

Gauge类型的指标侧重于反应系统的当前状态。因此这类指标的样本数据可增可减。

查看系统的当前剩余内存单位为GB：

```
node_memory_MemFree_bytes{instance="test-120-82"} /1024/1024/1024
```

通过PromQL内置函数delta()可以获取样本在一段时间返回内的变化情况，计算CPU温度在两个小时内的差异：

```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

使用deriv()计算样本的线性回归模型，预测系统磁盘空间在4个小时之后的剩余情况：

```
predict_linear(node_filesystem_free_bytes{instance="test-120-82",mountpoint="/"}[1h], 4 * 3600)
```

### Histogram 柱状图

histogram 是反应数值的分布情况，histogram 柱状图反映了样本的区间分布梳理，经常用来表示请求持续时间、响应大小等信息。例如我们 1 分钟内有 1000 个 http 请求，我们想要知道大多数的请求耗时是多少。这时我们使用评价耗时可能不太准，但是我们使用 histogram 柱状图就可以看出这些请求大多数都是分布在哪个耗时区间。

```
# HELP prometheus_tsdb_tombstone_cleanup_seconds The time taken to recompact blocks to remove tombstones.
# TYPE prometheus_tsdb_tombstone_cleanup_seconds histogram
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.005"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.01"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.025"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.05"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.1"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.25"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="0.5"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="1"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="2.5"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="5"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="10"} 0
prometheus_tsdb_tombstone_cleanup_seconds_bucket{le="+Inf"} 0
prometheus_tsdb_tombstone_cleanup_seconds_sum 0
prometheus_tsdb_tombstone_cleanup_seconds_count 0
```

### Summary 汇总

柱状图同样表示样本的分布情况，与 Histogram 类似，其会有总数、数量表示。**但其多了一个是中位数的表示。** 常用来表示类似于：请求持续时间、响应大小的信息。

```
# HELP prometheus_target_interval_length_seconds Actual intervals between scrapes.
# TYPE prometheus_target_interval_length_seconds summary
prometheus_target_interval_length_seconds{interval="15s",quantile="0.01"} 14.998622374
prometheus_target_interval_length_seconds{interval="15s",quantile="0.05"} 14.998814221
prometheus_target_interval_length_seconds{interval="15s",quantile="0.5"} 15.000017571
prometheus_target_interval_length_seconds{interval="15s",quantile="0.9"} 15.000840271
prometheus_target_interval_length_seconds{interval="15s",quantile="0.99"} 15.001259885
prometheus_target_interval_length_seconds_sum{interval="15s"} 569460.364581299
prometheus_target_interval_length_seconds_count{interval="15s"} 37964
```

**Histogram 指标直接反应了在不同区间内样本的个数，区间通过标签len进行定义。而 summary 则是使用中位数反映样本的情况。**

## PromQL 基础语法

### 合法的PromQL表达式

所有的PromQL表达式都必须至少包含一个指标名称(例如http_request_total)，或者一个不会匹配到空字符串的标签过滤器(例如{code="200"})。同时，除了使用`<metric name>{label=value}`的形式以外，我们还可以使用内置的`__name__`标签来指定监控指标名称：

```
{__name__=~"prometheus_http_requests_total"} # 合法
{__name__=~"node_disk_read_bytes_total|node_disk_written_bytes_total"} # 合法
```

### 匹配

- 通过使用`label=value`可以选择那些标签满足表达式定义的时间序列；
- 反之使用`label!=value`则可以根据标签匹配排除时间序列；
- 使用`label=~regx`表示选择那些标签符合正则表达式定义的时间序列；
- 反之使用`label!~regx`进行排除；

查看prometheus_http_requests_total 中，状态码为200且handler标签不是 `/api/` 开头的记录：

```
prometheus_http_requests_total{code="200",handler!~"/api/.*"}
```

### 范围查询

上面直接通过类似 prometheus_http_requests_total 表达式查询时间序列时，同一个指标同一标签只会返回一条数据。这样的表达式我们称之为**瞬间向量表达式**，而返回的结果称之为**瞬间向量**。而如果我们想查询一段时间范围内的样本数据，那么我们就需要用到**区间向量表达式**，其查询出来的结果称之为**区间向量**。时间范围通过时间范围选择器 `[]` 进行定义。

查看最近5分钟内的所有样本数据：

```
prometheus_http_requests_total{}[5m]
```

查询 5 分钟前的瞬时样本数据：

```
prometheus_http_requests_total{} offset 5m
```

查询昨天一天的区间内的样本数据：

```
prometheus_http_requests_total{}[1d] offset 1d
```

### 聚合操作

一般情况下，我们通过 PromQL 查询到的数据都是很多的。PromQL 提供的聚合操作可以用来对这些时间序列进行处理，形成一条新的时间序列。

```
# 查询系统指定样本的监控指标数目
count(prometheus_http_requests_total)

# 查询系统所有http请求的总量
sum(prometheus_http_requests_total)

# 按照mode计算主机CPU的平均使用时间
avg(node_cpu_seconds_total{instance="test-120-82"}) by (mode)

# 按照主机查询各个主机的CPU使用率
sum(sum(irate(node_cpu_seconds_total{mode!='idle'}[5m]))  / sum(irate(node_cpu_seconds_total[5m]))) by (instance)
```

### 标量

在 PromQL 中，标量是一个浮点型的数字值，没有时序。当使用表达式`count(http_requests_total)`，返回的数据类型，依然是瞬时向量。用户可以通过内置函数scalar()将单个瞬时向量转换为标量。

```
scalar(count(prometheus_http_requests_total))
```

## PromQL 操作符

### 数学运算符

查询 Prometheus 这个应用的 HTTP 响应字节总和，用MB显示：

```
prometheus_http_response_size_bytes_sum{handler="/metrics"}/1024/1024
```

计算主机节点的内存使用率：

```
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes) / node_memory_MemTotal_bytes
```

查看内存使用率超过50%的主机：

```
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes) / node_memory_MemTotal_bytes >0.5
```

### 布尔运算符

布尔运算符的默认行为是对时序数据进行过滤。而在其它的情况下我们可能需要的是真正的布尔结果。例如，要知道prometheus_http_requests_total 请求量是否>=1000，如果大于等于1000则返回1（true）否则返回0（false）。这时可以使用bool修饰符改变布尔运算的默认行为。 例如：

```
prometheus_http_requests_total > bool 1000
```

使用bool修改符后，布尔运算不会对时间序列进行过滤，而是直接依次瞬时向量中的各个样本数据与标量的比较结果0或者1。从而形成一条新的时间序列。

如果是在两个标量之间使用布尔运算，则必须使用bool修饰符：

```
2 == bool 2 # 结果为1
```

### 集合运算符

使用瞬时向量表达式能够获取到一个包含多个时间序列的集合，我们称为瞬时向量。 通过集合运算，可以在两个瞬时向量与瞬时向量之间进行相应的集合操作。目前，Prometheus支持以下集合运算符：

- `and` (并且)：例如我们有 vector1 为 A B C，vector2 为 B C D，那么 vector1 and vector2 的结果为：B C。
- `or` (或者)：例如我们有 vector1 为 A B C，vector2 为 B C D，那么 vector1 or vector2 的结果为：A B C D。
- `unless` (排除)：例如我们有 vector1 为 A B C，vector2 为 B C D，那么 vector1 unless vector2 的结果为：A。

### 操作符优先级

1. `^`
2. `*, /, %`
3. `+, -`
4. `==, !=, <=, <, >=, >`
5. `and, unless`
6. `or`

## PromQL 聚合操作

- sum (求和)
- min (最小值)
- max (最大值)
- avg (平均值)
- stddev (标准差)：标准差就是为了描述数据集的波动大小
- stdvar (标准方差)
- count (计数)
- count_values (对value进行计数)
- bottomk (后n条时序)
- topk (前n条时序)
- quantile (分位数)

聚合操作的语法如下：

```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

其中只有`count_values`, `quantile`, `topk`, `bottomk`支持参数(parameter)。

without用于从计算结果中移除列举的标签，而保留其它标签。by则正好相反，结果向量中只保留列出的标签，其余标签则移除。通过without和by可以按照样本的问题对数据进行聚合。

```
sum(prometheus_http_requests_total) without (instance)
sum(prometheus_http_requests_total) by (code,handler,job,method)
```

# PromQL内置函数

### 计算Counter指标增长率

counter 类型指标的特点是只增不减，在没有发生重置的情况下，其样本值是不断增大的。为了能直观地观察期变化情况，需要计算样本的增长率。

`increase(v range-vector)` 参数v是一个区间向量，increase函数获取区间向量中的第一个后最后一个样本并返回其增长量。

下面计算最近两分钟CPU的平均使用率：

```
increase(node_cpu_seconds_total[2m]) / 120
```

rate(v range-vector)函数可以直接计算区间向量v在时间窗口内平均增长速率：

```
rate(node_cpu_seconds_total[2m])
```

函数irate(v range-vector)同样用于计算区间向量的增长速率，但是其反应出的是瞬时增长率。irate函数是通过区间向量中最后两个两本数据来计算区间向量的增长速率。

```
irate(node_cpu_seconds_total[2m])
```

irate函数相比于rate函数提供了更高的灵敏度，不过当需要分析长期趋势或者在告警规则中，irate的这种灵敏度反而容易造成干扰。因此在长期趋势分析或者告警中更推荐使用rate函数。

### 预测Gauge指标变化趋势

predict_linear(v range-vector, t scalar) 函数可以预测时间序列v在t秒后的值。它基于简单线性回归的方式，对时间窗口内的样本数据进行统计，从而可以对时间序列的变化趋势做出预测。例如，基于2小时的样本数据，来预测主机可用磁盘空间的是否在4个小时候被占满，可以使用如下表达式：

```
predict_linear(node_filesystem_free_bytes{instance="test-120-82"}[2h], 4 * 3600) < 0
```

### 统计Histogram指标的分位数

Histogram和Summary都可以同于统计和分析数据的分布情况。区别在于Summary是直接在客户端计算了数据分布的分位数情况。而Histogram的分位数计算需要通过histogram_quantile(φ float, b instant-vector)函数进行计算。

```
histogram_quantile(0.5, prometheus_http_request_duration_seconds_bucket)
```

### 动态标签替换

通过up指标可以获取到当前所有运行的Exporter实例以及其状态：

```
up{instance="test-120-82", job="node-testEnv"} 
```

通过label_replace标签为时间序列添加额外的标签，参数如下：

```
label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)
```

该函数会依次对v中的每一条时间序列进行处理，通过regex匹配src_label的值，并将匹配部分relacement写入到dst_label标签中。如下所示：

```
label_replace(up{instance!="localhost:9090"}, "host", "192.168.$1.$2", "instance", ".*-(.*)-(.*)")
```

函数处理后，时间序列将包含一个host标签：

```
up{host="192.168.120.82", instance="test-120-82", job="node-testEnv"}
up{host="192.168.120.83", instance="test-120-83", job="node-testEnv"}
```











