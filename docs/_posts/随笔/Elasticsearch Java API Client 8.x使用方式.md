---
title: 高性能MySQL的实现策略 
date: 2023-03-31 22:43:00
permalink: /pages/3fe7f8/
sidebar: auto
categories:
  - 随笔
tags:
  - 
author: 
  name: hexin
  link: https://github.com/ahxinin
---

### 客户端的变化
众所周知，Elasticsearch是基于Lucene的，提供了更高层次的封装、分布式方面的扩展，以及REST API来方便使用，我们先来看看java client的变化：

![](/img_convert/2.png)
从图中可以看成，在8.x版本中，Elasticsearch提供了全新的Java API Client，用来代替之前广为使用的High Level Client，根据官网说法两者并无关联；而更具有灵活性和偏向底层的Low Level Client依旧在迭代，提供给用户更多的选择。

### 快速开始
话不多说，直接开始，Java API Client依赖于JSON来进行数据格式化，支持Jackson或者JSON-B库，引入相应maven依赖。

``` 
<dependency>
  <groupId>co.elastic.clients</groupId>
	<artifactId>elasticsearch-java</artifactId>
	<version>8.6.2</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.12.3</version>
</dependency>

<dependency>
  <groupId>jakarta.json</groupId>
  <artifactId>jakarta.json-api</artifactId>
  <version>2.0.1</version>
</dependency>
```

下一步连接Elasticsearch服务端：

```
@Component
public class ElasticsearchConfig {

    @Bean
    public ElasticsearchClient elasticsearchClient(){
        BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(userName, password));

        RestClient httpClient = RestClient.builder(new HttpHost(hostName, port))
            .setHttpClientConfigCallback(hc -> hc.setDefaultCredentialsProvider(credentialsProvider))
            .build();

        ElasticsearchTransport transport = new RestClientTransport(httpClient, new JacksonJsonpMapper());
        return new ElasticsearchClient(transport);
    }
}
```
在这里创建的是一个同步的客户端，Java API Client还支持创建异步的客户端：ElasticsearchAsyncClient，返回的是一个标准的 CompletableFuture，按需选择。另外我们还可以看到，创建对象时使用了构造器模式，以及lambda表达式，这两种方式使得代码更加简洁和高效。

### JakartaEE
补充一点额外内容，JSON-B库的全称是Jakarta JSON Binding，是用于Java对象与JSON消息相互转换的标准绑定层，来源于Jakarta EE。
而Jakarta EE并不是什么新鲜技术，它的前身是Java EE。之所以改名称，是因为2017年Oracle宣布开源Java EE并将项目移交给Eclipse基金会时，提出来的要求，导致改名事件。现阶段一般都在使用类似于SpringBoot的框架，Java EE的存在感就更弱了，这里就不再扩展了。

### 写入文档
Java API Client支持写入bean对象，或者直接写入JSON格式数据，其中bean对象会被自动映射为JSON。

```
public void index() throws IOException {
    Order order = new Order(1L, "test product", 233L);

    IndexRequest<Order> request = IndexRequest.of(i -> i
        .index("order-index")
        .id(String.valueOf(order.getId()))
        .document(order)
        .version(1L)
    );

    IndexResponse indexResponse = elasticsearchClient.index(request);
    log.info("indexResponse:{}", indexResponse.toString());
}
```
构建一个order对象后，使用其ID作为主键，写入到名为"order-index"的索引之中，同时通过version参数指定了数据版本号，来进行并发控制。

### 搜索文档
以产品名称、价格作为搜索条件，来看看具体的实现：

```
public void search() throws IOException {
    String keyword = "apple";
    Long maxPrice = 100L;

    Query byProduct = MatchQuery.of(m -> m
        .field("product")
        .query(keyword)
    )._toQuery();

    Query byMaxPrice = RangeQuery.of(r -> r
        .field("price")
        .gte(JsonData.of(maxPrice))
    )._toQuery();
    
    SortOptions sortOptions = SortOptions.of(s -> s
        .field(FieldSort.of(f->f
            .field("id")
            .order(SortOrder.Desc))
        )
    );

    SearchResponse<Order> response = elasticsearchClient.search(s -> s
        .index("order-index")
        .query(q -> q
            .bool(b -> b
                .must(byProduct)
                .must(byMaxPrice)
            )
        )
        .from(0)
        .size(10), 
        Order.class
    );

    List<Hit<Order>> hits = response.hits().hits();
    hits.forEach(hit -> {
        Order order = hit.source();
        List<FieldValue> sortList = hit.sort();
    });
}
```
### 深度分页
在搜索文档的实现中，指定了from和size参数，目的的为了实现分页效果；其中from是游标位置，size是返回数据量大小，类似于MySQL的limit功能。当from过大时Elasticsearch会返回一个错误：
`Result window is too large, from + size must be less than or equal to: [10000]`

分析这个错误，需要从搜索的实现原理来入手。在客户端请求到达协调节点后，会从各个分片的数据节点获取数据，因为数据分布不均匀的关系，例如在查询第2页的10条数据时，需要从每个节点都获取20条数据，进行排序来避免数据遗漏。

![](/img_convert/3.png)

在之前的版本中，Elasticsearch经常会出现的一个问题是查询1W条以后的数据会非常慢，也就是因为这个原因，解决办法是使用Scroll API，使用快照的思路实现。在最新版中，这种方式已经不再推荐使用，取而待之的是search_after参数。

```
elasticsearchClient.search(s -> s
    .index("order-index")
    .searchAfter(FieldValue.of(1878133432))
    .sort(sortOptions)
    .size(20),
    Order.class
);
```
### 聚合功能
Elasticsearch中聚合常见的应用场景是实现类似于MySQL的sum、count、group by功能，来看一个多参数group by的实现。

```
public void aggregation() throws IOException {
    MultiTermsAggregation aggregation = MultiTermsAggregation.of(s -> s.terms(
        MultiTermLookup.of(t->t.field("product")),
        MultiTermLookup.of(t->t.field("user"))
    ));

    Aggregation priceAggregation = Aggregation.of(s -> s.sum(AggregationBuilders.sum().field("price").build()));
    Aggregation idAggregation = Aggregation.of(s -> s.valueCount(ValueCountAggregation.of(v -> v.field("id"))));

    Aggregation aggs = Aggregation.of(s -> s
        .multiTerms(aggregation)
        .aggregations("price", priceAggregation)
        .aggregations("id", idAggregation)
    );

    SearchRequest searchRequest = new SearchRequest.Builder()
        .index("order-index")
        .aggregations("aggs", aggs)
        .build();
    SearchResponse<Void> searchResponse = elasticsearchClient.search(searchRequest, Void.class);
    Aggregate aggregate = searchResponse.aggregations().get("aggs");
    Buckets<MultiTermsBucket> buckets = aggregate.multiTerms().buckets();
    buckets.array().forEach(bu -> {
        String product = bu.key().get(0).stringValue();
        String user = bu.key().get(1).stringValue();

        double totalPrice = bu.aggregations().get("price").sum().value();;
        double totalNum = bu.aggregations().get("id").valueCount().value();
    });
}
```



