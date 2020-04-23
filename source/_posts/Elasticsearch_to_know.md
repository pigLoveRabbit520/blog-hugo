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