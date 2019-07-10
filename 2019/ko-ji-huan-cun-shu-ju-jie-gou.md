# KO 及缓存数据结构

## 背景

说到 KO，你以为是这样的吗？

![&#x56FE;&#x7247;&#x6765;&#x81EA;&#x7F51;&#x7EDC;](../.gitbook/assets/ko.jpg)

先回顾一下之前的分享：

{% page-ref page="java-yi-wei-yun-suan.md" %}

由于缓存的 key 由多个属性组成，就涉及到了缓存数据结构的设计

* 多维 map
* 一维 map + 组合 key

在那篇文章里，由于多个属性都是 int，最终选择了组合 key 的方案。

那么，如果多个属性不止是 int，还有 String，该怎么处理呢？多维 map 和 一维 map 如何取舍呢？

##  多维 map

guava 提供了 Table 接口，实际上是个二维 map，查看 guava 的源码可以发现，这个 Table 还是很重量级的，如果你的需求仅仅是 put\(\)， get\(\)，那我不建议使用 Table

##  一维 map + 组合 key

所以本文主要研究组合 key 的情况，并提出一个专用于组合 key 的对象： KO，含义为 key object

### KO

一个标准的 KO 应该满足如下条件

* 继承自 Object，且为 final 类
* 必须提供带参数的构造方法，且该方法为唯一的构造方法
* 建议使用静态工厂方法来创建实例，此时构造方法应为私有的
* 在构造方法里计算出 hashcode
* 类属性是 final 的
* 必须重写 hashCode\(\)， equals\(\)
* 建议重写 toString\(\)

示例：一个组合 imei\(整数\)，sn\(字符串\)的 KO

```java
import java.util.Objects;

public final class ImeiSnKo {

    private final Long   imei;
    private final String sn;
    private final int    hashCode;


    public static final ImeiSnKo create(Long imei, String sn) {
        return new ImeiSnKo(imei, sn);
    }


    private ImeiSnKo(Long imei, String sn) {
        this.imei = imei;
        this.sn = sn;

        this.hashCode = 31 * Objects.hashCode(imei) + Objects.hashCode(sn);
    }


    @Override
    public int hashCode() {
        return hashCode;
    }


    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }
        if (obj instanceof ImeiSnKo) {
            ImeiSnKo ko = (ImeiSnKo) obj;
            return Objects.equals(imei, ko.imei) && Objects.equals(sn, ko.sn);
        }
        return false;
    }

}
```

### KO 的优势

#### 和拼接字符串构造 key 来对比

* 含义明确，提升代码可读性
* 性能优势：无需进行字符串拼接操作
* 内存占用优势：没有产生新的字符串对象

#### 和通过移位构造 key 来对比

* 代码可读性的提升是明显的，移位运算确实很难理解
* 性能上应该还是移位更优
* 消除隐患：移位运算要小心计算移动的位数，避免随着业务发展构造 key 的整数值突破预留的位数，KO 完全不需要担心，下面示例一下我改造后的代码

```java
public final class TripleIntKey {

    private final int i;
    private final int j;
    private final int k;

    private final int hash;


    public static final TripleIntKey create(int i, int j, int k) {
        return new TripleIntKey(i, j, k);
    }


    private TripleIntKey(int i, int j, int k) {
        this.i = i;
        this.j = j;
        this.k = k;
        this.hash = 31 * (31 * i + j) + k;
    }


    @Override
    public int hashCode() {
        return hash;
    }


    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }

        if (obj instanceof TripleIntKey) {
            TripleIntKey ko = (TripleIntKey) obj;
            return i == ko.i && j == ko.j && k == ko.k;
        }
        return false;
    }


    @Override
    public String toString() {
        return new StringBuilder().append(i).append('-').append(j).append('-').append(k).toString();
    }
}

```

