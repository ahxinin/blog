---
title: Elasticsearch Client 进阶使用
date: 2023-04-14 23:31:00
permalink: /pages/3fe7f9/
sidebar: auto
categories:
  - 随笔
tags:
  - 
author: 
  name: hexin
  link: https://github.com/ahxinin
---

在上一篇文章中，我们介绍了Elasticsearch java  client的一些基本用法，为了达到生产级别的使用标准，下面介绍一些进阶的用法。

## 1.客户端tcp连接超时

在我们创建客户端时，实际上创建的是`RestClient`，而底层使用的是`apache`的`HttpClient`，在创建后长时间无操作时这个连接可能会被关闭，此时客户端并不知晓，直接使用就会提示下文中的错误。再次请求又是正常的，因为客户端会重新创建连接。

```Java
java.net.SocketTimeoutException: 30,000 milliseconds timeout on connection http-outgoing-6 [ACTIVE]
  at org.elasticsearch.client.RestClient.extractAndWrapCause(RestClient.java:915)
  at org.elasticsearch.client.RestClient.performRequest(RestClient.java:300)
  at org.elasticsearch.client.RestClient.performRequest(RestClient.java:288)
  at co.elastic.clients.transport.rest_client.RestClientTransport.performRequest(RestClientTransport.java:147)
  at co.elastic.clients.elasticsearch.ElasticsearchClient.search(ElasticsearchClient.java:1833)
Caused by: java.net.SocketTimeoutException: 30,000 milliseconds timeout on connection http-outgoing-6 [ACTIVE]
  at org.apache.http.nio.protocol.HttpAsyncRequestExecutor.timeout(HttpAsyncRequestExecutor.java:381)
  at org.apache.http.impl.nio.client.InternalIODispatch.onTimeout(InternalIODispatch.java:92)
  at org.apache.http.impl.nio.client.InternalIODispatch.onTimeout(InternalIODispatch.java:39)
  at org.apache.http.impl.nio.reactor.AbstractIODispatch.timeout(AbstractIODispatch.java:175)
  at org.apache.http.impl.nio.reactor.BaseIOReactor.sessionTimedOut(BaseIOReactor.java:263)
  at org.apache.http.impl.nio.reactor.AbstractIOReactor.timeoutCheck(AbstractIOReactor.java:492)
  at org.apache.http.impl.nio.reactor.BaseIOReactor.validate(BaseIOReactor.java:213)
  at org.apache.http.impl.nio.reactor.AbstractIOReactor.execute(AbstractIOReactor.java:280)
  at org.apache.http.impl.nio.reactor.BaseIOReactor.execute(BaseIOReactor.java:104)
  at org.apache.http.impl.nio.reactor.AbstractMultiworkerIOReactor$Worker.run(AbstractMultiworkerIOReactor.java:588)
  at java.lang.Thread.run(Thread.java:748)
```

这个问题在Github的官方Issues上面也有相关的讨论：[https://github.com/elastic/elasticsearch/issues/65213](https://github.com/elastic/elasticsearch/issues/65213)

解决问题的关键在于让连接能够保持`tcp keepalive`，有两种方案：

方案一：在客户端中显式的开启`keepalive`选项

```Java
RestClient httpClient = RestClient.builder(new HttpHost(hostName, port))
    .setHttpClientConfigCallback(hc -> hc
        .setDefaultIOReactorConfig(IOReactorConfig.custom().setSoKeepAlive(true).build())
     ).build();
```

另外，还需要设置系统层面的`tcp keepalive`探测时间，默认值7200s太长可能会被主动关闭，建议修改为300s，默认配置如下：

```Bash
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
```

方案二：在客户端设置`keepalive`策略，在超过指定时间后由客户端自行关闭，使用时再重新创建

```Java
RestClient httpClient = RestClient.builder(new HttpHost(hostName, port))
    .setHttpClientConfigCallback(hc -> hc
            .setKeepAliveStrategy((response, context) -> Duration.ofMinutes(5).toMillis()))
    .build();
```

## 2.聚合统计

在Elasticsearch中Aggregation 分为3种类型：

- Metric: 计算类型，对字段进行计算平均值、求和等；
- Bucket: 分组统计，根据字段或者范围将文档分组到桶中进行统计；
- Pipeline：对聚合结果再次进行聚合统计；

在分组统计中有2个参数需要特别关注：Size、Shard size。官方文档：[https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html)

Size：在使用`terms`对字段进行分桶时，默认值返回top 10文档，即只有10个统计结果，通过设置size的大小可以返回所需大小，最大值不超过` search.max_buckets `。
```Java
MultiTermsAggregation aggregation = MultiTermsAggregation.of(s -> s.terms(
    MultiTermLookup.of(t->t.field("product")),
    MultiTermLookup.of(t->t.field("user"))
).size(100));

```
Shard size：在上一篇文章中，我们介绍过Elasticsearch的查询过程，需要从每个分片获取结果后，再由协调节点进行合并排序。由于数据分布不均匀的缘故，如果每个分片只获取`size`大小的文档，可能会出现统计偏差。

Elasticsearch的解决方案是获取比所需更多的文档，在一定程度上避免这个问题，也就是`Shard size`参数的用途。默认值：`Shard size = size * 1.5 + 10`，在数据偏斜严重的情况下，可以适当调大这个参数，当然也意味着更多的性能损耗。

## 3.数据快照

之前我们介绍过`search_after` 能够实现深度分页功能，而在一些大批量数据导出的场景下，通常需要保持数据游标不变来导出完整的数据，类似于快照的功能。而这就需要用到 `point in time (PIT) `。

```Java
//获取pit id
OpenPointInTimeRequest openRequest = OpenPointInTimeRequest.of(o -> o
    .index(getIndex())
    .keepAlive(Time.of(t->t.time("10m"))));
OpenPointInTimeResponse openResponse = elasticsearchClient.openPointInTime(openRequest);

//查询数据
SearchRequest searchRequest = new SearchRequest.Builder()
    .size(pageSize)
    .sort(sortOptions)
    .pit(p -> p.id(params.getPit()));
    .build();
elasticsearchClient.search(searchRequest);    
    
//关闭pit
ClosePointInTimeRequest closeRequest = ClosePointInTimeRequest.of(c -> c.id(pit));
elasticsearchClient.closePointInTime(closeRequest);

```

## 4.获取搜索结果数量

在默认情况下`search`接口返回的`hits size`最大值是10000，如果需要获取实际的结果总数，需要开启`TrackHits`

```Java
SearchRequest searchRequest = new SearchRequest.Builder()
    .trackTotalHits(TrackHits.of(t->t.enabled(true)));
    .build();
elasticsearchClient.search(searchRequest);    
```

## 5.并发写入

一般情况下，Elasticsearch数据的写入会通过mq来进行触发，理论上可以通过mq的有序性来控制并发写入导致的数据覆盖问题，现实情况中考虑到性能、可靠性，较少采用这种方式。

方案一：增加version数据版本字段，通过CAS操作来实现乐观锁；

方案二：使用分布式锁，确保同一时间单个文档只有一个线程在执行更新操作，重试操作可以由mq来实现；

## 6.数据库事务

假设你正在使用Elasticsearch存储订单数据，在业务代码中的执行步骤如下：

- 更新MySQL中订单表数据；
- 发送订单变更的mq通知；
- 消费mq消息，从MySQL读取最新的数据写入Elasticsearch；

在运行一段时间后，你可能会发现Elasticsearch的数据与MySQL不一致，不是最新的版本；仔细分析上述过程会发现一个问题，在执行第2步操作时第1步的数据事务还没提交完成，将导致第3读取的不是最新数据。提供一种解决问题的思路，在事务提交完成后再发送mq消息。

```Java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter(){
    @Override
    public void afterCommit() {
        //发送mq
    }
});
```
今天就先写到这里，你学"废"了吗。