title: Elasticsearch简单使用
author: Salamander
tags:
  - Java
  - Elasticsearch
categories:
  - Java
date: 2020-04-21 16:00:00
---
![](https://s1.ax1x.com/2020/04/23/Jd4MOf.png)

## Elasticsearch
`Elasticsearch`是一个基于Lucene的搜索服务器（Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库）。Elasticsearch使用Java编写并使用Lucene来建立索引并实现搜索功能，但是它的目的是通过简单连贯的`RESTful API`让全文搜索变得简单并隐藏Lucene的复杂性。 

<!-- more -->


## 安装单节点Elasticsearch
`Elasticsearch`是用Java语言开发的，所以你需要安装Java环境，但为了方便起见，我这里选用了`Docker`安装ES（其实ES官网也有用Docker安装的[例子](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/docker.html#docker)，这里我就把它写成了docker-compose服务）。  
docker-compose.yml
```
version: '2.2'
services:
  es:
    image: elasticsearch:6.7.0
    container_name: es0
    environment:
      - node.name=es0
      - "discovery.type=single-node"
      - node.data=true
      - bootstrap.memory_lock=true
      - network.host=0.0.0.0
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
```
Elasticsearch 启动后，启动了两个端口 9200 和 9300：
* 9200 端口：HTTP RESTful 接口的通讯端口
* 9300 端口：TCP 通讯端口，用于集群间节点通信和与 Java 客户端通信的端口

```
- bootstrap.memory_lock=true
...
ulimits:
  memlock:
    soft: -1
    hard: -1
```
这两部分是为了禁止使用swap内存，因为当jvm开始swapping时ES的效率会降低。  

`-Xms1g -Xmx1g`是设置了JVM的初始堆大小和最大堆大小（具体参数含义可以参考[这里](https://www.cnblogs.com/redcreen/archive/2011/05/04/2037057.html)）。 

启动之前还是需要调高JVM线程数限制数量，不然启动会报错
```
vim /etc/sysctl.conf
# 添加这个
vm.max_map_count=262144 
# 保存后执行这个命令
sysctl -p
```


## 概念
在进一步使用 Elasticsearch 之前，让我们先了解几个关键概念。

在逻辑层面：  
* Index (索引)和Type (类型)：这里的 Index 是名词，在之前开始的时候，我们把**索引（index）和类型（type）**类比于SQL数据库中的 database 和 table，但是这样类比是不合适的。刚开始的ES版本一个索引可以有多个类型，在ES6之后每个索引只能有一个类型，所以他们现在都可以当做一张数据表了。
* Document (文档)：Elasticsearch 使用 JSON 文档来表示一个对象，就像是关系数据库中一个 Table 中的一行数据
* Field (字段)：每个文档包含多个字段，类似关系数据库中一个 Table 的列

>>>
为什么要移除映射类型
开始的时候，我们把**索引（index）和类型（type）**类比于SQL数据库中的 database 和 table，但是这样类比是不合适的。在SQL数据库中，表之间是相互独立的。一个表中的各列并不会影响到其它表中的同名的列。而在映射类型（mapping type）中却不是这样的。
在同一个 Elasticsearch 索引中，其中不同映射类型中的同名字段在内部是由同一个 Lucene 字段来支持的。换句话说，使用上面的例子，user 类型中的 user_name 字段与 tweet 类型中的 user_name 字段是完全一样的，并且两个 user_name 字段在两个类型中必须具有相同的映射（定义）。
这会在某些情况下导致一些混乱，比如，在同一个索引中，当你想在其中的一个类型中将 deleted 字段作为 date 类型，而在另一个类型中将其作为 boolean 字段。
在此之上需要考虑一点，如果同一个索引中存储的各个实体如果只有很少或者根本没有同样的字段，这种情况会导致稀疏数据，并且会影响到Lucene的高效压缩数据的能力


在物理层面：  

* Node (节点)：node 是一个运行着的 Elasticsearch 实例，一个 node 就是一个单独的 server
* Cluster (集群)：cluster 是多个 node 的集合
* Shard (分片)：数据分片，一个 index 可能会存在于多个 shard



## 使用
下面，我们将创建一个存储电影信息的 Document：
* Index 的名称为 movie
* Type 为 adventure
* Document 有两个字段：name 和 actors

我们使用 Elasticsearch 提供的 RESTful API 来执行上述操作，如图所示：  

![](https://s1.ax1x.com/2020/04/23/Jw9Ov6.png)

* 用 url 表示一个资源，比如 /movie/adventure/1 就表示一个 index 为 movie，type 为 adventure，id 为 1 的 document
* 用 http 方法操作资源，如使用 GET 获取资源，使用 POST、PUT 新增或更新资源，使用 DELETE 删除资源等

用`curl`命令实现上述操作：
```
curl -i -X PUT "localhost:9200/movie/adventure/1" -H 'Content-Type: application/json' -d '{"name": "Life of Pi", "actors": ["Suraj", "Irrfan"]}'
```
ES6之后需要加上head`Content-Type: application/json`

上述命令返回：
```
HTTP/1.1 201 Created
Location: /movie/adventure/1
content-type: application/json; charset=UTF-8
content-length: 158

{"_index":"movie","_type":"adventure","_id":"4","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":1,"_primary_term":1}
```




























## elasticsearch.yml配置说明
```
#集群的名称
cluster.name: es6.2
#节点名称,其余两个节点分别为node-2 和node-3
node.name: node-1
#指定该节点是否有资格被选举成为master节点，默认是true，es是默认集群中的第一台机器为master，如果这台机挂了就会重新选举master
node.master: true
#允许该节点存储数据(默认开启)
node.data: true
#索引数据的存储路径
path.data: /usr/local/elk/elasticsearch/data
#日志文件的存储路径
path.logs: /usr/local/elk/elasticsearch/logs
#设置为true来锁住内存。因为内存交换到磁盘对服务器性能来说是致命的，当jvm开始swapping时es的效率会降低，所以要保证它不swap
bootstrap.memory_lock: true
#绑定的ip地址
network.host: 0.0.0.0
#设置对外服务的http端口，默认为9200
http.port: 9200
# 设置节点间交互的tcp端口,默认是9300 
transport.tcp.port: 9300
#Elasticsearch将绑定到可用的环回地址，并将扫描端口9300到9305以尝试连接到运行在同一台服务器上的其他节点。
#这提供了自动集群体验，而无需进行任何配置。数组设置或逗号分隔的设置。每个值的形式应该是host:port或host
#（如果没有设置，port默认设置会transport.profiles.default.port 回落到transport.tcp.port）。
#请注意，IPv6主机必须放在括号内。默认为127.0.0.1, [::1]
discovery.zen.ping.unicast.hosts: ["192.168.8.101:9300", "192.168.8.103:9300", "192.168.8.104:9300"]
#如果没有这种设置,遭受网络故障的集群就有可能将集群分成两个独立的集群 - 分裂的大脑 - 这将导致数据丢失
discovery.zen.minimum_master_nodes: 3
```



参考：
* [Elasticsearch6.2集群搭建](https://blog.csdn.net/qq_34021712/article/details/79330028)
* [Elasticsearch入门，这一篇就够了](https://www.cnblogs.com/sunsky303/p/9438737.html)