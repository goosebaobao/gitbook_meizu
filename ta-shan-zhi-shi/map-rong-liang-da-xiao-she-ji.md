---
description: by 董唯良 at 2016.7
---

# Map 容量大小设计

## 背景

之前分析了下 Map 的源码，发现了这种底层源码的设计确实有独到之处，特别分享一下。

map 寻址：根据 key 的 hashCode 以及 Map 的长度计算出这个 key 所在的 index

```java
static int indexFor(int h, int length) {
        return h & (length-1);
}
```

这段代码中使用了按位取与的一个操作，而之前我所了解的根据 hashcode 的定位操作时用 `hashcode % length` 这种方式。

 经过验证：这种按位与的结果与 `hashcode % length` 的结果是一致的，但是有一个前提——length 必须为 2 的幂次方。

## map 如何保证自己的 length 为 2 的幂次方？

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);

    // Find a power of 2 >= initialCapacity
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;

    this.loadFactor = loadFactor;
    threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    useAltHashing = sun.misc.VM.isBooted() &&
            (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    init();
}
```

从 HashMap 的构造函数可以看到，map 的容量大小并不是我们传入的 initialCapacity，而是刚刚大于 initialCapacity 并等于 2 的幂次方。

## 举例

```text
假设数组长度分别为 15 和 16，key 的 hash 码分别为 8 和 9，那么 & 运算后的结果如下：

h & (table.length-1)   hash     table.length-1

8 & (15-1)：           0100  &  1110           =     0100

9 & (15-1)：           0101  &  1110           =     0100

------------------------------------------------------------------

8 & (16-1)：           0100  &  1111           =     0100

9 & (16-1)：           0101  &  1111           =     0101
```

* 从上面的例子中可以看出：当 8、9 两个数和 \(15-1\)2=\(1110\) 进行与运算 & 的时候，产生了相同的结果，都为 0100，也就是说它们会定位到数组中的同一个位置上去，这就产生了碰撞，8 和 9 会被放到数组中的同一个位置上形成链表，那么查询的时候就需要遍历这个链 表，得到 8 或者 9，这样就降低了查询的效率。同时，我们也可以发现，当数组长度为 15 的时候，hash 值会与 \(15-1\)2=\(1110\) 进行与运算 &，那么最后一位永远是0，而 0001，0011，0101，1001，1011，0111，1101 这几个位置永远都不能存放元素了，空间浪费相当大。
* 而当数组长度为 16 时，即为 2 的 n 次方时，2n-1 得到的二进制数的每个位上的值都为 1（比如 \(24-1\)2=1111），这使得在低位上 & 时，得到的和原 hash 的低位相同，就使得只有相同的 hash 值的两个值才会被放到数组中的同一个位置上形成链表。

## 结论

我们知道 `&` 比 `%` 具有更高的效率，底层源码为了实现更高的效率还是比较煞费苦心的，也说明这些人确实很厉害。

