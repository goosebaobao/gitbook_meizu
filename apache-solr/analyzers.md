# Analyzers

## StandardTokenizer

* 将文本分割为 token，空白符和标点被作为分隔符丢弃，但有如下例外
  * 句点后面跟着的不是空白符，会作为 token 的一部分保留，包括互联网域名
  * @ 也视同为标点（也即会作为分隔符被丢弃），所以电子邮件地址不会作为一个 token 保留
* 工厂类
  * solr.StandardTokenizerFactory
* 参数
  * maxTokenLength：如果组成 token 的字符超过这个数字会被忽略，默认=255
* 示例

```bash
输入
  "Please, email john.doe@foo.com by 03-09, re: m37-xq."

输出
  "Please", "email", "john.doe", "foo.com", "by", "03", "09", "re", "m37", "xq"
```

## ClassicTokenizer

* 在 solr 3.1 和之前版本，与 StandardTokenizer 一样
* 将文本分割为 token，空白符和标点作为分隔符被丢弃，但有如下例外
  * 句点后面跟着的不是空白符，则会作为 token 的一部分被保留
  * 如果用连字符\(-\)连接的词中有数字，则连字符不会被当作分隔符；否则连字符或被视同分隔符
  * 识别域名和电子邮件地址，将其作为单个 token 保留
* 工厂类
  * solr.ClassicTokenizerFactory
* 参数
  * maxTokenLength：如果组成 token 的字符超过这个数字会被忽略，默认=255 
* 示例

```bash
输入
  "Please, email john.doe@foo.com by 03-09, re: m37-xq."

输出
  "Please", "email", "john.doe@foo.com", "by", "03-09", "re", "m37-xq"
```

## KeywordTokenizer

* 将整个文本当作一个 token
* 工厂类
  * solr.KeywordTokenizerFactory
* 示例

```bash
输入
  "Please, email john.doe@foo.com by 03-09, re: m37-xq."

输出
  "Please, email john.doe@foo.com by 03-09, re: m37-xq."
```

## LetterTokenizer

* 连续的字符作为 token，非字符全部丢弃
* 工厂类
  * solr.LetterTokenizerFactory
* 示例

```bash
输入
  "I can't."

输出
  "I", "can", "t"
```

## LowerCaseTokenizer

* 非字符作为分隔符，大写转换为小写，空白符和非字符全部丢弃
* 工厂类
  * solr.LowerCaseTokenizerFactory
* 示例

```bash
输入
  "I just LOVE my iPhone!"

输出
  "i", "just", "love", "my", "iphone"
```

## N-GramTokenizer

* 使用 n-gram 模型来分词
* 工厂类
  * solr.NGramTokenizerFactory
* 参数
  * minGramSize: 步长的最小值，必须大于 0，默认=1
  * maxGramSize: 步长的最大值，必须大于等于 minGramSize，默认=2
* 示例

```markup
<analyzer>
    <tokenizer class="solr.NGramTokenizerFactory"/>
</analyzer>
```

```bash
输入
  "hey man"

输出
  "h", "e", "y", " ", "m", "a", "n", "he", "ey", "y ", " m", "ma", "an"
```

* 示例2

```markup
<analyzer>
    <tokenizer class="solr.NGramTokenizerFactory" minGramSize="4" maxGramSize="5"/>
</analyzer>
```

```bash
输入
  "bicycle"

输出
  "bicy", "bicyc", "icyc", "icycl", "cycl", "cycle", "ycle"
```

## EdgeN-GramTokenizer

* 和 n-gram 类似，区别是只能从输入的边缘进行分词，且支持从头和尾进行分词
* 工厂类
  * solr.EdgeNGramTokenizerFactory
* 参数
  * minGramSize: 步长的最小值，必须大于 0，默认=1
  * maxGramSize: 步长的最大值，必须大于等于 minGramSize，默认=1
  * side: 是从头还是从尾开始分词，取值为 \[front, back\]，默认=front
* 示例

```markup
<analyzer>
    <tokenizer class="solr.EdgeNGramTokenizerFactory"/>
</analyzer>
```

```bash
输入
  "babaloo"

输出
  "b"
```

* 示例2

```markup
<analyzer>
    <tokenizer class="solr.EdgeNGramTokenizerFactory" minGramSize="2" maxGramSize="5"/>
</analyzer>
```

```bash
输入
  "babaloo"

输出
  "ba", "bab", "baba", "babal"
```

* 示例3

```markup
<analyzer>
    <tokenizer class="solr.EdgeNGramTokenizerFactory" minGramSize="2" maxGramSize="5"
               side="back"/>
</analyzer>
```

```bash
输入
  "babaloo"

输出
  "oo", "loo", "aloo", "baloo"
```

## ICUTokenizer

* 多语言分词器，需要指定各语言的规则文件
* 工厂类
  * solr.ICUTokenizerFactory
* 参数
  * rulefile: 指定了语言编码及其规则文件，如果有多个语言需要支持，则用逗号分隔。语言编码是 4字符的，根据 ISO 15924 规范
* 示例

```bash
<analyzer>
    <tokenizer class="solr.ICUTokenizerFactory"
               rulefiles="Latn:my.Latin.rules.rbbi,Cyrl:my.Cyrillic.rules.rbbi"/>
</analyzer>
```

## PathHierarchyTokenizer

* 目录层级进行分词
* 工厂类
  * solr.PathHierarchyTokenizerFactory
* 参数
  * delimiter: 目录分隔符
  * replace: 在输出的 token 里用这个代替输入文本里的目录分隔符
* 示例

```bash
<fieldType name="text_path" class="solr.TextField" positionIncrementGap="100">
    <analyzer>
        <tokenizer class="solr.PathHierarchyTokenizerFactory" delimiter="\" replace="/"/>
    </analyzer>
</fieldType>
```

```bash
输入
  "c:\usr\local\apache"

输出
  "c:", "c:/usr", "c:/usr/local", "c:/usr/local/apache"
```

## RegularExpressionPatternTokenizer

* 用 java 的正则表达式来分词
* 工厂类
  * solr.PatternTokenizerFactory
* 参数
  * pattern: 必须，java 正则表达式
  * group: 指定哪个 group 被抽取为 token，默认=-1
    * group=-1：正则表达式作为分隔符来分词
    * group=0：以匹配的 group\(0\) 作为 token，全部匹配\(？\)
    * group&gt;0：以匹配的 group\(1\)...group\(n\) 作为 token
* 示例

```bash
<analyzer>
    <tokenizer class="solr.PatternTokenizerFactory" pattern="\s*,\s*"/>
</analyzer>
```

```bash
输入
  "fee,fie, foe , fum, foo"

输出
  "fee", "fie", "foe", "fum", "foo"
```

* 示例2

```bash
<analyzer>
    <tokenizer class="solr.PatternTokenizerFactory" pattern="[A-Z][A-Za-z]*" group="0"/>
</analyzer>
```

```bash
输入
  "Hello. My name is Inigo Montoya. You killed my father. Prepare to die."

输出
  "Hello", "My", "Inigo", "Montoya", "You", "Prepare"
```

* 示例3

```bash
<analyzer>
<tokenizer class="solr.PatternTokenizerFactory" 
           pattern="(SKU|Part(\sNumber)?):?\s(\[0-9-\]+)" group="3"/>
</analyzer>
```

```bash
输入
  "SKU: 1234, Part Number 5678, Part: 126-987"

输出
  "1234", "5678", "126-987"
```

## UAX29URLEmailTokenizer

* 将文本分割为 token，空白符和标点作为分隔符被丢弃，但有如下例外
  * 句点后面跟着的不是空白符，则会作为 token 的一部分被保留
  * 如果用连字符\(-\)连接的词中有数字，则连字符不会被当作分隔符；否则连字符或被视同分隔符
  * 识别下列情况，并作为单个 token 保留
    * 互联网域名
    * 电子邮件地址
    * file://, http\(s\)://, ftp://
    * IPv4 and IPv6 地址
* 工厂类
  * solr.UAX29URLEmailTokenizerFactory
* 参数
  * maxTokenLength：如果组成 token 的字符超过这个数字会被忽略，默认=255 
* 示例

```bash
输入
  "Visit http://accarol.com/contact.htm?from=external&a=10 or e-mail bob.cratchet@accarol.com"

输出
  "Visit", "http://accarol.com/contact.htm?from=external&a=10", "or", "e", "mail", "bob.cratchet@accarol.com"
```

## WhiteSpaceTokenizer

* 用空白符作为分隔符进行分词，标点符号也作为 token 的一部分保留
* 工厂类
  * solr.WhitespaceTokenizerFactory
* 参数
  * rule：如何定义空白符
    * java: 默认，Character.isWhitespace\(int\)
    * unicode: 使用 Unicode 的空白符属性
* 示例

```bash
输入
  "To be, or what?"

输出
  "To", "be,", "or", "what?"
```

