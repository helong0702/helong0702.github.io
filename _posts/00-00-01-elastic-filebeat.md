---
categories: ELK
date: 2017-08-30 14:31
description: 'filebeat日志收集，性能选择'
keywords: filebeat,logstash,ELK
layout: post
status: public
title: filebeat日志收集
---

&nbsp;&nbsp;&nbsp;&nbsp;轻量型日志采集器，这是官网上对filebeat的第一句描述，[]

&nbsp;&nbsp;&nbsp;&nbsp;filebeat日志收集器，一样是elastic公司旗下产品，所设定的角色与logstash类似又有不同之处。

&nbsp;&nbsp;&nbsp;&nbsp;logstash灵活的日志收集器，集成大量的插件，几乎可以满足你的所有日志收集的需求。但是logstash在性能上面为人们所病垢，logstash启动堆内存1G占用大量的系统资源。

&nbsp;&nbsp;&nbsp;&nbsp;filebeat作为轻量级日志收集器，性能方面比logstash优异非常多，logstash却省在灵活。

> 重点
>
> filebeat作为elastic的成员，可以无缝的加入到原来的ELK结构当中，可以使用filebeat替代logstash的Shipper角色。同时filebeat可以直接输出到redis、logstash、elasticsearch等。

### beats系列采集器

filebeat是beats下面的一个子产品，beats下面还有其他的产品：

|产品         |描述         |
|:-----------|:------------|
|Filebeat    |文件日志      |
|Metricbeat  |指标          |
|Packetbeat  |网络数据      |
|Winlogbeat  |Windows 事件日志 |
|Heartbeat   |运行时间监控   |

### 安装filebast采集器

MAC安装：

```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.5.2-darwin-x86_64.tar.gz
tar xzvf filebeat-5.5.2-darwin-x86_64.tar.gz
```

rpm:

```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.5.2-x86_64.rpm
sudo rpm -vi filebeat-5.5.2-x86_64.rpm
```

```
docker pull docker.elastic.co/beats/filebeat:5.5.2
```

### 配置

配置文件：filebeat/filebeat.yml, [官方文档](https://www.elastic.co/guide/en/beats/filebeat/5.5/configuration-filebeat-options.html#_input_type)

```
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/*.log
    #- c:\programdata\elasticsearch\logs\*
    
output.elasticsearch:
  hosts: ["192.168.1.42:9200"] 
  
  //负载均衡
output.elasticsearch:
  hosts: ["localhost:5044", "localhost:5045"]
  loadbalance: true
```

|配置         |功能描述                               |
|:------------|:-------------------------------------|
|input_type   |log,stdin          |
|include_lines|接受正则表达式，接受的行：include_lines：[“^ ERR”，“^ WARN”]|
|exclude_lines|接受正则表达式，排除的行                 |
|exclude_files|接受正则表达式，排除的文件               |
|processors   |通过规定的语法过滤消息                   |


目前5.5版本支持的output：[官方文档详细配置](https://www.elastic.co/guide/en/beats/filebeat/5.5/filebeat-configuration-details.html)

结构思考：

1、多个filebeat -> (可以是多个缓存作用)redis -> logstash - elasticsearch

2、多个filebeat -> elasticsearch

3、多个filebeat -> logstash -> elasticsearch

4、filebeat与logstash结合(作为Shipper角色) -> (可以是多个缓存作用)redis -> logstash - elasticsearch


&nbsp;&nbsp;&nbsp;&nbsp;不同情况不同的选择，filebeat -> elasticsearch filebeat及elasticsearch目前5.0以后的版本已经支持数据过滤(filter)，服务器数量不是非常多，并且filebeat能够满足日志收集的话，可以使用这种方式

&nbsp;&nbsp;&nbsp;&nbsp;filebeat -> logstash -> elasticsearch把logstash作为日志过滤器来使用，不推荐（个人）

&nbsp;&nbsp;&nbsp;&nbsp;由于logstash的灵活性及强大之处（非常多的插件，各种日志处理），可以使用filebeat与logstash结合，logstash中定义多个input，filebeat作为input之一，满足一种文件收集

### redis output

常用属性: 

|属性          |                  |
|:------------|:------------------|
|hosts        |redis主机，可以配置多个|
|port         |端口               |
|key          |redis发布的频道     |
|keys         |设置动态频道        |
|password     |密码               |
|db           |默认0, redis的0数据库|
|dataType     |list（Redis RPUSH）、channel(Redis PUBLISH)|
|codec        |编解码器            |
|worker       |启动多少线程工作，2个host，配置3，那么一共启动6个  |
|loadbalance  |true,负载均衡,     |
|timeout      |链接redis超时时间   |
|bulk_max_size|单个Redis请求或管道中的批量事件的最大数量。默认值为2048。|
|max_retries  |发布失败时，重试的次数，小于0的值重试，直到所有事件发布|

> bulk_max_size
>
> 如果节拍发送单个事件，则会将事件分批收集。如果Beat发布大批事件（大于指定的值 bulk_max_size），则批处理将被拆分
>
> 指定更大的批量大小可以通过降低发送事件的开销来提高性能。然而，大批量大小也可能会增加处理时间，这可能会导致API错误，杀死连接，超时发布请求以及最终降低吞吐量。
>
> 设置bulk_max_size为小于或等于0的值将禁用libbeat中的缓冲。当禁用缓冲时，发布单个事件（例如Packetbeat）的Beats会将每个事件直接发送到Redis。批量发布数据（如Filebeat）的节拍将根据假脱机程序大小批量发送事件

&nbsp;&nbsp;&nbsp;&nbsp;redis outout 通过Redis RPUSH（默认）、Pub/Sub（发布/订阅）实现日志的传输，key/keys属性来指定对应的频道，其中keys作为动态设置频道

```
output.redis:
  hosts: ["localhost"]
  password: "my_password"
  key: "filebeat"
  db: 0
  timeout: 5
```

```
output.redis:
  hosts: ["localhost"]
  key: "default_list"
  keys:
    - key: "info_list"   # send to info_list if `message` field contains INFO
      when.contains:
        message: "INFO"
    - key: "debug_list"  # send to debug_list if `message` field contains DEBUG
      when.contains:
        message: "DEBUG"
    - key: "%{[type]}"
      mapping:
        "http": "frontend_list"
        "nginx": "frontend_list"
        "mysql": "backend_list"
```


配置完成之后通过命令测试：

```
filebeat.sh -configtest -e
```

> 注意
>
> conf目录下面有一个文件filebeat.full.yml，这个文件里面相信描述了所有的配置属性
>
> 开发的时候可以开启日志级别为debug级别，以及实时发送日志。不开启filelib日志缓存
```
logging.level: debug
bulk_max_size: 0
```

下面来看一段filebeat运行时的日志：

DBG  Drop line as it does match one of ... 这一段话的意思是符合exclude_lines表达式删除了行

DBG  Publish: {... 正确发送数据了

```
2017/08/31 07:49:25.867301 log.go:206: DBG  Drop line as it does match one of the exclude patterns2017-08-31 15:49:25.721 DEBUG 15312 --- [nio-8080-exec-8] com.lefit.service.UserService            : {"result":{"openid":"o23m8s9Zaxrs2eVaLo6CDIHsp0As","user_id":117146,"exp_timestamp":1504251639},"status":{"code":"00000","msg":"处理成功","runtime":"42.6650"}}
2017/08/31 07:49:25.867321 log_file.go:84: DBG  End of file reached: /opt/apps/logs/stdout.log; Backoff now.
2017/08/31 07:49:26.380197 metrics.go:39: INFO Non-zero metrics in the last 30s: libbeat.publisher.published_events=7 libbeat.redis.publish.read_bytes=28 libbeat.redis.publish.write_bytes=3297 publish.events=93 registrar.states.update=93 registrar.writes=6
2017/08/31 07:49:26.556064 prospector.go:183: DBG  Run prospector
2017/08/31 07:49:26.556092 prospector_log.go:70: DBG  Start next scan
2017/08/31 07:49:26.556156 prospector_log.go:226: DBG  Check file for harvesting: /opt/apps/logs/stdout.log
2017/08/31 07:49:26.556170 prospector_log.go:259: DBG  Update existing file for harvesting: /opt/apps/logs/stdout.log, offset: 1253244
2017/08/31 07:49:26.556176 prospector_log.go:311: DBG  Harvester for file is still running: /opt/apps/logs/stdout.log
2017/08/31 07:49:26.556183 prospector_log.go:91: DBG  Prospector states cleaned up. Before: 1, After: 1
2017/08/31 07:49:26.613796 spooler.go:89: DBG  Flushing spooler because of timeout. Events flushed: 17
2017/08/31 07:49:26.613927 client.go:214: DBG  Publish: {
  "@timestamp": "2017-08-31T07:49:25.867Z",
  "beat": {
    "hostname": "iZbp10iksa8sifg1cmgs6aZ",
    "name": "iZbp10iksa8sifg1cmgs6aZ",
    "version": "5.5.2"
  },
  "input_type": "log",
  "message": "2017-08-31 15:49:25.170  INFO 15312 --- [nio-8080-exec-3] com.lefit.controller.CoachController     : 查询请求入参：{\"coachId\":841,\"page\":{\"pageSize\":10,\"start\":0},\"userId\":117146}",
  "offset": 1252661,
  "source": "/opt/apps/logs/stdout.log",
  "type": "log"
}

```