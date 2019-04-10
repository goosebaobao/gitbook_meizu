# Faceting

{% hint style="info" %}
facet 字段必须索引
{% endhint %}

## 通用参数

* facet：true 表示启用 faceting
* facet.query：lucene 默认语法的查询

### **字段值参数**

一些参数可以用来触发 faceting，基于索引的字段 使用这些参数时，要记住 term 是 lucene 的一个特殊概念。

* facet.field
  * 将这个字段作为 facet
* facet.prefix
  * 限定那些以此为前缀的 term
* facet.contains
  * 限定那些包含此字符串的 term
* facet.contains.ignoreCase
  * 如果 facet.contains 被使用，是否忽略大小写
* facet.sort
  * 控制结果如何排序，可接受的值为 `true(count)|false(index,lex)`.默认情况下为 true\(count\).当facet.limit值为负数时,默认 `facet.sort = false(index,lex)`
    * true\(count\) 表示按照 count 值从大到小排列. 
    * false\(index,lex\) 表示按照字段值的自然顺序\(字母,数字的顺序\)排列
* facet.limit
  * 限制 Facet 字段返回的结果条数.默认值为 100.如果此值为负数,表示不限制
* facet.offset
  * 返回结果集的偏移量,默认为 0.它与 facet.limit 配合使用可以达到分页的效果
* facet.mincount
  * 限制了 Facet 字段值的最小 count,默认为 0.合理设置该参数可以将用户的关注点集中在少数比较热门的领域.
* facet.missing
  * solr 是否应计算出总数：在所有匹配结果中该字段没有值
* facet.method
  * 取值为 enum 或 fc,默认为 fc.该字段表示了两种 Facet 的算法,与执行效率相关.
    * enum 适用于字段值比较少的情况,比如字段类型为布尔型,或者字段表示中国的所有省份.Solr 会遍历该字段的所有取值,并从 filterCache 里为每个值分配一个 filter\(这里要求 solrconfig.xml 里对 filterCache 的设置足够大\).然后计算每个 filter 与主查询的交集.
    * fc \(表示 Field Cache\)适用于字段取值比较多,但在每个文档里出现次数比较少的情况.Solr 会遍历所有的文档,在每个文档内搜索 Cache 内的值,如果找到就将 Cache 内该值的 count 加 1.
* facet.enum.cache.minDF
  * 当 facet.method=enum 时,此参数起作用, minDf 表示 minimum document frequency.也就是文档内出现某个关键字的最少次数.该参数默认值为 0.设置该参数可以减少 filterCache 的内存消耗,但会增加总的查询时间\(计算交集的时间增加了\).如果设置该值的话,官方文档建议优先尝试 25-50 内的值.
* facet.overrequest.count
  * \(Advanced\) A number of documents, beyond the effective facet.limit to request from each shard in a distributed search
* facet.overrequest.ratio
  * \(Advanced\) A multiplier of the effective facet.limit to request from each shard in a distributed search
* facet.threads
  * \(Advanced\) Controls parallel execution of field faceting

