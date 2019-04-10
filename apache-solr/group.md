# Group

## 分组查询

group 是对查询结果按一个字段分组，并返回各组的前排文档。

示例

`在一家电子产品在线商店搜索“DVD”，你可能会获得3个类别：“TV and Video”， “Movies”， “Computers”， 每类3个结果。   
在这个案例里，查询关键字“DVD”在3个类别里都有出现，所以 solr 将他们分组，提升用户体验`

## group 参数

group.field 必须满足如下条件\(存疑\)

* 单值\(single-valued\)
* 索引\(indexed\)或 value source 且在一个 function query 里，例如 ExternalFileField
* string 类型，如 StrField 或 TextField

| 参数 | 类型 | 说明 |
| :--- | :--- | :--- |
| group | boolean | 是否开启 group 查询 |
| group.field | string | 分组字段 |
| group.func | query | 分组基于一个 function query 的唯一值 |
| group.query | query | 该查询返回一组匹配的文档 |
| rows | int | 返回几个组，默认=10 |
| start | int | 对于组的列表，指定一个初始的偏移 |
| group.limit | int | 每组返回多少结果，默认=1 |
| group.offset | int | 针对每组里的文档集合，指定一个初始的偏移量 |
| sort | sortspec | solr 如何对组排序，默认=score desc。 |
| group.sort | sortspec | solr 如何对每个组的文档排序 |
| group.format | 取值=\[grouped,simple\] | simple 表示返回一个简单列表，且 start，rows参数对文档起作用而非组 |
| group.main | boolean | true 表示首个字段的分组结果为主要结果，返回格式为 simple |
| group.ngroups | boolean | 默认=false，true 表示返回结果带上组数 |
| group.truncate | boolean | 默认=false，true 表示 facet count 基于每组里最相关的文档 |
| group.facet | boolean | 是否计算分组 facet |
| group.cache.percent | int，取值\[0-100\] | 默认=0，大于 0 表示开启结果缓存。注意：开启缓存仅对 boolean，通配符，模糊查询有改善，对其他查询会降低性能 |

## group query

不指定分组字段，而是指定了分组条件，例如将查询结果按如下条件分组

* price&lt;100
* price&lt;=500 and price&gt;=100
* price&gt;500

那么查询可以这么写

```bash
group=true&
group.query=price:[0 TO 99.99]&
group.query=price:[100 TO 500]&
group.query=price:[500.01 TO *]
```

## 分布式分组查询

group 支持分布式查询，但要注意

* group.func 不支持分布式
* group.ngroups 和 group.facet 需要每个组的所有文档都在同一个分片

