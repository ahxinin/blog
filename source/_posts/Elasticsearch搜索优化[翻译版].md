---
title: Elasticsearch写入速度优化[翻译版]
date: 2023-05-16
---

在上一篇文章中翻译了索引写入，接下来看看索引的查询有哪些优化手段。

原文链接：[https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html)

### 1.增加文件系统缓存

Elasticsearch严重依赖文件系统缓存来加快查询速度。一般来说，至少需要保留一半的可用内存给文件系统，以便Elasticsearch在物理内存中保留索引热点数据。

### 2.使用更快的硬件

如果搜索遇到了I/O瓶颈，考虑增加文件系统缓存或者使用更快的存储设备。每次查询涉及随机读和顺序读的混合操作，跨越多个文件，而且每个分片上可能有多个搜索的并发请求，因SSD磁盘比普通硬盘性能更佳。

本地磁盘比网络云盘更加高效，因为配置更加简单而且可以避免频繁的网络通信。经过优化配置，网络云盘有时也能达到所期望的性能。通过真实负载下的基准测试，来确定优化参数是否生效。如果不能达到你所期望的效果，可以联系你的存储供应商来解决问题。

如果你的搜索遇到的是CPU瓶颈，可以考虑增加速度更快的CPU。

### 3.文档结构化

文档结构越简单搜索成本越低。

在实际使用中，应当避免文档关联。嵌套结构`nested`可能导致查询速度慢上好几倍，父子文档`parent-child`关联的查询速度可能要慢几百倍。如果可以使用非规范化文档来规避关联关系，查询速度可以得到显著的提升。

### 4.只搜索必要的字段

当`query_string` 或者 `multi_match` 包含的字段越多时，搜索速度就会越慢。改善多字段查询速度的通用技巧，是在写入文档时将多个字段的内容复制到一个字段中，在搜索搜索时只需要查询这个字段。使用文档结构的`copy-to` 指令就可以自动完成，无需修改源文档。下面的例子关于如何改善电影索引的搜索性能，把需要查询的电影名称、情节都放到`name_and_plot`字段中。

```SQL
PUT movies
{
  "mappings": {
    "properties": {
      "name_and_plot": {
        "type": "text"
      },
      "name": {
        "type": "text",
        "copy_to": "name_and_plot"
      },
      "plot": {
        "type": "text",
        "copy_to": "name_and_plot"
      }
    }
  }
}
```



### 5.预索引数据

在搜索和数据存储优化方面需要取得一个平衡。比如说，如果你所有的文档都有一个 `price`字段，大多数查询使用了固定范围内的`range` 聚合查询，你可以将 `terms`聚合查询的结果存储到新的索引中，来加快查询效率。

例如，文档如下所示：

```SQL
PUT index/_doc/1
{
  "designation": "spoon",
  "price": 13
}
```

搜索请求：

```SQL
GET index/_search
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 10 },
          { "from": 10, "to": 100 },
          { "from": 100 }
        ]
      }
    }
  }
}
```

文档在写入时可以填充到`price_range`字段中，使用`keyword`字段类型：

```SQL
PUT index
{
  "mappings": {
    "properties": {
      "price_range": {
        "type": "keyword"
      }
    }
  }
}

PUT index/_doc/1
{
  "designation": "spoon",
  "price": 13,
  "price_range": "10-100"
}
```

然后在搜索时，可以对新的字段进行聚合来代替对`price`字段进行聚合：

```SQL
GET index/_search
{
  "aggs": {
    "price_ranges": {
      "terms": {
        "field": "price_range"
      }
    }
  }
}
```

### 6.唯一标记使用 keword字段类型

不是所有的数字都应该使用 `numeric` 字段类型。Elasticsearch对数字类型进行了优化，例如`integer` 或者 `long`，在`range`查询场景下。然而，`keyword`类型在`term` 和 `term-level`查询中表现更好。

唯一标记，例如ISBN或者产品id，很少使用范围查询，却经常使用`term-level`查询。

以下情况可以考虑将数字类型的唯一标记存储为keyword类型：

- 唯一标记不会用于范围查询；
- 更看重搜索性能。keyword字段类型上的term查询比数字类型要快许多；

如果你不确定使用哪种方式，可以使用`multi-field`来同存储keywrod和数字类型。

### 7.避免使用脚本

如果可能，避免使用基于脚本的排序、聚合，以及用脚本计算评分。

### 8.搜索近似时间

使用`now`条件来搜索时间字段通常没有缓存，因为匹配的条件一直在变化。然而使用近似时间在条件查询中经常上适用的，而且可以更好的利用查询缓存。

例如下面的查询：


```SQL
PUT index/_doc/1
{
  "my_date": "2016-05-11T16:30:55.328Z"
}

GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "my_date": {
            "gte": "now-1h",
            "lte": "now"
          }
        }
      }
    }
  }
}
```

可以进行如下替换：

```SQL
GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "my_date": {
            "gte": "now-1h/m",
            "lte": "now/m"
          }
        }
      }
    }
  }
}
```

在这个例子中我们使用了分钟近似值，如果当前时间是`16:31:29` ，`my_date`字段的范围查询将返回所有从`15:31:00`到`16:31:59`时间段内的数据。如果同一时间好几个用户的查询条件包含这个范围，查询缓存能够加快查询速度。近似查询的范围越长，缓存的效果越明显，但是需要注意过度的近似值可能会破坏用户体验。

### 9.强制合并只读索引

强制合并为一个段对只读索引来说是有益的。在时间线索引中比较常见的场景：只有当前时间的索引会新增数据，历史索引是只读的。分片被强制合并为一个段，可以让查询更加简单和有效。

### 10.预热全局序号

全局序号是用来优化聚合的一种数据结构。他们作为字段缓存的一部分会在`JVM`中延迟计算和存储。作为分桶查询中被频繁使用的字段，你可以让`Elasticsearch`在请求到达前实例化和缓存。这个操作应该谨慎使用，因为他会占用更多内存使得`refresh`变长。这个选项可以在已经创建的索引上动态设置，通过修改`eager global ordinals` 参数：

```SQL
PUT index
{
  "mappings": {
    "properties": {
      "foo": {
        "type": "keyword",
        "eager_global_ordinals": true
      }
    }
  }
}
```

### 11.预热文件系统缓存

如果运行`Elasticsearch`的机器重启了，文件系统缓存会被清空，因此操作系统需要花费一些时间来加载热点索引缓存数据到内存，以便加快查询速度。你可以明确的告诉操作系统哪些文件需要提前加载到缓存中，通过`index.store.preload` 参数来进行指定。

### 12.使用索引排序来加快连接速度

索引排序在加快连接速度方面很有效，代价是文档写入会变慢。

### 13.使用preference来优化缓存使用

有多种缓存可以用来加快查询速度，诸如文件系统缓存、请求缓存、查询缓存。这些缓存大多是在节点层面的，意味着如果你连续发起2次相同的请求，有一个或者多个副本而且所有了负载策略，根据默认的路由算法，2次请求会分配到不同的分片节点，节点层面的缓存无法有效利用。

由于搜索程序的用户会一个接着一个的发起类似的查询请求，比如说为了分析索引索引的子集，使用偏好值来标记当前用户或者请求能够帮助优化缓存的使用。

### 14.副本或许可以提升吞吐量

除了弹性扩展，副本还能提升吞吐量。例如你有一个单一分片索引和三个节点，你需要将副本数量设置为2，这样总共3份副本让每个节点都能充分利用。

现在假设你有2个分片索引和2个节点。第一种情况，副本数量设置为0，意味着每个节点有一个分片。第二种情况副本数量设置为1，意味着每个节点有2个分片。哪种情况能够有更好的查询性能呢？通常情况下，节点的分片数量越少的方案更优。因为能够给每个分片更多的文件系统缓存，而文件系统缓存可能是`Elasticsearch`最有效的优化策略。与此同时，需要注意物副本的方案在单节点失败情况下的风险，需要权衡吞吐量和可用性。

因此分片数量设置为多少比较合适？如果你的集群有`num_nodes`个节点，总共有`num_primaries`个主分片，你期望同时可以应对最多`max_failures`个节点失败的状况，正确的副本数量结算方式为：`max(max_failures, ceil(num_nodes / num_primaries) - 1)`

### 15.使用Search Profiler优化查询

`Profile API`提供了查询和聚合在每一步处理耗时的详细信息。

在`Kibana`上使用` Search Profiler`可以清楚直观的看到分析结果，以及如何优化查询和减轻负载压力。

因为`Profile API`在查询上增加了大量开销，返回的结果适用于了解各个查询阶段的相对耗时。不代表实际的处理时间。

### 16.使用index_phrases加快短语查询

`text`字段有一个`index_phrases`选项来索引2-shingles，能够被短语查询自动应用，在没有`slop`的情况下。如果你的案例中有大量的短语查询，可以显著的加快查询速度。

### 17.使用index_prefixes来加快前缀查询

text字段有一个index_prefixes选项来索引前缀，在前缀查询条件中能够被自动的应用。如果你的案例中有大量的前缀查询，可以显著的加快查询速度。

### 18.使用constant_keyword来加快过滤速度

有一个通用规则，过滤查询的耗时基本上是匹配文档数量的一个函数。假设你有一个包含骑行的索引。有大量的自行车数据，许多查询上基于过滤条件：`cycle_type: bicycle`。这个常见的过滤会很耗时，因为匹配到了大量的文档。有一个简单的方法来避免运行此类查询，把自行车移动他自己的索引中，查询这个索引来代替过滤查询。

不幸的是这样会使客户端的逻辑变得复杂，而这正是`constant_keyword`发挥作用的地方。通过在bicycles索引上将`cycle_type`字段类型设置为`constant_keyword`，值设置为`bicycle`，客户端依然可以运行和之前单片索引上一样的查询语句，Elasticsearch会在bicycles索引上忽略条件为`cycle_type`并且值为`bicycle`的过滤条件，返回正确的结果。

索引结构如下所示：

```SQL
PUT bicycles
{
  "mappings": {
    "properties": {
      "cycle_type": {
        "type": "constant_keyword",
        "value": "bicycle"
      },
      "name": {
        "type": "text"
      }
    }
  }
}

PUT other_cycles
{
  "mappings": {
    "properties": {
      "cycle_type": {
        "type": "keyword"
      },
      "name": {
        "type": "text"
      }
    }
  }
}
```

我们将索引一分为二，一个仅包含自行车，一辆一个包含其他车辆：独轮车、三轮车等等。在查询时，我们需要查询所有的索引，但是不需要修改查询语句。

```SQL
GET bicycles,other_cycles/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "description": "dutch"
        }
      },
      "filter": {
        "term": {
          "cycle_type": "bicycle"
        }
      }
    }
  }
}
```

在`bicycles`索引中，Elasticsearch会忽略`cycle_type`过滤条件将查询请求重写如下：

```SQL
GET bicycles,other_cycles/_search
{
  "query": {
    "match": {
      "description": "dutch"
    }
  }
}
```

在`other_cycles`索引，Elasticsearch会快速的发现在`cycle_type`中不包含`bicycle`，返回无匹配结果。

这是一个非常有效的手段，将通用的查询字符内容放在专属索引里面。这个方法也适用于多字段，例如你需要追踪每个骑行工具的颜色，你的`bicycles`索引中大部分都是黑色的自行车，你可以将他分为`bicycles-black`和`bicycles-other-colors`索引。

`constant_keyword`不属于严格意义上的索引优化：更像是将客户端的逻辑查询路由到指定的索引上。但是`constant_keyword`使其透明化，将查询和索引结果解耦来优化性能。