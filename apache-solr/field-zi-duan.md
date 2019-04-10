# Field 字段

## fieldType

定义了 solr 如何理解 field 的数据，及 field 如何被查询；solr 默认包含了许多 field type，也可以自定义 field type

field type 定义包括4类信息

* 名称，必须的
* 实现类名，必须的
* 如果是 TextField，field analysis（分析器）
* field type 属性，有哪些属性是由实现类决定的，某些属性可能是必须的

## 定义 field type

通过 schema.xml 定义 fieldType，每个 field type 的定义都位于 `fieldType` 元素，在 `types` 元素内，示例如下

```markup
<fieldType name="text_general" class="solr.TextField" positionIncrementGap="100">
    <analyzer type="index">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt"/>
        <!-- in this example, we will only use synonyms at query time
        <filter class="solr.SynonymFilterFactory" ynonyms="index_synonyms.txt" ignoreCase="true" expand="false"/>
        -->
        <filter class="solr.LowerCaseFilterFactory"/>
    </analyzer>
    <analyzer type="query">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt"/>
        <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
        <filter class="solr.LowerCaseFilterFactory"/>
    </analyzer>
</fieldType>
```

## fieldType 属性

* name
  * 类型名称，用于 field 元素。名称使用字母、数字、下划线组成，且首字母不能位数字
* class
  * 存储和索引该类型数据的类。以 solr 为前缀，实际上 solr 是包名 org.apache.solr.schema 或 org.apache.solr.analysis 的缩写，solr 会自动识别是哪个包名
* positionIncrementGap
  * 用于多值\(multivalued\)字段，指定多个值之间的距离，以防止虚假的短语匹配
  * 取值为 integer
* autoGeneratePhraseQueries
  * 用于 text 字段，如果为 true，solr 将相邻的词识别为短语，如果为 false，只有用双引号包围的词才会被识别为短语
  * 取值为boolean
* docValuesFormat
  * Defines a custom DocValuesFormat to use for fields of this type.This requires that a schema-aware codec, such as the SchemaCodecFactory has been configured in olrconfig.xml.
* postingsFormat
  * Defines a custom PostingsFormat to use for fields of this type.This requires that a schema-aware codec, such as the SchemaCodecFactory has been configured in olrconfig.xml.
* sortMissingLast
* omitNorms

## solr 包含哪些 fieldType

所有内置的 fieldType 都在包 org.apache.solr.schema 里

* BinaryField
  * 二进制数据
* BoolField
  * 取值为true/false。首字母为1，T，t作为true，其他的是false
* CollationField
  * 排序和范围查询时支持 Unicode 的排序规则。如果使用 ICU4J，那么 ICUCollationField 更好
* CurrencyField
  * 金额或汇率
* DateRangeField
  * 支持日期范围的索引
* ExternalFileField
  * 从磁盘上的文件拉取数据
* EnumField
  * 定义一个枚举值列表，枚举值按顺序定义在一个配置文件中，该字段可以给一些难以排序的值做排序
* ICUCollationField
  * 支持 Unicode 排序规则；参考 CollationFiled
* LatLonType
  * 一对经纬度坐标，第一个坐标是维度
  * latitude：维度；longitude：经度
* PointType
  * 任意维度的点
* PreAnalyzedField
  * 
* RandomSortField
  * 不包含值\(?\)查询如果在这个字段上做排序将随机排列。使用动态字段来使用这个特性
* SpatialRecursivePrefixTreeFieldType
  * 维度+逗号+经度字符串或其他的 WKT 格式
* StrField
  * 字符串\(UTF8 编码或 Unicode\)
* TextField
  * 文本，通常是多个词或标记
* TrieDateField
  * 日期
  * precisionStep="0" 可按日期排序，索引最小
  * precisionStep="8" 默认值，可范围查询
* TrieDoubleField
  * double，8字节
  * precisionStep="0" 可按数字排序，索引最小
  * precisionStep="8" 默认值，可范围查询
* TrieField
  * 这个类型必须指定 type 属性，可选的值为 \[integer, long, float, double, date\]。等效于对应的 Trie 字段
  * precisionStep="0" 可按数字排序，索引最小
  * precisionStep="8" 默认值，可范围查询
* TrieFloatField
  * float，4字节
  * precisionStep="0" 可按数字排序，索引最小
  * precisionStep="8" 默认值，可范围查询
* TrieIntField
  * int，4字节
  * precisionStep="0" 可按数字排序，索引最小
  * precisionStep="8" 默认值，可范围查询
* TrieLongField
  * long，8字节
  * precisionStep="0" 可按数字排序，索引最小
  * precisionStep="8" 默认值，可范围查询
* UUIDField
  * 通用唯一标识符，传入“NEW”，solr会创建一个 UUID，cloud 模式不建议这样使用

## field 属性用例

| 用例 | indexed | stored | multiValued | omitNorms | termVectors | termPositions | docValues |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 查询 | true |  |  |  |  |  |  |
| 取值 |  | true |  |  |  |  |  |
| 唯一 | true |  | false |  |  |  |  |
| 排序 | true\(7\) |  | false | true\(1\) |  |  | true\(7\) |
| boost\(5\) |  |  |  | false |  |  |  |
| 文档boost |  |  |  |  | false |  |  |
| 高亮 | true\(4\) | true |  |  | true\(2\) | true\(3\) |  |
| faceting\(5\) | true\(7\) |  |  |  |  |  | true\(7\) |
| 多值 |  |  | true |  |  |  |  |
| 字段长度影响文档得分 |  |  |  | false |  |  |  |
| Morelikethis\(5\) |  |  |  |  | true\(6\) |  |  |

* 1：推荐，但不是必须的
* 2：如果提供了就会使用，非必须
* 3：if termVectors=true
* 4：该字段必须定义 tokenizer，但不必是 indexed 的
* 5
* 6：Term vectors 不是强制的。如果不是 true，那么一个 stored 字段被分析。所以，term vector 是建议的，但仅当 stored=false 时才是必须的
* 7：indexed 和 docValues 都必须是 true，但都不是必须的。DocValues 在很多场景更有效

## 定义 field

field 属性

* name
  * 名字，字母、数字、下划线组成，且首字母不能为数字，每个 field 都必须有 name 属性
  * 以下划线开头和结尾的 name 是为 solr 保留的，例如： version 
* type
  * fieldType 的名字，每个 field 都必须有 type 属性
* default
  * 默认值，任何文档里如果这个字段没有值，将会自动使用默认值作为该字段的值，非必须的

## field 默认属性

既可以用在 filedType，也可以用在 field \(覆盖 fieldType 里的值\)的属性

* indexed
  * true
  * 是否索引
* stored
  * true
  * 是否存储
* docValues
  * false
  * 是否将字段值放到面向列的DocValues结构里，若为否则是面向行的存储
* sortMissingFirst/sortMissingLast
  * false
  * 控制文档的位置，如果文档没有该字段；即有该字段的文档是否在没有该字段的文档之前
* multiValued
  * false
  * 单个文档是否有多个该字段
* omitNorms
  * 对原始数据类型（int/float/date/bool/string）为 true
  * 是否省略 norms
* omitTermFreqAndPositions
  * 对所有非 text 类型都是 true
  * 
* omitPositions
  * * 和 omitTermFreqAndPositions 类似，但是不包含 term frequency 信息
* termVectors/termPositions/termOffsets/termPayloads
  * false
  * 
* required
  * false
  * 字段值是否必须的，如果为 true，solr 将不会保存该字段值为空的文档
* useDocValuesAsStored
  * true
  * 如果该字段的 docValues 为 true，那么将该值设为 true 将允许该字段在查询条件匹配 fl=\* 时，如同一个 stored 字段（即使 stored=false）一样返回

## Copying Fields

```markup
<!-- maxChars：从 source 最多复制多少字符到 dest -->
<copyField source="cat" dest="text" maxChars="30000" />
```

如果 dest 所指向的 field 将会接收多个字段，其 multivalued 属性应为 true

