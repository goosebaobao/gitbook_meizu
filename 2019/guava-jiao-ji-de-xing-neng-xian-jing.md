# guava 交集的性能陷阱

## guava 交集

guava 工具类 Sets 提供了计算 2 个 set 交集的方法，如下

```java
public static <E> SetView<E> intersection(final Set<E> set1, final Set<?> set2) {
    ......
}
```

一般认为 guava 源码比较优秀，其集合工具包大大方便了开发，谁能想到这个交集的方法存在极大的性能陷阱呢？

## 背景

在我去年的分享的一篇文章里，提到了用倒排索引来构建本地缓存

{% page-ref page="../2018/yong-dao-pai-suo-yin-gou-jian-ben-di-huan-cun.md" %}

最近一个需求是“未安装应用定向”，也就是广告只能投放给没有安装特定应用的用户，首先我们构建对应的倒排索引 `Map<AppId, List<AdId>>`，根据应用 Id，可以查询到不能投放的广告 Id

在竞价时，首先从大数据平台查询到用户已安装的应用列表，然后从全部 “未安装定向” 的应用里清除用户已安装应用，得到该用户的未安装应用，即

```java
Set<Long> uninstalled = 
    Sets.difference(targetIndexService.<Long> keywords(UNINSTALLED), installed);
```

接下来，对每一个未安装的应用，查询出不能投放的广告列表，多个应用的不能投放广告计算交集，就得到该用户的不匹配广告，如下

```java
    @Override
    public Set<Integer> unmatch(Collection<T> keywords) {
        Set<Integer> result = allUnitCache;
        for (T keyword : keywords) {
            result = Sets.intersection(result, unmatch(keyword));
            if (result.size() == 0) {
                break;
            }
        }
        return result;
    }
```

新功能上线后，广告竞价接口大量超时，同时 cpu 占用也开始告警，发生了严重的性能问题。经过调试后，问题定位到了 `Sets.intersection()` 方法

## 测试

为了验证这个性能问题，做一个简单的测试

```java
public class GuavaSetsTest extends TestNg {

    @Test
    public void test() {

        Random random = new Random();

        // 构造 300 个 set 做交集
        int setCount = 300;

        // 每个 set 里固定包括 0~~127
        Set<Long> base = new HashSet<>();
        for (long i = 0; i <= 127; i++) {
            base.add(i);
        }


        List<Set<Long>> sets = new ArrayList<>(300);
        for (int i = 0; i < setCount; i++) {
            Set<Long> set = new HashSet<>(base);
            // set 的尺寸为 128+(0~~100)
            int setSize = random.nextInt(100) + 128;
            int max = setSize * 10;
            for (; set.size() < setSize;) {
                set.add(Long.valueOf(random.nextInt(max)));
            }
            sets.add(set);
        }

        Set<Long> result = sets.get(0);

        long start = System.currentTimeMillis();
        for (int n = 1; n < sets.size(); n++) {
            result = Sets.intersection(result, sets.get(n));
            // result.size();
        }
        System.out.println("elapse=" + (System.currentTimeMillis() - start));
    }
}
```

执行测试用例，输出耗时为 `elapse=8` 8 毫秒完成 300 个 set 的交集，这性能岂不是很好？

如果在执行交集以后，获取一下交集的 size 呢？

执行测试用例，输出耗时为 `elapse=207` 

问题就出在这里：guava 的交集操作本身很快，但是对交集返回的结果进行操作就很慢了——尤其是 `size()` 方法，如果改成 `isEmpty()` 就会发现，性能又 ok 了

## 原因分析

其实看下 guava 源码就很清楚了

```java
  @Override
  public UnmodifiableIterator<E> iterator() {
    return Iterators.filter(set1.iterator(), inSet2);
  }

  @Override
  public int size() {
    return Iterators.size(iterator());
  }

  @Override
  public boolean isEmpty() {
    return !iterator().hasNext();
  }
  
  public static int size(Iterator<?> iterator) {
    long count = 0L;
    while (iterator.hasNext()) {
      iterator.next();
      count++;
    }
    return Ints.saturatedCast(count);
  }
```

size\(\) 操作需要遍历整个集合，之所以要遍历整个集合，原因是 guava 的交集实际上是 set1 和 set2 的一个逻辑视图，这个遍历包括了判断 set1 的元素是否存在 set2 里

## 解决办法

这里提供 2 种解决办法

### 基于 guava

```java
result = ImmutableSet.copyOf(Sets.intersection(result, sets.get(n)));
```

将 set 逻辑视图转换成真实的 set

### 自己实现交集

```java
    public static final <E> Set<E> intersection(Set<E> set1, Set<E> set2) {
        Set<E> shortSet, longSet;
        if (set1.size() > set2.size()) {
            longSet = set1;
            shortSet = set2;
        } else {
            longSet = set2;
            shortSet = set1;
        }

        if (shortSet.size() == 0) {
            return Collections.emptySet();
        }

        Set<E> result = Sets.newHashSetWithExpectedSize(shortSet.size());
        for (E item : shortSet) {
            if (longSet.contains(item)) {
                result.add(item);
            }
        }
        return result;
    }
```

