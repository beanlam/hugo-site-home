---
title: "redis sort 命令详解"
date: 2017-06-07T22:24:47+08:00
categories : ["redis"]
---


# 基本使用

**命令格式**：  
```bash
SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination] 
```

 默认情况下，排序是基于数字的，各个元素将会被转化成双精度浮点数来进行大小比较，这是`SORT`命令最简单的形式，也就是下面这种形式：
```bash
SORT mylist
```

如果`mylist`是一个包含了数字元素的列表，那么上面的命令将会返回升序排列的一个列表。如果想要降序排序，要使用`DESC`描述符，如下所示：
```bash
SORT mylist DESC
```
如果`mylist`包含的元素是string类型的，想要按字典顺序排列这个列表，那么就要用到`ALPHA`描述符，如下所示：
```bash
SORT mylist ALPHA
```
Redis是基于UTF-8编码来处理数据的, 要确保你先设置好了!LC_COLLATE环境变量。

对于返回的元素个数也是可以进行限制的，只需要使用`LIMIT`描述符。使用这个描述符，你需要提供偏移量参数，来指定需要跳过多少个元素，返回多少个元素。 下面这个例子将会返回一个已经排序好了的列表中的10个元素，从下标为0开始：
```bash
SORT mylist LIMIT 0 10
```
几乎全部的描述符都可以同时使用。例如下面这个例子所示，它将会返回前5个元素，以字典顺序降序排列：
```bash
SORT mylist LIMIT 0 5 ALPHA DESC
```

# 通过外部key来排序

有时候，你会想要用外部的key来作为权重去排列列表或集合中的元素，而不是使用列表或集合中本来就有的元素来排列。
下面以一个例子来解释：
假设现在有一张这样的表，有商品id，商品价格，以及商品的重量。

| gid | price_{gid} | weight_{gid} |
| ------------- | ------------- |----|
| 1 | 20 |3|
| 2  | 40 |2|
| 3|30|4|
|4|10|1|

首先将上述表格的数据导入到redis中(redis版本：2.8.6)

```bash
127.0.0.1:6379> lpush gid 1
(integer) 1
127.0.0.1:6379> lpush gid 2
(integer) 2
127.0.0.1:6379> lpush gid 3
(integer) 3
127.0.0.1:6379> lpush gid 4
(integer) 4
127.0.0.1:6379> set price_1 20
OK
127.0.0.1:6379> set price_2 40
OK
127.0.0.1:6379> set price_3 30
OK
127.0.0.1:6379> set price_4 10
OK
127.0.0.1:6379> set weight_1 3
OK
127.0.0.1:6379> set weight_2 2
OK
127.0.0.1:6379> set weight_3 4
OK
127.0.0.1:6379> set weight_4 1
OK

```

默认情况下对gid排序，只会按gid的值来排序

```bash
127.0.0.1:6379> sort gid
1) "1"
2) "2"
3) "3"
4) "4"
```
但是通过BY描述符，可以指定gid按照别的key来排序。例如我想要按照商品的价格来排序：

```bash
127.0.0.1:6379> sort gid by price_*
1) "4"
2) "1"
3) "3"
4) "2"
```
可以看到，gid为4的商品价格最低，为10，gid为2的商品价格最高，为40。

在这里，price_\*中的\*是一个占位符，先会把gid的值取出，代入到\*中，再去查找price_\*的值。例如在本例中，price_\*中的\*会分别被1,2,3,4代替，然后去取price_1,price_2,price_3,price_4的值来进行排序。


# 跳过排序(不进行排序)

BY描述符也可以使用一个根本不存在的key来作为排序规则，则直接导致的结果就是不进行任何排序。 这看上去似乎没什么用，但是如果你想要取得外部的keys时，跳过排序就非常有用了，这也会减少性能损耗。

要理解不排序的好处，首先要先了解一下GET描述符。
使用GET描述符，可以根据排序结果取出外部键值。例如以下命令：

```bash
127.0.0.1:6379> sort gid get price_*
1) "20"
2) "40"
3) "30"
4) "10"
```

先对gid排序，然后再分别取出price_{gid}的值。
也可以使用多个GET，获取多个外部key的值，例如：
```bash
127.0.0.1:6379> sort gid get price_* get weight_*
1) "20" # 这是price
2) "3"  # 这是weight，以下类推
3) "40"
4) "2"
5) "30"
6) "4"
7) "10"
8) "1"
```

get也可以使用#来获取被排序的key的值，例如：
```bash
127.0.0.1:6379> sort gid get # get price_*
1) "1"  #这里取出的就是gid
2) "20" #这里是price，以下类推
3) "2"
4) "40"
5) "3"
6) "30"
7) "4"
8) "10"
```

通过结合不排序和GET，能够在不排序的情况下，一次取得多个key的值。例如：
```bash
127.0.0.1:6379> sort gid by no-existed get # get price_* get weight_*
 1) "4"   #这是gid
 2) "10"  #这是price
 3) "1"   #这是weight，以下类推
 4) "3"
 5) "30"
 6) "4"
 7) "2"
 8) "40"
 9) "2"
10) "1"
11) "20"
12) "3"
```

# 将排序结果保存在Redis中
默认情况下，排序结果会返回给客户端。使用STORE描述符，可以将结果存储在指定key上，如果Key已经存在，则覆盖。而不将排序结果返回给客户端。用法如下：
```bash
SORT mylist BY weight_* STORE resultkey
```
下面举一个例子：
```bash
127.0.0.1:6379> rpush num 1 3 5
(integer) 3
127.0.0.1:6379> rpush num 2 4 6
(integer) 6
127.0.0.1:6379> lrange num 0 -1
1) "1"
2) "3"
3) "5"
4) "2"
5) "4"
6) "6"
127.0.0.1:6379> sort num store num_sorted
(integer) 6
127.0.0.1:6379> lrange num_sorted 0 -1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
```

# 将哈希表作为GET或BY的参数

可以将哈希表的字段作为GET或者BY的参数，用法如下：
```bash
SORT mylist BY weight_*->fieldname GET object_*->fieldname
```
对于前面所用到的例子：

| gid | price_{gid} | weight_{gid} |
| ------------- | ------------- |----|
| 1 | 20 |3|
| 2  | 40 |2|
| 3|30|4|
|4|10|1|

我们可以不把price_{gid}和weight_{id}用string类型来保存，而是对于每一个gid，创建一个包含price和weight字段的hash表good_info_{gid}来存储商品信息。
```bash
127.0.0.1:6379> hmset good_info_1 price 20 weight 3
OK
127.0.0.1:6379> hmset good_info_2 price 40 weight 2
OK
127.0.0.1:6379> hmset good_info_3 price 30 weight 4
OK
127.0.0.1:6379> hmset good_info_4 price 10 weight 1
OK
```
之后， by 和 get 选项都可以用 key->field 的格式来获取哈希表中的域的值， 其中 key 表示哈希表键， 而 field 则表示哈希表的域：
```bash
127.0.0.1:6379> sort gid by good_info_*->price
1) "4"
2) "1"
3) "3"
4) "2"
127.0.0.1:6379> sort gid by good_info_*->price get good_info_*->weight
1) "1"
2) "3"
3) "4"
4) "2"
```
# 返回值
如果没有使用 store 描述符，返回列表形式的排序结果。
如果使用 store 参数，返回排序结果的元素数量。

# 参考资料
[Redis官网SORT命令介绍](http://redis.io/commands/sort)