# Analyzers, Tokenizers, and Filters

## 概述

solr 如何分解和处理文本数据？有 3 个主要的概念需要理解：analyzers，tokenizers，filters

* analyzers 用于索引和查询时，一个 Analyzer 检查文本并且生成词元\(token\)流，可以是单个 class 或者一系列 tokenizer 和 filters 组成
* tokenizers 将文本分解为词法单位
* filters 检查词元\(token\)流，保留、转换、丢弃或创建新的词。tokenizers 和 filters 可以组合成管道或者链，前一个的输出作为后一个的输入：这样的一个 tokenizers 和 filters 序列被称为 analyzer，其输出的结果用来匹配查询结果或者索引

## 分析器 Analyzers

分析器\(Analyzer\)检查文本内容并生成词元\(token\)流。在 schema.xml 文件的 fieldType 节点定义分析器

一般情况，只有 solr.TextField 需要指定 analyzer。配置 analyzer 最简单的方法是单个 analyzer 元素，示例

```markup
<fieldType name="nametext" class="solr.TextField">
    <analyzer class="org.apache.lucene.analysis.core.WhitespaceAnalyzer"/>
</fieldType>
```

上面是最简单的例子，但是通常需要更复杂的分析器。

大多数复杂的分析需求，通常是由一系列分离的、相对简单的步骤组成。solr 有大量可选的分词器\(tokenizers\)和过滤器\(filters\)能覆盖你可能遇到的大多数场景。

装配一个分析器链是很简单的：指定一个 analyzer 节点\(不需要 class 元素\)，带有 tokenizer 和 filter 子节点，按实际运行的顺序排列，示例

```markup
<fieldType name="nametext" class="solr.TextField">
    <analyzer>
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.StandardFilterFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
        <filter class="solr.StopFilterFactory"/>
        <filter class="solr.EnglishPorterFilterFactory"/>
    </analyzer>
</fieldType>
```

提示 

{% hint style="info" %}
org.apache.solr.analysis 包下的类，可以用 solr 为前缀简化书写
{% endhint %}

注意 

{% hint style="info" %}
分析器将字段内容进行分词处理后输出，只影响索引，不会对存储有影响。例如，“黑牛”也许会分词为“黑”，“牛”，但保存时还是保存为“黑牛”
{% endhint %}

## 分析的阶段

分析发生在 2 个阶段：索引和查询。很多情况下，相同的分析适用于 2 个阶段，某些情况下你也许希望索引和查询有不同的分析步骤，这样可以定义 2 个不同 type 的 analyzer 节点，示例

```markup
<fieldType name="nametext" class="solr.TextField">
    <analyzer type="index">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
        <filter class="solr.KeepWordFilterFactory" words="keepwords.txt"/>
        <filter class="solr.SynonymFilterFactory" synonyms="syns.txt"/>
    </analyzer>
    <analyzer type="query">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
    </analyzer>
</fieldType>
```

## Multi-Term Expansion

如果要按前缀，通配符，正则表达式进行查询，自然语言分析就无能为力了，同义词、停止词过滤器都不适用这种查询

这类查询，有效的分析器是 MultiTermAwareComponents，可以定义 type 为 multiterm 的分析器，示例

```markup
<fieldType name="nametext" class="solr.TextField">
    <analyzer type="index">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
        <filter class="solr.KeepWordFilterFactory" words="keepwords.txt"/>
        <filter class="solr.SynonymFilterFactory" synonyms="syns.txt"/>
    </analyzer>
    <analyzer type="query">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
    </analyzer>

    <!-- No analysis at all when doing queries that involved Multi-Term expansion -->
    <analyzer type="multiterm">
        <tokenizer class="solr.KeywordTokenizerFactory" />
    </analyzer>
</fieldType>
```

停止词 ：类似汉语的助词，`的地得`、`这那`等等，英语里的 `a`、`an`、`the`，等等，索引时会忽略

## Tokenizers

tokenizer 的工作就是把文本分割为词元\(token\)，每个词元通常是文本里的字符的子序列。tokenizer 读取字符流，并生成一连串词元\(token\)对象

* 输入流的字符可能会被丢弃，例如空格或其他分隔符；也可能增加或替换，例如别名或标准的缩写词 
* 一个词元\(token\)，除了它的文本值以外，还包含不同的元数据，例如词元在字段里出现的位置。
* 由于 tokenizer 可能产生和输入文本偏离的词元，你不应假设词元的文本和字段里出现的文本相同，或者长度和原始输入文本长度一样
* 同样，多个词元在文本中有相同的位置或指向相同的偏移，也是可能滴 
* 在诸如高亮查询中使用词元元数据时，记住这些

```markup
<fieldType name="text" class="solr.TextField">
    <analyzer>
        <tokenizer class="solr.StandardTokenizerFactory"/>
    </analyzer>
</fieldType>
```

tokenizer 元素里的 class 名称，并非实际的 tokenizer，而是一个 实现了 TokenizerFactory 接口的类名称。这个工厂类将在需要的时候创建 tokenizer 的实例，该实例应源于 Tokenizer\(子类吗?\)，用于创建一连串的词元\(token\)

如果这个分词器\(tokenizer\)生成的词元可用，它可能会是 analyzer 的唯一组件；否则，这个分词器输出的词元将作为管道里第一个过滤器\(filter\)的输入

一个 TypeTokenFilterFactory 可以创建一个 TypeTokenFilter 来过滤词元，基于他们的 TypeAttribute, 在 factory 的 getStopTypes 方法里设置

## filters

在 schema.xml 里，作为 `<analyzer>` 的子元素，使用 `<filter>` 元素来配置过滤器\(filter\)，需要在 `<tokenizer>` 或另一个 `<filter>` 的后面，因为 filter 需要 token 流作为输入

