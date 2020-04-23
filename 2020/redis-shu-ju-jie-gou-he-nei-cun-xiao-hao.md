# redis 数据结构和内存消耗

在上一篇分享

{% page-ref page="redis-nei-cun-zhan-yong-you-hua.md" %}

里，使用了多种数据结构来实现同一个业务需求，同时还有一个关于节省内存的建议

> 如果 value 的值没有实际意义，建议存为 0

本文就研究下实现同样的功能，不同数据结构的内存的消耗

以下测试均使用 redis 4.0 版本

## integer vs string

```bash
127.0.0.1:6379> set key1 1587018589
OK
127.0.0.1:6379> memory usage key1
(integer) 48
127.0.0.1:6379> set key2 "0"
OK
127.0.0.1:6379> memory usage key2
(integer) 48
127.0.0.1:6379> set key3 ""
OK
127.0.0.1:6379> memory usage key3
(integer) 50
127.0.0.1:6379> set key4 x
OK
127.0.0.1:6379> memory usage key4
(integer) 51
127.0.0.1:6379> set key5 xyz
OK
127.0.0.1:6379> memory usage key5
(integer) 53
```

比较意外

1. 保存一个 10 位的整数，和保存 0 占用内存一样，如果这样的话，redis 的共享整数是如何节约内存的？
2. 保存一个空字符串竟然比保存 10 位整数还要多占用 2 个字节

## hash vs set vs bitmap

```bash
127.0.0.1:6379> hset t:ac12041c01717cd2d48200000000 e0 "" e1 "" e2 "" c0 ""
(integer) 4
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000000
(integer) 109
127.0.0.1:6379> hset t:ac12041c01717cd2d48200000001 e0 0 e1 0 e2 0 c0 0
(integer) 4
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000001
(integer) 109
127.0.0.1:6379> sadd t:ac12041c01717cd2d48200000002 0 2 4 1
(integer) 4
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000002
(integer) 90
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 0 1
(integer) 1
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 1 1
(integer) 0
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 2 1
(integer) 0
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 5 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 77
```

可见，内存占用是 hash &gt; set &gt; bitmap

如果是单个元素呢

```bash
127.0.0.1:6379> hset t:ac12041c01717cd2d48200000000 e0 0
(integer) 1
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000000
(integer) 91
127.0.0.1:6379> hset t:ac12041c01717cd2d48200000001 e0 ""
(integer) 1
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000001
(integer) 91
127.0.0.1:6379> sadd t:ac12041c01717cd2d48200000002 0
(integer) 1
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000002
(integer) 84
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 0 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 77
127.0.0.1:6379> setnx t:ac12041c01717cd2d48200000004 0
(integer) 1
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000004
(integer) 74
```

结论不变

## bitmap 扩展

```bash
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 0 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 77
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 1 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 77
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 7 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 77
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 8 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 82
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 15 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 82
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 16 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 82
127.0.0.1:6379> setbit t:ac12041c01717cd2d48200000003 24 1
(integer) 0
127.0.0.1:6379> memory usage t:ac12041c01717cd2d48200000003
(integer) 82
```

初次调用 setbit 时，只分配了 1 个字节的位图, 当位图需要扩展时，会一次扩展 5 个字节

## redis 共享整数和空字符串

```bash
127.0.0.1:6379> hset key1 f1 1234567890
(integer) 1
127.0.0.1:6379> memory usage key1
(integer) 69
127.0.0.1:6379> hset key1 f2 2345678901
(integer) 1
127.0.0.1:6379> memory usage key1
(integer) 83
127.0.0.1:6379> hset key1 f3 3456789012
(integer) 1
127.0.0.1:6379> memory usage key1
(integer) 97
127.0.0.1:6379> hset key2 f1 0
(integer) 1
127.0.0.1:6379> memory usage key2
(integer) 65
127.0.0.1:6379> hset key2 f2 0
(integer) 1
127.0.0.1:6379> memory usage key2
(integer) 71
127.0.0.1:6379> hset key2 f3 0
(integer) 1
127.0.0.1:6379> memory usage key2
(integer) 77
```

这里可以看出差异了，当值为 10 位整数时，每多一个字段，内存占用增加 14，所以每个 k-v 占用 14 字节；当值为 0 时，每多一个字段，内存占用增加 6，即每个 k-v 的内存占用是 6，刚好少了 8 字节

所以我们基本可以确定，10 位的整数，内存占用是 8 字节，而 0 在大量重用时，基本可以认为是不占用内存的

```bash
127.0.0.1:6379> hset key3 f1 ""
(integer) 1
127.0.0.1:6379> memory usage key3
(integer) 65
127.0.0.1:6379> hset key3 f2 ""
(integer) 1
127.0.0.1:6379> memory usage key3
(integer) 71
127.0.0.1:6379> hset key3 f3 ""
(integer) 1
127.0.0.1:6379> memory usage key3
(integer) 77
```

有趣的是，使用 hash 时空字符串和 0 的内存占用是一致的

## string vs list vs hash vs set

在 redis 4.0 里保存 6 个数字: 123456 234561 345612 456123 561234 612345

数据结构分别是 string, list, hash, set

```bash
127.0.0.1:6379> set key1 '123456,234561,345612,456123,561234,612345'
OK
127.0.0.1:6379> object encoding key1
"embstr"
127.0.0.1:6379> memory usage key1
(integer) 91

127.0.0.1:6379> lpush key2 123456 234561 345612 456123 561234 612345
(integer) 6
127.0.0.1:6379> object encoding key2
"quicklist"
127.0.0.1:6379> memory usage key2
(integer) 161

127.0.0.1:6379> hset key3 123456 0 234561 0 345612 0 456123 0 561234 0 612345 0
(integer) 6
127.0.0.1:6379> hgetall key3
 1) "123456"
 2) "0"
 3) "234561"
 4) "0"
 5) "345612"
 6) "0"
 7) "456123"
 8) "0"
 9) "561234"
10) "0"
11) "612345"
12) "0"
127.0.0.1:6379> object encoding key3
"ziplist"
127.0.0.1:6379> memory usage key3
(integer) 101

127.0.0.1:6379> sadd key4 123456 234561 345612 456123 561234 612345
(integer) 6
127.0.0.1:6379> memory usage key4
(integer) 80
127.0.0.1:6379> object encoding key4
"intset"
```

结论：list &gt; hash &gt; string &gt; set

