---
title: 'Redis Guide 使用手册'
date: 2020-07-10 18:28:44
tags: Redis
categories: Redis
---

# Redis 数据结构与应用

## 普通字符串

### SET 命令

1. SET KEY VALUE 设置
2. NX 不存在才设置  可用于实现分布式锁
3. XX 存在才设置

### GET 命令

1. GET KEY 返回 VALUE 值  
2. GETSET 返回旧 OLD_VALUE 值 , 并且设置新 NEW_VALUE

### MSET 命令

1. MSET KEY1 VALUE1 [ KEY2 VALUE2 ...] 一次设置多个键值对

### MGET 命令

1. MGET KEY1 [ KEY2 ...] 一次设置多个键值对
   返回VALUE列表, 这在批量获取多个键值对的时候非常实用, 业务层不用做封装

### MSETNX 命令

1.  MSETNX KEY VALUE [ KEY2 VALUE2 ...] 
   设置多个键值对, 这是一个原子操作, 当且仅当所有KEY都不存在的时候才会成功

### STRLEN 命令

1. STRLEN KEY 获取值的字节长度

### GETRANGE 命令

1. GETRANGE KEY start end 
   根据指定索引范围设置值，相当于取值的子串

### SETRANGE 命令

1. SETRANGE KEY startIndex substitute 
   根据指定开始索引，开始替换新串substitute; 当startIndex大于值长度(len - 1), redis会自动扩展值长度，并且填充空字节(\x00)

### APPEND 命令

1. APPEND KEY suffix 对值进行追加操作, 并返回新串长度
   当KEY不存在, 那么APPEND命令等价于SET操作

### INCRBY DECRBY 命令

1. INCRBY/DECRBY KEY incrment 当存储的值能够被解释为整数时， 可以使用这两个命令来对值进行加或减
   当KEY不存在，那么会被初始化为0，然后再进行操作

### INCR DECR 命令

1. INCR/DECR KEY incrment 当存储的值能够被解释为整数时， 可以使用这两个命令来对值进行加或减1
   当KEY不存在，那么会被初始化为0，然后再进行操作

### INCRBYFLOAT 命令

1. INCRBYFLOAT KEY incrment， 命令与INCRBY无差别，主要是针对的是浮点数
   但是没有DECRBYFLOAT命令，可以通过设置incrment为负数来实现减法
   INCRBYFLOAT 命令对值的格式更加宽松，那么值是整数的时候，命令与INCRBY等同，存储的值也会是整数
   注意Redis在处理浮点数的时候，小数位长度是有限制，最长为17位，对于大部分应用是足够的


## 散列

### 结构

|   KEY      |       FIELD       |         VALUE        |
|  :----:    |      :----:       |      :----:          |
|article     |     title         |     "Hello World"    |
|            |     content       |     "son of bitch"   |
|            |     author        |     genge            |
|            |     created_at    |     2010-10-10       |

### HSET 命令

1. HSET HASHKEY FIELD VALUE
   如果key或者field不存在, 那么会创建，返回值为1
   如果field存在, 那么会更新，返回值为0

### HSETNX 命令

1. HSETNX HASH FIELD VALUE
   只在字段不存在的时候设置值，返回值为1，设置失败返回0

### HGET 命令

1. HGET HASH FIELD VALUE
   获取字段值

### HINCRBY or HINCRFLOATBY

1. HINCRBY HASH FIELD INCRMENT 对数值进行加减


### HSTRLEN 命令

1. HSTRLEN HASH FIELD 获取字段值长度

### HEXISTS 命令

1. 检查字段值是否存在

### HDEL 命令

1. HDEL HASH FILED 删除字段

### HLEN 命令

1. HLEN HASH 获取散列表字段个数

### HMSET/HMGET 系列命令

1. 效果同MSET/MGET

### HKEYS/HVALS/HGETALL 命令

1. 获取散列表中所有字段FIELD
2. 获取散列表中所有VALUE
3. 获取散列表中所有FIELD和VALUE
   格式为数组： FIELD1:VALUE1:FILED2:VALUE2......

## List列表结构

### Redis List列表是一种线性的有序结构

### LPUSH 命令

1. LPUSH KEY VALUE0 [ VALUE1 VALUE2 ...]  返回推入会列表元素个数
   向列表左端新增n个VALUE, 'L' 表示是left 而不是 list

### RPUSH 命令

1. RPUSH KEY VALUE0 [ VALUE1 VALUE2 ...] 返回推入会列表元素个数
   向列表右端新增n个VALUE, 'R' 表示是right

### RPUSHX/LPUSHX 命令

1. 与上两个命令同，但是在列表KEY不存在的时候的结果不一样，这两个命令在列表KEY不存在的时候会推入失败， 返回0

### LPOP/RPOP

1. LPOP/RPOP LISTKEY 弹出最左边/右边的元素， 返回POP后的元素

### RPOPLPUSH 命令

1. RPOPLPUSH source target 返回被弹出的元素值
   将源source列表最右端元素弹出，插入到目标target列表最左端
   其中source和target可以相同，即将列表首尾对调
   当source 为空时，执行会失败，返回空

### LLEN 命令

1. 获取列表长度

### LINDEX 命令

1. LINDEX list index 获取列表指定索引元素
   index为正数： 左端为0， 范围为 0 -- N-1
   index为负数： 右端为-1， 范围为 -N -- -1

### LRANGE 命令

1. 获取指定索引范围内的所有元素
   LRANGE list start end
   LRANGE list 0 -1 获取全部元素
   当start和end都超出范围时，将会返回空列表；当只有一个超出时，Redis会对超出的索引进行修正，开始索引超出会被修正为0，结束索引会被修正为-1

### LSET 命令

1. LSET list index new_value
   对列表指定索引元素值更新

### LINSERT 命令

1. LINSERT list BEFORE/AFTER target_element new_element

### LTRIM 裁剪

1. LTRIM list start index 删除索引范围外所有元素

### LREM 移除

1. LREM list count element  返回被移除的元素数量
   - 如果count等于0，那么命令将会移除list中所有值等于element的元素
   - 如果count等于正数，那么命令会从左端开始扫描，移除列表中值为element的count个元素
   - 如果count等于负数，那么命令会从右端开始扫描，移除列表中值为element的abs(count)个元素

### BLPOP 阻塞式左端弹出

1. BLPOP list1 [ list2 list3 ...] timeout
   - 命令会按照传入的列表从左至右挨个检查是否为空，如果发现某个列表不为空，那么执行LPOP操作，返回值为两个元素的数组，第一个元素是被弹出的列表list名，第二个元素是被弹出的元素值；
   - 如果当前传入的所有list为空，那么Redis将会阻塞等待直至timeout超时，返回空值，超时时间单位为秒，设置为0时表示会一直等待
   - 如果当前有多个客户端因为某个列表空而阻塞，那么按照先阻塞先服务原则进行唤醒
   - 这个命令只会当前Redis客户端

### BRPOP 命令 与上同

### BRPOPLPUSH 阻塞式弹出和推入操作 与上同

1. 可用于实现带有阻塞式的消息队列

## 无序集合Set

### 数据结构

说明无序集合，集合中元素不重复

### SADD 命令

SADD set element [ element ...]

返回值为新增元素个数，会去重

### SREM 命令

SREM set element [element ...]
移除一个或多个元素，返回真实移除元素的个数

### SMOVE 命令

`SMOVE source target element` 将指定元素从source移除，并且加入到目标集合，当source中不存在element的时候会返回失败

### SMEMBERS 命令

`SMEMBERS set` 获取集合所有元素

### SCARD 命令

1. `SCARD set` 获取集合元素个数

### SISMEMBER 命令

1. `SISMEMBER set element` 判断指定元素是否存在集合中

### SRANDMEMBER 命令

1. `SRANDMEMBER SET [count]` 随机获取集合汇总count个元素，count默认值为1

2. count为正数时候，返回随机不重复的min(count, SCARD) 个元素，属于不放回随机抽取
3. count为负数的时候，随机的机制发生变化，属于放回随机抽取，也就是说返回集合有可能出现重复的元素

### SPOP 命令

1. `SPOP set [count]` 随机的重集中移除count个元素，返回被移除的元素集合

### SINTER/SINTERSTORE 命令

1. `SINTER set [set]` 求多个集合的交集

2. `SINTER dest_set set [set ]` 求多个集合的交集，并将结果存储到新的集合中，返回新集合的元素个数

### SUNION/SUNIONSTORE 命令

1. 并集, 含义同上

### SDIFF/SDIFFSTORE 命令

1. 差集，含义同上
2. 集合操作都非常消耗性能，可能导致Redis主线程阻塞

## 有序集合Sorted SET

### 对应数据结构

1. 同时具有有序和集合的性质
2. 每个元素都由一个成员和一个与成员相关联的分值score组成
3. 排行榜最佳实现

`sorted set`
|score   | member  |
|:----:  |  :----: |
| 1 | genge |
| 3 | apple |
| 7 | inuby |
| 7 | oiuby |
| 19| qwmok |

### ZADD 命令

1. `ZADD sorted_set [CH] score element [score element]` 向有序集合中添加元素
2. 默认返回新添加元素的个数，当命令带 CH 的时候返回修改元素的个数

### ZREM 命令

1. `ZREM sorted_set member [member]`  移除指定元素  返回真实被移除的元素个数

### ZSCORE 命令

1. `ZSCORE sorted_set member` 获取指定元素的分值

### ZINCRBY 命令

1. `ZINCRBY sorted_set increment_score memeber` 对指定元素的分值加减
2. 当member不存在的时候，命令等同于ZADD

### ZCARD 命令

1. 获取有序集合元素个数

### ZRANK/ZREVRANK 命令

1. `ZRANK sorted_set member` 指定元素的的正排名 （从小到大）
2. `ZREVRANK sorted_set member` 指定元素的的排名 （从大到小）

### ZRANGE / ZREVRANGE 命令

1. 获取指定范围内成员 `ZRANGE/ZREVRANGE sorted_set start end`
2. start 和 end均可接受负值, 含义是排名
3. `ZRANGE/ZREVRANGE sorted_set start end WITHSCORES` 会返回分值和元素值

### ZRANGEBYSCORE/ZREVRANGEBYSCORE 命令

1. `ZRANGEBYSCORE/ZREVRANGEBYSCORE sorted_set min max/max min` 获取指定范围分数内的成员
2. `WITHSCORES` 可以附带返回分数值
3. `LIMIT offset count` 可以限制返回元素的数量， offset为起止偏移量，count为最大返回数量
4. `(min (max` 使用左括号的表示开区间，默认是闭区间
5. `-inf +inf`来表示负无穷和正无穷

### ZCOUNT 命令

1. `ZCOUNT sorted_set min max` 获取指定分值范围内的成员数量
2. 和上面的RAGNE命令相同，都支持开闭区间无穷等设置

### ZREMRANGEBYRANK 命令

1. `ZREMRANGEBYRANK sorted_set start end` 根据给定的排名区间来移除成员
2. 参数支持负值，表示倒数排名

### ZREMRANGEBYSCORE 命令

1. `ZREMRANGEBYSCORE sorted_set start end` 根据给定的分数区间来移除成员
2. 和上面的RAGNE命令相同，都支持开闭区间无穷等设置

### ZINTERSTORE/ZUNIONSTORE 命令

1. `ZINTERSTORE/ZUNIONSTORE destination numbers sorted_set [sorted_set ]` 求多个集合的交集和并集
2. numbers为sorted_set 参数的个数
3. 返回交集元素或者并集元素个数
4. 集合元素的分值是由两个集合分值的和
5. `[AGGREGATE SUM/MIN/MAX]` 可以通过设置聚合函数来控制分值(求和/最小值/最大值)
6. `[WEIGHTS w1 w2 w3]` 可以为每个集合设置权重，这样计算方式将会是分值乘以权重再相加
7. 除此之外，还可以接受集合（非有序）来执行命令，此时score默认都是1，还可以带WEIGHTS

### ZEANGEBYLEX/ZREVRANGEBYLEX/ZLEXCOUNT/ZREMRANGEBYLEX 系列命令

1. 以上所有命令格式雷同，都是处理当有序集合中分数全部相当的情况min和max分别指定是字典字母
2. `ZEANGEBYLEX sorted_set min max`
3. 比如`ZEANGEBYLEX sorted_set - +`返回所有成员，`[a (t` 返回字典大于等于a并且小于t的所有成员
4. 逆序/成员个数/移除操作都是类似的

### ZPOPMAX/ZPOPMIN 命令

1. 弹出分值最高或者最低分值的元素
2. 返回成员和分值
3. `ZPOPMAX sorted_set [count]`可以通过 count 来指定最多移除的成员数量，默认为1
4. Redis5.0  以上版本才支持

### BZPOPMIN/BZPOPMAX 命令

1. 阻塞式的最小或最大的弹出操作, 可以接受多个集合参数，进行遍历检测
2. `BZPOPMIN/BZPOPMAX sorted_set [sorted_set ] timeout`
3. timeout 为 0 表示无限阻塞等待

## HperLogLog

### HperLogLog数据结构

1. 神奇的HyperLogLog算法<http://www.rainybowe.com/blog/2017/07/13/%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog%E7%AE%97%E6%B3%95/index.html>
2. Sketch of the Day: HyperLogLog — Cornerstone of a Big Data Infrastructure <http://content.research.neustar.biz/blog/hll.html>
3. 这是一个专门解决大数据计数器消耗太多内存问题的一个概率算法，只需要12k就可以统计2^64个元素
4. 当然这不是精确统计，存在误差，数据量大的时候误差有的时候是允许的，可容允的

### PFADD 命令

1. `PFADD hperloglog element [element]` 新增元素
2. 当新增元素是的统计基数值发生变化就返回1，否则反正0

### PFCOUNT 命令

1. `PFCOUNT hyperloglog [hyperloglog ...]` 计算集合的近视基数
2. 当参数为多个时候，计算方式为：首先求多个集合的并集，然后对并集求近视基数

### PFMERGE 命令

1. `PFMERGE destination hyperloglog [hyperloglog ...]`  对多个hyperloglog集合求并集，然后将结果存在dest中

2. PFCOUNT 其实是有调用PFMERGE命令的

## 位图

### 位图结构

1. Redis位图bitmap是由多个二进制位组成的数组，数组中每一位都有与之对应的偏移量(索引)

2. BITMAP 图

|index|0|1|2|3|
|-|-|-|-|-|
|位|1|0|0|1｜

### SETBIT 命令

1. `SETBIT bitmap offset value` 设置指定偏移位的值
2. 返回指定偏移量旧值，默认为0
3. bitmap默认按照字节扩展
4. offset只能为正值

### GETBIT 命令

1. `GETBIT bitmap offset`获取指定位置的值

### BITCOUNT 命令

1. `BITCOUNT bitmap` 统计位图中1的个数
2. `BITCOUNT bitmap start end`  返回指定字节范围内1的个数，注意start和end为字节偏移量，并不是位offset， 可以使用负数作为参数

### BITPOS 命令

1. `BITPOS bitmap value` 查询bitmap中第一个被设置为value值的位置
2. `BITPOS bitmap value [start end]`  在指定范围内查找，但是返回的offset是基于整个bigmap的偏移
3. start和end可以为负值

### BITOP 命令

1. `BITOP OP result_key bitmap [bitmap ...]` 对多个bitmap数组执行op操作，将结果存储在result中
2. op可以是 AND / OR /XOR / NOT

### BITFIELD 命令

1. `BITFIELD bitmap SET type offset value` 根据位偏移来设置bitmap中值value，其中type是指定value的类型，比如i8：8位有符号，u16：16位无符号等
2. offset 可以换成 #index， 这样可以以字节位来索引具体位置，然后设置值
3. 可以同时执行多个set命令
4. `BITFIELD bitmap GET type offset/#index` 获取对应的值，同样的也可以同时执行多个GET
5. `BITFIELD bitmap INCRYBY type offset/#index increment` 对指定范围值加减操作
6. `BITFIELD bitmap [OVERFLOW WRAP/SAT/FAIL] INCRYBY type offset/#index increment` 可以用来处理加减法结果溢出的情况，分别为环绕/饱和运算/失败

### BITMAP STRING

1. 可以把二进制数组当作是字符串来操作
2. GET 命令来获取二进制数组值，返回值为二进制字符串
3. STRLEN 可以得到二进制字符串的长度
4. GETRANGE 获取指定范围的二进制字符串

## GEO位置服务

### GEOADD 命令

1. `GEOADD location_set longitude latitude name [longitude latitude name]` 添加一个或者多个位置坐标（经纬度）
2. 当执行的是添加的，那么返回添加的位置个数；如果是更新那么返回0

### GEOPOS 命令

1. `GEOPOS location_set name [name ...]`  获取指定位置的经纬度
2. 返回值是数组，其中数组元素为二元数组，第一项为经度，第二项为纬度

### GEODIST 命令

1. `GEODIST location_set name1 name2` 计算俩个位置的直线距离
2. 默认单位为米，可以通过`[unit]` 来指定单位m/km/mi英里/ft英寸

### GEORADIUS 命令

1. `GEORADIUS location_set longitude latitude radius unit`  获取指定位置为中心点，radius半径内所有的地点
2. `WITHDIST` 加上这个后缀参数，可以返回地点和地点与中心位置的直线距离
3. `WITHCOORD`  返回地点和地点坐标
4. `[ASD|DESC]` 对返回的结果排序
5. `[COUNT n]`  限制返回地点的数量
6. 可以同时指定多个可选参数

### GEORADIUSBYMEMBER 命令

1. `longitude latitude` 参数换成 `name`地名

### GEOHASH 命令

1. 获取指定位置的GEOhash值，GEOhash值是经纬度转换而来，并且可以通过GEohash来计算得到经纬度
2. 在上面的两个命令中都可以指定`WITHHASH`  来返回GEOHASH 而非 经纬度

### GEO数据内部存储结构

1. 为有序集合，因此可以使用ZSORTED来操作数据，其中score为Geohash值

## Stream流

