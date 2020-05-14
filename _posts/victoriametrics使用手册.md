# **Victoriametrics**使用手册

[TOC]

 

# 1.  基本介绍

VictoriaMetrics是快速、经济高效且可扩展的时间序列数据库，它可以用作Prometheus的长期远程存储。

# 2.  为什么使用VictoriaMetrics

- 支持Prometheus查询API，因此可以在Grafana中用作Prometheus的替代产品。

- 支持全局查询视图。可以将多个Prometheus实例数据写入VictoriaMetrics，然后一起查询

- inserts和selects有好的性能和可伸缩性。 性能比InfluxDB和TimescaleDB高出20%。

- 在处理数百万个不同的时间序列时，使用的内存比InfluxDB少10倍。

- 较高的数据压缩率。与TimescaleDB相比，有限的存储空间中最多可以塞满70倍的数据。

- 针对具有高延迟IO和低IOPS（AWS，Google Cloud，Microsoft Azure等中的HDD和网络存储）的存储进行了优化。

- 单节点VictoriaMetrics可以替代一些（如Thanos，M3DB，Cortex，InfluxDB或TimescaleDB）中等大小的集群。

- 操作简便：

- - VictoriaMetrics由一个没有外部依赖性的小型可执行文件组成。
  - 所有配置都是通过带有合理默认值的显式命令行标志完成的。
  - 所有数据都存储在`-storageDataPath`指向的单个目录中。
  - 使用vmbackup/ vmrestore从即时快照轻松快速地备份到S3或GCS 。

- 可以保护存储免受不正常关机（例如OOM，硬件重置或kill -9）损坏的损坏。

- 支持以下协议实现指标的抓取，提取和回填：

- - Prometheus支持的exporters指标
  - Prometheus远程写入API
  - HTTP，TCP和UDP的InfluxDB线路协议。
  - graphite
  - OpenTSDB
  - HTTP接口` /api/v1/import`。
  - 任意CSV数据。

- 理想情况下，可处理来自Kubernetes，IoT传感器，联网汽车，工业遥测，财务数据和各种企业工作负载的大量时间序列数据。

- 具有开源集群版本。

 

# 3.  安装部署

## 3.1. 下载

下载二进制文件或者docker镜像。[VictoriaMetrics单节点下载地址](https://github.com/VictoriaMetrics/VictoriaMetrics/releases)

## 3.2. 二进制文件启动

`./victoria-metrics-prod`（注意二进制文件导入虚拟机后修改文件权限）

启动参数：

- `-storageDataPath` -数据存储路径。默认当前目录的victoria-metrics-data 。
- `-retentionPeriod` -数据的保留期限（以月为单位）。旧数据将自动删除。默认期限为1个月。
- `-httpListenAddr`-监听HTTP请求的TCP地址。默认8428。
- `-dedup.minScrapeInterval`删除多少秒之内的重复数据。默认为0即不删除重复数据。
- `-help`查看所有可用的参数以及说明和默认值。

# 4.  数据插入

​         VictoriaMetrics支持exporters指标、Prometheus写入、InfluxDB协议、graphite、OpenTSDB、HTTP接口、任意CSV数据等数据插入，这里只介绍Prometheus远程写入API、InfluxDB协议、HTTP接口。其他方式可见[VictoriaMetrics官方文档](https://github.com/VictoriaMetrics/VictoriaMetrics)

## 4.1. Prometheus数据插入

·     在prometheus配置文件（prometheus.yml）中加入如下设置

```
remote_write:

   - url: http://<victoriametrics-addr>:8428/api/v1/write
```

·    可以添加标签来区分不同prometheus的数据

```
global:
  external_labels:
    datacenter: <label>
```

·    对于高负载的prometheus，可以通过`max_samples_per_send`和`capacity`来控制使用的内存，推荐设置如下

```
remote_write:
  - url: http://<victoriametrics-addr>:8428/api/v1/write
    queue_config:
      max_samples_per_send: 10000
      capacity: 20000
      max_shards: 30
```

·    在线更新prometheus配置

```
kill -HUP `pidof prometheus`
```

## 4.2. influxDB协议

可以使用influxDB的往victoriametrics插入数据，这里以python使用influxDB协议为例。

导入influxDB库

```
 pip install influxdb
```

使用InfluxDBClient方法连接victoriametrics，只需要在后面的参数中加入host和port，插入数据的格式同influxDB的格式。其中measurement+fields的属性为指标名，比如下面这个demo，指标名为`students_age`，标签为name，8、13是对应的value。

```python
from influxdb import InfluxDBClient
import datetime
client = InfluxDBClient(host=<victoriametrics-addr>,port='8428')
current_time = datetime.datetime.utcnow().isoformat("T")
body1 = [
    {
        "measurement": "students",
         "time": current_time,
        "tags": {
            "name": "lisi"
        },
        "fields": {
            "age": 8
        },
    },
    {
        "measurement": "students",
         "time": current_time,
        "tags": {
            "class": "zhangsan"
        },
        "fields": {
            "age": 13
        },
    },
]
client.write_points(body1)
```

## 4.3. HTTP接口

`/write`：可以直接接收InfluxDB和prometheus格式的数据。例如，想要写入InfluxDB格式的数据 “foo,tag1=value1,tag2=value2 field1=12,field2=40”

```
curl -d 'measurement,tag1=value1,tag2=value2 field1=123,field2=1.23' -X POST 'http://<victoriametrics-addr>:8428/write'
```

`/api/v1/import`：使用`/api/v1/export`可以将数据导出为json或者压缩文件，下文会有介绍。可以通过`/api/v1/import`导入这些json和压缩文件。

```
# Export the data from <source-victoriametrics>:
curl http://source-victoriametrics:8428/api/v1/export -d 'match={__name__!=""}' > exported_data.jsonl

# Import the data to <destination-victoriametrics>:
curl -X POST http://destination-victoriametrics:8428/api/v1/import -T exported_data.jsonl

# Export gzipped data from <source-victoriametrics>:
curl -H 'Accept-Encoding: gzip' http://source-victoriametrics:8428/api/v1/export -d 'match={__name__!=""}' > exported_data.jsonl.gz

# Import gzipped data to <destination-victoriametrics>:
curl -X POST -H 'Content-Encoding: gzip' http://destination-victoriametrics:8428/api/v1/import -T exported_data.jsonl.gz
```

# 5.  数据查询和导出

victoriaMetrics数据的查询主要是支持使用prometheus查询API，/api/v1/export接口也可以查询和导出

## 5.1.  prometheus查询API

 VictoriaMetrics支持以下[Prometheus查询API](https://prometheus.io/docs/prometheus/latest/querying/api/)接口:

- [/ api / v1 / query](https://prometheus.io/docs/prometheus/latest/querying/api/#instant-queries) ：单个时间点查询
- [/ api / v1 / query_range](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries)：范围时间查询
- [/ api / v1 /系列](https://prometheus.io/docs/prometheus/latest/querying/api/#finding-series-by-label-matchers)
- [/ api / v1 / labels](https://prometheus.io/docs/prometheus/latest/querying/api/#getting-label-names)：查询所有标签
- [/api/v1/label/.../values](https://prometheus.io/docs/prometheus/latest/querying/api/#querying-label-values)：查询指定标签的所有指标
- [/ api / v1 / status / tsdb](https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-stats)：返回有关TSDB的各种基数统计信息

可以从与Prometheus兼容的客户端（例如Grafana或curl）中查询这些处理程序。

此外，VictoriaMetrics还提供以下处理程序：

- `/api/v1/series/count`-它返回数据库中时间序列的总数。请注意，此处理程序将扫描所有反向索引，因此如果数据库包含数千万个时间序列，则处理速度可能会很慢。
- `/api/v1/labels/count`-返回`label: values_count`条目列表。可用于确定具有最大数量值的标签。

下面介绍一下写代码最常用的[/ api / v1 / query_range](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries)参数和返回值格式，也可以点击链接去官网学习.。

请求参数：

- `query=`：Prometheus表达式查询字符串。
- `start=`：开始时间戳。
- `end=`：结束时间戳。
- `step=`：查询步长、单位秒。
- `timeout=`：超时时间。

返回值格式：

```
{
   "status" : "success",
   "data" : {
      "resultType" : "matrix",
      "result" : [
         {
            "metric" : {//指标名和所有标签
               "__name__" : "up",
               "job" : "prometheus",
            },
            "values" : [//数据值
               [ 1435781430.781, "1" ],
               [ 1435781445.781, "1" ],
               [ 1435781460.781, "1" ]
            ]
         }
      ]
   }
}
```

## 5.2.  /api/v1/export接口

将请求发送到`http://:8428/api/v1/export?match[]=<timeseries_selector_for_export>`，其中`<timeseries_selector_for_export>`可能包含用于导出指标的任何选择条件，这样可以查询出对应数据。接口后面可以接`>json_name.json `将查询结果导出为json格式，也导出为gz压缩文件。

下面是常用的选择条件和参数:

`{__name__="<name>"}`可用于选择指标名

`{start=“start_time”,end="end_time"}`可用于选择时间，时间可以是以秒为单位的Unix时间戳或[RFC3339](https://www.ietf.org/rfc/rfc3339.txt)值

`{max_rows_per_line=int}`可以限制导出的每条json最大行数

`Accept-Encoding: gzip`在头部中加入，将对导出的数据启用gzip压缩。

示例：

```
curl -H 'Accept-Encoding: gzip' http://source-victoriametrics:8428/api/v1/export -d 'match={__name__!=""}' > exported_data.jsonl.gz
```

返回格式

```
{"metric":{"__name__":"up","job":"node_exporter","instance":"localhost:9100"},"values":[0,0,0],"timestamps":[1549891472010,1549891487724,1549891503438]}
{"metric":{"__name__":"up","job":"prometheus","instance":"localhost:9090"},"values":[1,1,1],"timestamps":[1549891461511,1549891476511,1549891491511]}
```

# 6.  删除数据和使用快照

## 6.1.  删除数据API

将请求发送到`http://:8428/api/v1/admin/tsdb/delete_series?match[]=<timeseries_selector_for_delete>`，其中`<timeseries_selector_for_delete>`可能包含用于删除指标的任何选择条件.参数同`/api/v1/export`

delete API主要用于以下情况：

- 一次性删除意外写入的无效（或不需要的）时间序列。
- 通过[GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation)一次性删除用户数据。

在以下情况下，建议不要使用delete API：

- 定期清理不需要的数据，只是防止将不需要的数据写入VictoriaMetrics，这可以通过在[vmagent](https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/app/vmagent/README.md)中重新标记来完成。
- 通过删除不需要的时间序列来减少磁盘空间使用。这不能按预期方式工作，因为删除的时间序列会占用磁盘空间，直到下一次合并操作为止，这在删除太旧的数据时永远不会发生。

最好使用`-retentionPeriod`命令行标志来有效修剪旧数据。

## 6.2.  快照

VictoriaMetrics可以为`-storageDataPath`目录下存储的所有数据创建[即时快照](https://medium.com/@valyala/how-victoriametrics-makes-instant-snapshots-for-multi-terabyte-time-series-data-e1f3fb0e0282)。使用`http://<victoriametrics-addr>:8428/snapshot/create`创建即时快照。该页面将返回以下JSON响应：

```
{"status":"ok","snapshot":"<snapshot-name>"}
```

快照是在`<-storageDataPath>/snapshots`目录下创建的，该目录`<-storageDataPath>` 是命令行标志值。可以使用[vmbackup](https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/app/vmbackup/README.md)随时将快照存档到备份存储中。

`http://<victoriametrics-addr>:8428/snapshot/list`页面包含可用快照的列表。

`http://<victoriametrics-addr>:8428/snapshot/delete?snapshot=<snapshot-name>`为了删除`<snapshot-name>`快照。

`http://<victoriametrics-addr>:8428/snapshot/delete_all`为了删除所有快照。

从快照还原的步骤：

1. 使用停止VictoriaMetrics `kill -INT`。
2. 使用[vmrestore](https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/app/vmrestore/README.md)将快照中的快照内容还原 到指向的目录中`-storageDataPath`。
3. 启动VictoriaMetrics。

# 7.  高可用

1. 安装多个VictoriaMetrics实例。

2. 将数据同时写入多个VictoriaMetrics实例，例如使用prometheus作为数据源：

   ```
   remote_write：
     - URL：HTTP：// <victoriametrics-ADDR-1>：8428 / API / V1 /写
       queue_config：
          max_samples_per_send：10000 
     ＃ ... 
     - URL：HTTP：// <victoriametrics-ADDR-N>：8428 / api / v1 / write 
       queue_config：
          max_samples_per_send：10000
   ```

3. 在所有VictoriaMetrics副本之前设置[Promxy](https://github.com/jacksontj/promxy)。

4. 所有请求指向Promxy.

# 8.  集群版本

建议使用[单节点版本](https://github.com/VictoriaMetrics/VictoriaMetrics)而不是群集版本，以使接收速率低于每秒一百万个数据点。单节点版本可以 根据CPU内核，RAM和可用存储空间的数量进行[完美扩展](https://medium.com/@valyala/measuring-vertical-scalability-for-time-series-databases-in-google-cloud-92550d78d8ae)。与群集版本相比，单节点版本更易于配置和操作。

集群版本的优点为支持性能和容量水平扩展、支持多个独立的名称空间用于时间序列数据。

## 8.1.  架构概述

VictoriaMetrics集群包含以下服务：

- `vmstorage` -存储数据
- `vminsert`-使用一致性hash算法插入`vmstorage`
- `vmselect` -查询 `vmstorage`

每个服务可以独立扩展，并且可以在最合适的硬件上运行。 `vmstorage`节点之间彼此不了解，彼此不通信并且不共享任何数据。它提高了群集可用性，简化了群集维护和群集扩展。

[![img](https://camo.githubusercontent.com/67fb28071f537a8837812e9e8ea9dcf6c649a648/68747470733a2f2f646f63732e676f6f676c652e636f6d2f64726177696e67732f642f652f32504143582d317654766b32726155396b46675a38346f462d4f4b6f6c72477748616550684852735a4563665131495f45433541425f5850577742333932587368785072616d4c4a38453462717074546e466e354c4c2f7075623f773d3131303426683d373436)](https://camo.githubusercontent.com/67fb28071f537a8837812e9e8ea9dcf6c649a648/68747470733a2f2f646f63732e676f6f676c652e636f6d2f64726177696e67732f642f652f32504143582d317654766b32726155396b46675a38346f462d4f4b6f6c72477748616550684852735a4563665131495f45433541425f5850577742333932587368785072616d4c4a38453462717074546e466e354c4c2f7075623f773d3131303426683d373436)

## 8.2.  集群搭建和使用

1. 下载二进制文件或者docker镜像。[VictoriaMetrics集群下载地址](https://github.com/VictoriaMetrics/VictoriaMetrics/tree/cluster)

2. 启动实例，启动方式参考单节点。最小群集必须包含以下节点：

   - `vmstorage`具有`-retentionPeriod`和`-storageDataPath`标志的单个节点
   - 一个`vminsert`节点`-storageNode=<vmstorage_host>:8400`
   - 一个`vmselect`节点`-storageNode=<vmstorage_host>:8401`

   为了实现高可用性，建议为每个服务至少运行两个节点。

3. 将http负载均衡器放在`vminsert`and `vmselect`节点的前面：

   - 插入请求必须被路由到`vminsert`节点的`8480`端口，其余同单节点。
   - 查询请求必须被路由到`vmselect`节点的`8481`端口，其余同单节点

   可以通过`-httpListenAddr`在相应节点上进行设置来更改端口。

## 8.3.  集群调整和可用性

​    群集性能和容量随添加新节点而扩展。

- `vminsert`并且`vmselect`节点是无状态的，可以随时添加/删除。不要忘记在http负载均衡器上更新这些节点的列表。添加更多`vminsert`节点可扩展数据摄取率。添加更多`vmselect`节点可扩展选择查询速率。
- `vmstorage`节点拥有提取的数据，因此无法删除它们而不会丢失数据。添加更多`vmstorage`节点可扩展群集容量。

添加`vmstorage`节点的步骤：

1. 以与集群中现有节点`vmstorage`相同的方式启动新节点`-retentionPeriod`。
2. `vmselect`使用`-storageNode`添加新的`vmstorage`，逐步重新启动所有节点`<new_vmstorage_host>:8401`。
3. `vminsert`使用`-storageNode`添加新的`vmstorage`，逐步重新启动所有节点`<new_vmstorage_host>:8400`。

HTTP负载平衡器停止将请求发送到不可用`vminsert`和`vmselect`节点。

如果至少`vmstorage`存在一个节点，则群集仍然可用：

- `vminsert`将传入数据从不可用`vmstorage`节点重新路由到运行状况良好的`vmstorage`节点
- `vmselect`如果至少有一个`vmstorage`节点可用，则继续提供部分响应。