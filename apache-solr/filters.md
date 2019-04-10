# Filters

## ASCIIFoldingFilter

* 将 Unicode 转换成 ascii
* 工厂类
  * solr.ASCIIFoldingFilterFactory
* 示例

```bash
输入
  "á" (Unicode character 00E1)

输出
  "a" (ASCII character 97)
```

## Beider-MorseFilter

* Beider-Morse Phonetic Matching \(BMPM\)，貌似是一个识别名字的算法，即使名字拼写错误或者是用的不同的语言
* 工厂类
  * solr.BeiderMorseFilterFactory
* 参数
  * nameType：名字的类型，取值为 \[GENERIC , ASHKENAZI , SEPHARDIC\]。如果不处理 Ashkenazi 或者 Sephardic 名字, 就用 GENERIC
  * ruleType：规则类型，取值为 \[APPROX（近似） , EXACT（精确）\]
  * concat：多个可能的匹配是否用管道符"\|"连接
  * languageSet：语言集，"auto" 表示允许过滤器识别语言，或者是一个逗号分隔的列表
* 示例

```markup
<analyzer>
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.BeiderMorseFilterFactory" nameType="GENERIC" ruleType="APPROX"
            concat="true" languageSet="auto">
    </filter>
</analyzer>
```

## ClassicFilter

* 这个过滤器接收 ClassicTokenizer 的输出，去掉缩写词里的句点和所有格里的 s
* 工厂类
  * solr.ClassicFilterFactory
* 示例

```markup
<analyzer>
    <tokenizer class="solr.ClassicTokenizerFactory"/>
    <filter class="solr.ClassicFilterFactory"/>
</analyzer>
```

```bash
输入
  "I.B.M. cat's can't"

词元
  "I.B.M", "cat's", "can't"

输出
  "IBM", "cat", "can't"
```

## CommonGramsFilter

* 貌似是用来过滤常用词的，例如停止词：将常用词和词元组合
* 工厂类
  * solr.CommonGramsFilterFactory
* 参数
  * words：常用词文件名，txt 格式，例如 stopwords.txt.
  * format：可选的，如果常用词文件格式是 Snowball，通过这个参数指定
  * ignoreCase：与常用词文件比较时，是否忽略大小写，默认=fasle
* 示例

```markup
<analyzer>
    <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.CommonGramsFilterFactory" words="stopwords.txt"
                ignoreCase="true"/>
</analyzer>
```

```bash
输入
  "the Cat"

词元
  "the", "Cat"

输出
  "the_cat"
```

## CollationKeyFilter

* 可用于语言相关的排序，一般用于排序，也可用于高级搜索

## Daitch-MokotoffSoundexFilter

* 实现了 Daitch-Mokotoff Soundex 算法\(?\)，用于识别相似的名字，即使有拼写错误
* 工厂类
  * solr.DaitchMokotoffSoundexFilterFactory
* 参数
  * inject ：true 表示新的词元会添加到词元流，false 表示开启匹配但是精确的拼写将不会被匹配。默认=true

## DoubleMetaphoneFilter

* 使用 DoubleMetaphone 编码来创建词元
* 工厂类
  * solr.DoubleMetaphoneFilterFactory
* 参数
  * inject：true 表示新的词元会添加到词元流，false 表示开启匹配但是精确的拼写将不会被匹配。默认=true
  * maxCodeLength：将要生成的 code 的最大长度
* 示例

```markup
<analyzer>
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.DoubleMetaphoneFilterFactory"/>
</analyzer>
```

```bash
输入
  "four score and Kuczewski"

词元
  "four"(1), "score"(2), "and"(3), "Kuczewski"(4)

输出
  "four"(1), "FR"(1), "score"(2), "SKR"(2), "and"(3), "ANT"(3), "Kuczewski"(4), "KSSK"(4), "KXFS"(4)
```

## EdgeN-GramFilter

* 貌似和 edge n-gram Tokenizer 没什么不同
* 工厂类
  * solr.EdgeNGramFilterFactory
* 参数
  * minGramSize：gram 最小值，默认=1
  * maxGramSize：gram 最大值，默认=1
* 示例

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.EdgeNGramFilterFactory"/>
</analyzer>
```

```bash
输入
  "four score and twenty"

词元
  "four", "score", "and", "twenty"

输出
  "f", "s", "a", "t"
```

* 示例2

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.EdgeNGramFilterFactory" minGramSize="1" maxGramSize="4"/>
</analyzer>
```

```bash
输入
  "four score"

词元
  "four", "score"

输出
  "f", "fo", "fou", "four", "s", "sc", "sco", "scor"
```

## EnglishMinimalStemFilter

* 将英文单词的复数形式变为单数形式
* 工厂类
  * solr.EnglishMinimalStemFilterFactory
* 参数
  * 无
* 示例

```markup
<analyzer type="index">
  <tokenizer class="solr.StandardTokenizerFactory "/>
  <filter class="solr.EnglishMinimalStemFilterFactory"/>
</analyzer>
```

```bash
输入
  "dogs cats"

词元
  "dogs", "cats"

输出
  "dog", "cat"
```

## FingerprintFilter

* 将输入的多个 token 排序，去重，然后串联在一起，返回单个 token，可用于聚集/链接的用例\(?\)
* 工厂类
  * solr.FingerprintFilterFactory
* 参数
  * separator ：用于连接多个 token 的字符，默认是空格\(" "\)
  * maxOutputTokenSize ：输出的 token 最大长度，如果超过的话，未输出的 token 被忽略\(?\)，默认=1024
* 示例

```markup
<analyzer type="index">
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.FingerprintFilterFactory" separator="_" />
</analyzer>
```

```bash
输入
  "the quick brown fox jumped over the lazy dog"

词元
  "the", "quick", "brown", "fox", "jumped", "over", "the", "lazy", "dog"

输出
  "brown_dog_fox_jumped_lazy_over_quick_the"
```

## HunspellStemFilter

貌似是个英文词根过滤器，把各种时态的动词变成一般现在时态

## HyphenatedWordsFilter

用于处理用连字符连接的单词：去掉连字符；一般是因为换行导致一个单词位于 2 行，这时可以用连字符表示这是同一个单词

## ICUFoldingFilter

这个过滤器可以更好的代替 ASCII Folding Filter, Lower Case Filter, ICU Normalizer 2 Filter 的组合使用

## ICUNormalizer2Filter

## ICUTransformFilter

## KeepWordFilter

* 凡是不在保留字文件里的 token 都会被丢弃
* 工厂类
  * solr.KeepWordFilterFactory
* 参数
  * words：必须，保留字文件路径，绝对路径或者相对路径（相对 solr 配置目录）均可；该文件是个文本文件，每行一个保留字，空行或者以 \# 开头的行会被忽略
  * ignoreCase：是否忽略大小写，默认=false；若为 true，那么会假设保留字文件里的保留字全部为小写
  * enablePositionIncrements：如果 luceneMatchVersion&gt;=5.0，该参数无效
* 示例

```bash
keepwords.txt内容如下

happy
funny
silly
```

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.KeepWordFilterFactory" words="keepwords.txt"/>
</analyzer>
```

```bash
输入
  "Happy, sad or funny"

词元
  "Happy", "sad", "or", "funny"

输出
  "funny"
```

* 示例2

```bash
keepwords.txt内容如下

happy
funny
silly
```

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.KeepWordFilterFactory" words="keepwords.txt" ignoreCase="true"/>
</analyzer>
```

```bash
输入
  "Happy, sad or funny"

词元
  "Happy", "sad", "or", "funny"

输出
  "Happy", "funny"
```

* 示例3

  先用 LowerCaseFilterFactory 过滤，KeepWordFilterFactory 不忽略大小写

```bash
keepwords.txt内容如下

happy
funny
silly
```

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.LowerCaseFilterFactory"/>
  <filter class="solr.KeepWordFilterFactory" words="keepwords.txt"/>
</analyzer>
```

```bash
输入
  "Happy, sad or funny"

词元
  "Happy", "sad", "or", "funny"

输出
  "happy", "funny"
```

## KStemFilter

貌似是对英文动词时态进行过滤

## LengthFilter

* 对 token 的长度进行过滤，范围内的 token 保留，范围外的 token 被丢弃
* 工厂类
  * solr.LengthFilterFactory
* 参数
  * min：token 最小长度
  * max：token 最大长度，必须 &gt;= min
* 示例

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.LengthFilterFactory" min="3" max="7"/>
</analyzer>
```

```bash
输入
  "turn right at Albuquerque"

词元
  "turn", "right", "at", "Albuquerque"

输出
  "turn", "right"
```

## LowerCaseFilter

很显然是把大小转换为小写的过滤器

## ManagedStopFilter

* 和 StopFilter 一样，只不过 stopword 文件是通过一个 url 获取而不是直接访问磁盘文件
* 参数
  * managed：貌似是一个 REST API 接口的名字

## ManagedSynonymFilter

* 和 SynonymFilter 一样，只不过 synonyms 文件是通过一个 url 获取而不是直接访问磁盘文件
* 参数
  * managed：貌似是一个 REST API 接口的名字

## N-GramFilter

和 n-gram 分词器类似吧

## NumericPayloadTokenFilter

貌似是给 token 一个浮点数的值

## PatternReplaceFilter

* 将 token 与一个正则表达式匹配，匹配的就替换成指定的值，不匹配的保持原样
* 工厂类
  * solr.PatternReplaceFilterFactory
* 参数
  * pattern：必须，正则表达式，用来匹配 token
  * replacement：必须，用来替换 token 中匹配的部分的字符串
  * replace：取值为 \["all" , "first"\]，默认=all，表示是替换所有匹配的还是仅替换第一个匹配的
* 示例

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.PatternReplaceFilterFactory" pattern="cat" replacement="dog"/>
</analyzer>
```

```bash
输入
  "cat concatenate catycat"

词元
  "cat", "concatenate", "catycat"

输出
  "dog", "condogenate", "dogydog"
```

* 示例2

  仅替换第一个匹配

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.PatternReplaceFilterFactory" pattern="cat" replacement="dog" replace="first"/>
</analyzer>
```

```bash
输入
  "cat concatenate catycat"

词元
  "dog", "condogenate", "dogycat"

输出
  "dog", "condogenate", "dogydog"
```

* 示例3

  复杂的

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.PatternReplaceFilterFactory" pattern="(\D+)(\d+)$"
replacement="$1_$2"/>
</analyzer>
```

```bash
输入
  "cat foo1234 9987 blah1234foo"

词元
  "cat", "foo1234", "9987", "blah1234foo"

输出
  "cat", "foo_1234", "9987", "blah1234foo"
```

## PhoneticFilter

基于 phonetic 语音编码算法过滤......

## PorterStemFilter

貌似这又是一个对英文单词时态做过滤的

## RemoveDuplicatesTokenFilter

* 移除重复的 token，所谓重复 token 是文本和位置值相同的 token
* 工厂类
  * solr.RemoveDuplicatesTokenFilterFactory
* 参数
  * 无
* 示例

  本例中用到了同义词过滤器，同义词文件（synonyms.txt）内容如下

```bash
Television, Televisions, TV, TVs
```

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt"/>
  <filter class="solr.EnglishMinimalStemFilterFactory"/>
  <filter class="solr.RemoveDuplicatesTokenFilterFactory"/>
</analyzer>
```

```bash
输入
  "Watch TV"

词元1
  "Watch"(1) "TV"(2)

词元2
  "Watch"(1) "Television"(2) "Televisions"(2) "TV"(2) "TVs"(2)

词元3
  "Watch"(1) "Television"(2) "Television"(2) "TV"(2) "TV"(2)

输出
  "Watch"(1) "Television"(2) "TV"(2)
```

## ReversedWildcardFilter

通配符反转过滤器，不含通配符的 token 不会反转，例如输入 "abc_"，_输出 _"_cba"

## ShingleFilter

把输入的 token 组合成一个新的 token，组合采用 n-gram 算法

## SnowballPorterStemmerFilter

一个词根相关的过滤器，貌似不支持中文

## StandardFilter

貌似在 luceneMatchVersion &gt;= 3.1时无效

提示：luceneMatchVersion 的值在 solrconfig.xml 里查看

## StopFilter

* 停止词过滤，停止词会被丢弃，solr 自带了一个标准的停止词文件，即配置目录下的 stopwords.txt，用于英文......
* 工厂类
  * solr.StopFilterFactory
* 参数
  * words：可选，停止词文件路径，停止词文件每行一个停止词，空行和 \# 开头的行被忽略，可以用绝对路径，或相对\(solr 配置目录\)
  * format：可选，停止词格式，取值可以是 snowball
  * ignoreCase：是否忽略大小写，默认=false
  * enablePositionIncrements：如果 luceneMatchVersion &gt;=5.0，该参数无效
* 示例

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.StopFilterFactory" words="stopwords.txt"/>
</analyzer>
```

```bash
输入
  "To be or what?"

词元
  "To"(1), "be"(2), "or"(3), "what"(4)

输出
  "To"(1), "what"(4)
```

## SuggestStopFilter

* 与 StopFilter 的区别是：不会把最后一个 token 移除，除非是该 token 紧跟一个分隔符（例如空格，标点符号，......）
* StopFilter 用于索引的分词，而 SuggestStopFilter 用于查询的分词

## SynonymFilter

* 同义词过滤器，每个 token 都会在同义词列表里查找，如果有匹配，会用同义词代替原 token
* 工厂类
  * solr.SynonymFilterFactory
* 参数
  * synonyms：必须，同义词文件路径，有2种指定同义词映射的方式
    * 逗号分隔的单词列表，token 匹配任意一个单词，所有同义词都会替换原 token，且原 token 会保留
    * 2个逗号分隔的单词列表，2个列表之间用"=&gt;"分隔，token 匹配左边列表中任意单词，会用右边列表里的单词进行替换的同义词列表，除非右边列表里有原 token，否则原 token 不会保留
  * ignoreCase：可选的，是否忽略大小写，默认=false
  * expand：可选，true 表示用所有同义词来替换原 token，false 表示只用第一个同义词来替换，默认=true
  * format：可选，默认=solr，同义词格式
  * tokenizerFactory：可选，默认=WhitespaceTokenizerFactory，用来对同义词文件进行分析的分词器
  * analyzer：可选，默认=WhitespaceTokenizerFactory，用来对同义词文件进行分析的分析器
* 同义词文件 mysynonyms.txt:

```bash
couch,sofa,divan
teh => the
huge,ginormous,humungous => large
small => tiny,teeny,weeny
```

* 示例

```markup
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.SynonymFilterFactory" synonyms="mysynonyms.txt"/>
</analyzer>
```

```bash
输入
  "teh small couch"

词元
  "teh"(1), "small"(2), "couch"(3)

输出
  "the"(1), "tiny"(2), "teeny"(2), "weeny"(2), "couch"(3), "sofa"(3), "divan"(3)


输入
  "teh ginormous, humungous sofa"

词元
  "teh"(1), "ginormous"(2), "humungous"(3), "sofa"(4)

输出
  "the"(1), "large"(2), "large"(3), "couch"(4), "sofa"(4), "divan"(4)
```

## TokenOffsetPayloadFilter

* 将 token 在原输入流中的位置记录下来，位置表示为\[起始位置, 结束位置\]
* 工厂类
  * solr.TokenOffsetPayloadTokenFilterFactory
* 参数
  * 无
* 示例

```markup
<analyzer>
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.TokenOffsetPayloadTokenFilterFactory"/>
</analyzer>
```

```bash
输入
  "bing bang boom"

词元
  "bing", "bang", "boom"

输出
  "bing"[0,4], "bang"[5,9], "boom"[10,14]
```

## TrimFilter

* 去掉 token 头尾的空白，多数分词器都是用空白进行分词的，所以这个过滤器用于一些特殊的场景
* 工厂类
  * solr.TrimFilterFactory
* 参数
  * updateOffsets：如果 luceneMatchVersion&gt;=5.0，该参数无效
* 示例

```markup
<analyzer>
  <tokenizer class="solr.PatternTokenizerFactory" pattern=","/>
  <filter class="solr.TrimFilterFactory"/>
</analyzer>
```

```bash
输入
  "one, two , three ,four "

词元
  "one", " two ", " three ", "four "

输出
  "one", "two", "three", "four"
```

## TypeAsPayloadFilter

为 token 添加类型，例如

```bash
输入
  "Pay Bob's I.O.U."

词元
  "Pay", "Bob's", "I.O.U."

输出
  "Pay"[<ALPHANUM>], "Bob's"[<APOSTROPHE>], "I.O.U."[<ACRONYM>]
```

## TypeTokenFilter

* 貌似是黑白名单的过滤器\(?\)，又或者是给 token 标记类型的过滤器\(?\)
* 工厂类
  * solr.TypeTokenFilterFactory
* 参数
  * types：类型文件的位置
  * useWhitelist：
    * true， types 是个白名单
    * false，types 是个黑名单
  * enablePositionIncrements：luceneMatchVersion&gt;=5.0，该值无效

## WordDelimiterFilter

* 用分隔符 token 的过滤器，下面的规则用于决定什么是分隔符
  * 大小写，例如："CamelCase" -&gt; "Camel", "Case"。开关为 splitOnCaseChange="0"
  * 字母数字，例如："Gonzo5000" -&gt; "Gonzo", "5000" "4500XL" -&gt; "4500", "XL"。开关为 splitOnNumerics="0"
  * 非字母数字的字符被丢弃，例如："hot-spot" -&gt; "hot", "spot"
  * 末尾的"s"被移除，例如："O'Reilly's" -&gt; "O", "Reilly"
  * 头尾的分隔符被丢弃，例如："--hot-spot--" -&gt; "hot", "spot"
* 工厂类
  * solr.WordDelimiterFilterFactory
* 参数
  * generateWordParts：默认=1，If non-zero, splits words at delimiters. For example:"CamelCase","hot-spot" -&gt; "Camel", "Case", "hot", "spot"
  * generateNumberParts：默认=1，If non-zero, splits numeric strings at delimiters:"1947-32" -&gt;"1947", "32"
  * splitOnCaseChange：默认=1，If 0, words are not split on camel-case changes:"BugBlaster-XL" -&gt; "BugBlaster", "XL". Example 1 below illustrates the default \(non-zero\) splitting behavior.
  * splitOnNumerics：默认=1，If 0, don't split words on transitions from alpha to numeric:"FemBot3000" -&gt; "Fem", "Bot3000"
  * catenateWords：默认=0，If non-zero, maximal runs of word parts will be joined："hot-spot-sensor's" -&gt; "hotspotsensor"
  * catenateNumbers：默认=0，If non-zero, maximal runs of number parts will be joined：1947-32" -&gt; "194732"
  * catenateAll：默认=0，If non-zero, runs of word and number parts will be joined："Zap-Master-9000" -&gt; "ZapMaster9000"
  * preserveOriginal：默认=0，If non-zero, the original token is preserved："Zap-Master-9000" -&gt; "Zap-Master-9000", "Zap", "Master", "9000"
  * protected：可选，保护文件的路径，该文件包含一个保护词列表，这些保护词不会被分割
  * stemEnglishPossessive：默认=1，去掉表示所有格的"s"
* 示例

```markup
<analyzer>
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.WordDelimiterFilterFactory"/>
</analyzer>
```

```bash
输入
  "hot-spot RoboBlaster/9000 100XL"

词元
  "hot-spot", "RoboBlaster/9000", "100XL"

输出
  "hot", "spot", "Robo", "Blaster", "9000", "100", "XL"
```

* 示例2

  不用大小写分隔，不生成数字 token

```markup
<analyzer>
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.WordDelimiterFilterFactory" generateNumberParts="0" splitOnCaseChange="0"/>
</analyzer>
```

```bash
输入
  "hot-spot RoboBlaster/9000 100-42"

词元
  "hot-spot", "RoboBlaster/9000", "100-42"

输出
  "hot", "spot", "RoboBlaster", "9000"
```

* 示例3

```markup
<analyzer>
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.WordDelimiterFilterFactory" catenateWords="1" catenateNumbers="1"/>
</analyzer>
```

```bash
输入
  "hot-spot 100+42 XL40"

词元
  "hot-spot"(1), "100+42"(2), "XL40"(3)

输出
  "hot"(1), "spot"(2), "hotspot"(2), "100"(3), "42"(4), "10042"(4), "XL"(5), "40"(6)
```

* 示例3

```markup
<analyzer>
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.WordDelimiterFilterFactory" catenateAll="1"/>
</analyzer>
```

```bash
输入
  "XL-4000/ES"

词元
  "XL-4000/ES"(1)

输出
  "XL"(1), "4000"(2), "ES"(3), "XL4000ES"(3)
```

* 示例4

  使用一个保护文件，包含了"AstroBlaster", "XL-5000"

```markup
<analyzer>
  <tokenizer class="solr.WhitespaceTokenizerFactory"/>
  <filter class="solr.WordDelimiterFilterFactory" protected="protwords.txt"/>
</analyzer>
```

```bash
输入
  "FooBar AstroBlaster XL-5000 ==ES-34-"

词元
  "FooBar", "AstroBlaster", "XL-5000", "==ES-34-"

输出
  "FooBar", "FooBar", "AstroBlaster", "XL-5000", "ES", "34"
```



