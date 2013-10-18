---
layout: post
comments: true
title: Setting the BooleanQuery maxClauseCount in Elasticsearch
tags:
- elasticsearch
- lucene
type: post
---

If you have come across the [TooManyClauses](http://lucene.apache.org/core/4_5_0/core/org/apache/lucene/search/BooleanQuery.TooManyClauses.html) exception while querying Elasticsearch, chances are you are using a [terms query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html), [prefix query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html), [fuzzy query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html), [wildcard query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html) or [range query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) that ends up expanding into more than 1024 boolean clauses. That's the default setting in Lucene's [BooleanQuery](http://lucene.apache.org/core/3_0_3/api/core/org/apache/lucene/search/BooleanQuery.html#getMaxClauseCount(\)) which is what lies underneath.

That setting is there as a sensible default for memory conservation and performance reasons so unless you know what you are doing you should probably avoid changing it. The [Lucene FAQ](http://wiki.apache.org/lucene-java/LuceneFAQ#Why_am_I_getting_a_TooManyClauses_exception.3F) mentions a few approaches to overcoming the [TooManyClauses](http://lucene.apache.org/core/4_5_0/core/org/apache/lucene/search/BooleanQuery.TooManyClauses.html) exception which apply to Elasticsearch as well. One of them is to replace the query that causes the exception by a filter which is cacheable and much more efficient. In Elasticsearch, most the the above queries have filter equivalents:

* [terms query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html) => [terms filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-terms-filter.html)
* [prefix query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html) => [prefix filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-prefix-filter.html)
* [fuzzy query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html) => N/A
* [wildcard query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html) => N/A
* [range query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) => [range filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-range-filter.html)

Having said that, if you really need to use a query instead of a filter (e.g. you need your results sorted by relevance), then you can bump the `maxClauseCount` by setting the following in the Elasticsearch config file:

```yaml
[...]
index.query.bool.max_clause_count: 4096
[...]
```

Note that since this is a [static Lucene setting](https://groups.google.com/d/msg/elasticsearch/LqywKHKWbeI/KbxgZnPH6WoJ), it can only be set in the Elasticsearch config file and get picked up at startup.

## Update
[@imotov](https://twitter.com/imotov) (who pointed out `index.query.bool.max_clause_count` to me in the first place) also suggests looking into the `rewrite` parameter if you are using a [multi term query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-multi-term-rewrite.html) as an additional source of options for controlling how boolean queries are re-written in Elasticsearch.

