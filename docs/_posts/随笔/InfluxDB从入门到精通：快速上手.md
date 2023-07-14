---
title: InfluxDB从入门到精通：快速上手
date: 2023-06-27 21:00:00
permalink: /pages/3fe7f74j/
sidebar: auto
categories:
  - 随笔
tags:
  - 
author: 
  name: hexin
  link: https://github.com/ahxinin
---

## 1.InfluxDB介绍

InfluxDB是一个开源的时间序列平台。包括用于存储和查询的API，在后台为ETL或监控和告警处理数据，用户仪表盘，以及可视化和探索数据等。它由Go写成，致力于高性能时序型数据库，被广泛应用于存储系统的监控数据，loT行业的实时数据等场景。

> ETL，代表提取、转换和加载， 是一个数据集成过程，它将多个来源的数据结合到一个单一的、一致的数据存储库中，然后再将这个存储库装入到数据仓库或其他目标系统中。

### 技术概览

-   InfluxDB提供了一个类似于SQL的查询语言以及一系列内置函数来方便用户进行数据查询；
-   InfluxDB存储的数据从逻辑上由 Measurement, tag组以及 field组以及一个时间戳组成，tag信息是默认被索引的，可以实现快速查询；
-   InfluxDB支持基于HTTP的数据插入与查询，同时也支持基于TCP或UDP协议的连接；
-   InfluxDB允许用户定义数据保存策略(Retention Policies)来实现对存储超过指定时间的数据进行删除或者降采样。

### 数据模型

-   Bucket：存储时间序列数据的命名位置，包含多个 Measurement；
-   Measurement：时序数据的逻辑分组，表示该条记录对应的含义。在一个特定的测量中，所有点都应具备相同的标签。包含多个 tags 和 fields；
-   Tags：表示该条记录的一系列属性信息。键值对的值不一样，但是不经常变化。tags用来存储每个点的元数据，默认是被索引的，例如：地点；
-   Fields：表示该条记录的内容信息，值随时间变化的键值对，不能被索引，例如温度、股价等；
-   Timestamp：数据关联的时间戳。在存储和查询时，所有数据都是基于时间顺序的。

### 重要概念

-   Point：单条数据记录是由 measurement，tag keys, tag values, field key, timestamp来定义的；
-   Series：一组具有相同 measurement, tag keys, tag values的点；

![](https://blog-7gg8g2uhb3511448-1254197092.tcloudbaseapp.com/img_convert/f355bfd5dc31467db5c21978f1d83ecb~tplv-k3u1fbpfcp.png)

## 2.InfluxDB安装

InfluxDB支持MacOS、Linux、Windows各大平台，以及Docker、k8s容器化部署，v2.7安装步骤参考：

<https://docs.influxdata.com/influxdb/v2.7/install/>

InfluxDB启动成功后，支持使用命令行方式或者管理后台进行管理：

-   命令行工具 <https://docs.influxdata.com/influxdb/v2.7/tools/influx-cli/>
-   控制台地址 <http://localhost:8086/>

## 3.InfluxDB应用

### 3.1.初始化

设置 organization, bucket, username, password

<https://docs.influxdata.com/influxdb/v2.7/install/#set-up-influxdb>

### 3.2.写入数据

#### 行协议

所有写入InfluxDB的数据都是基于行协议的，是一种文本格式，让你提供所需信息，将数据点写入InfluxDB。

#### 行协议元素

-   measurement：字符串格式，用于标识存储数据的测量指标；
-   tag set：以逗号分割的键值对列表，每个代表一个标签。标签的键和值是无引号的字符串。空格、逗号和等于号必须要转义；
-   field set：以逗号分割的键值对老表，每个代表一个领域。领域的键和值是无引号的字符串。逗号和等于号必须要转义。领域的值格式可以是： strings (quoted), floats, integers, unsigned integers, or booleans.
-   timestamp：与数据相关的时间戳。InfluxDB支持纳秒级别的精度。如果精度不是纳秒级别，在写入数据时需要指定精度；

#### 行协议元素解析

-   measurement：第一个空格前，第一个未被转义的逗号前的内容；
-   tag set：第一个未被转义的逗号和空格之间的键值对；
-   field set：第一个和第二个未被转义空格之间的键值对；
-   timestamp：第二个未被转义的空格之后的整数内容；
-   行与行之间用换行符分割(\n)。行协议对空格敏感；

![](https://blog-7gg8g2uhb3511448-1254197092.tcloudbaseapp.com/img_convert/a12bb59a1d344b8abbdc51e8ee58349b~tplv-k3u1fbpfcp-zoom.png)

### 3.3.查询数据

#### 查询示例

![](https://blog-7gg8g2uhb3511448-1254197092.tcloudbaseapp.com/img_convert/b5805dbb5c9e4e878b4a7e0681c51b3e~tplv-k3u1fbpfcp-zoom-1.png)

### 3.4.处理数据

除了基本的查询操作，数据还可以进行转换、聚合、采样、告警等一系列处理操作。

#### 1.数据映射及修改操作

使用map()函数可以遍历所有数据，进行赋值操作。

#### 2.数据分组

使用group()函数，根据特定的列值对数据进行重新组合，以便进一步处理数据。

#### 3.数据聚合

使用aggregate()函数，对数据进行聚合统计。

#### 4.数据采样

使用aggreageWindow()函数，来对指定时间间隔进行数据采样操作。

#### 5.自动处理任务

InfluxDB task是一个定时Flux脚本，获取输入数据，以某种方式进行修改、分析，将结果写回InfluxDB或者执行其他操作。

### 3.5.数据可视化

有多种方式可以实现时序数据的图表化展示，例如： InfluxDB UI、Grafana。

#### 1.InfluxDB UI

在InfluxDB控制台的dashboard模块进行配置，即可实现数据可视化，效果图如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2194ba13058847caa610a699dc760ce9~tplv-k3u1fbpfcp-zoom-1.image)

#### 2.Grafana

Grafana是一个开源的度量分析和可视化工具，提供了丰富的可视化展示方式，能够帮助用户快速的查看和编辑dashboard的前端。

## 4.InfluxDB开发

InfluxDB提供了丰富的API和客户端SDK，来与应用程序集成。

API文档：<https://docs.influxdata.com/influxdb/v2.7/api/>

Java SDK：<https://github.com/influxdata/influxdb-client-java>