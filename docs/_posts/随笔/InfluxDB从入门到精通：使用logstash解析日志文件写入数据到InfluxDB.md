---
title: InfluxDB从入门到精通：使用logstash解析日志文件写入数据到InfluxDB
date: 2023-07-19 21:00:00
permalink: /pages/3787f74j43/
sidebar: auto
categories:
  - 随笔
tags:
  - 
author: 
  name: hexin
  link: https://github.com/ahxinin
---


经过上一篇文章，我们对InfluxDB已经有了基本的了解，接下来看看如何实现：读取日志文件并写入数据到InfluxDB中。
## 1.方案设计
### 1.1.整体流程图
由Logstash监听应用服务的日志文件变更，读取数据并进行处理后，写入到InfulxDB中。
![在这里插入图片描述](https://blog-7gg8g2uhb3511448-1254197092.tcloudbaseapp.com/img_convert/%E4%B8%B4%E6%97%B6%E6%96%87%E4%BB%B6.png)
### 1.2.Logstash流程
Logstash是一个开源的数据收集引擎，具有实时流水线功能。它通过插件的方式提供各类输入、过滤、输出数据的处理方式。
![在这里插入图片描述](https://blog-7gg8g2uhb3511448-1254197092.tcloudbaseapp.com/img_convert/diagram-logstash-filters.svg)
## 2.实现步骤
###  2.1.安装logstash
```
docker pull docker.elastic.co/logstash/logstash:8.8.2
```
### 2.2.配置文件
#### 1.logstash配置
在 /data/logstash/config 路径下创建 logstash.yml 作为 logstash 的配置文件，详细配置项：https://www.elastic.co/guide/en/logstash/8.8/logstash-settings-file.html

```bash
node.name: logstash2023
path.logs: /usr/share/logstash/logs
log.level: debug
```
#### 2.流水线定义
在 /data/logstash/pipeline 路径下创建 logstash.conf，配置数据处理流程，输出仅打印了日志，稍后会增加写入InfluxdDB步骤。

```bash
input {
	file{
		path => "/usr/share/logstash/data/test.log"
		start_position => "beginning"
	}
}

filter {
  grok {
    match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] \[%{LOGLEVEL:loglevel}\] - %{GREEDYDATA:log_json}" }
  }
  json {
	source => "log_json"
  }
  mutate {
    remove_field => ["message", "path", "original"]
  }
}

output {
	stdout {
        codec => rubydebug
    }
}
```
#### 3.流水线配置
在 /data/logstash/config 路径下创建 pipelines.yml，增加以下配置：

```bash
- pipeline.id: main
  path.config: /usr/share/logstash/pipeline/logstash.conf
```

### 2.3.启动logstash
```bash
docker run -it --network host --name logstashV2 -v /data/logstash/config:/usr/share/logstash/config -v /data/logstash/data:/usr/share/logstash/data -v /data/logstash/pipeline:/usr/share/logstash/pipeline  --privileged=true -d docker.elastic.co/logstash/logstash:8.8.2
```
### 2.4.写入日志
在日志文件中写入以下内容，从日志可以看到解析之后的数据。
```bash
[2023-07-14T10:22:15.810] [INFO] - {"flowId":"1","requestId":"r1","node":"e76c4bd185a45bd7"}
```
### 2.5.安装插件
开发文档提供的Influxdb output plugin插件，处于无人维护状态，因此并不支持InfluxDB 2.x版本，所以这里安装的是logstash-output-influx-v2插件。从以下仓库下载编译好的gem文件，上传到/data/logstash/data目录下，在容器中执行：
```bash
bin/logstash-plugin install /usr/share/logstash/data/logstash-output-influx_v2-1.0.1.gem
```
插件地址：
[https://github.com/ahxinin/logstash-output-influx-v2](https://github.com/ahxinin/logstash-output-influx-v2)

### 2.6.写入InfluxdDB
编辑 logstash.conf文件，修改output配置：

```bash
output {
	stdout {
        codec => rubydebug
    }
	
    influx_v2 {
		url => "http://127.0.0.1:8086"
		org =>  "org"
		token => "token"
		bucket => "logstash"
		measurement => "log"
		excludes => ['log_json']
		tags => ['flowId']
    }
}
```
重启容器，新的日志内容将会通过流水线写入到InfluxDB中。

