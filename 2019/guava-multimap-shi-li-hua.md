# guava Multimap 实例化

以 `SetMultimap` 为例介绍一下

## 直接构造实现类

```java
SetMultimap<Integer, String> dict = HashMultimap.create(8, 256);
```

这种方式需要对实现类底层比较了解，例如 `HashMultimap`，其底层实现是 `HashMap<Integer, HashSet<String>>`，其源码如下

```java
  private HashMultimap() {
    super(new HashMap<K, Collection<V>>());
  }
  ......
  
    @Override
  Set<V> createCollection() {
    return Sets.<V>newHashSetWithExpectedSize(expectedValuesPerKey);
  }
```

## 使用 Builder

```java
SetMultimap<Integer, String> dict = SetMultimapBuilder.treeKeys().linkedHashSetValues().build();
```

这是经典的 builder 模式，可以自定义底层实现

