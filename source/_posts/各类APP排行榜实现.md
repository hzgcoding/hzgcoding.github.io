---
title: 各类APP排行榜实现
date: 2020-01-23 15:12:45
tags:
---

## 需求背景
1. 查看TopN的用户排名
2. 查看自己的排名
3. 用户积分变更后，排名及时更新

## 实现方案
### 方案一 利用MYSQL 排序
利用MySQL来实现，存放一张用户积分表user_score
取前top N，自己的排名都可以通过简单的sql语句搞定。
算法简单，利用sql的功能，不需要其他复杂逻辑，对于数据量比较少、性能要求不高，可以使用。但是对于海量数据，性能是无法接受的。可能会导致全局锁表之类的问题。

### 方案二 内存数组排序
在内存中预分配所要排名用户大小的数组，所有的积分排名变更基于此移动元素，成熟排序算法有最小/大堆、快速排序等，这种方案优点是当数据量小的时候，简单快捷，容易实现，不需要其他任何组件支持，但是当面对海量数据的时候数组空间占用可能不太现实

### 方案三 利用GCC库支持
具体的，用GCC的pb_ds库中有assoc_container来进行实现。
参考[tree_order_statistics.cc](https://opensource.apple.com/source/llvmgcc42/llvmgcc42-2336.9/libstdc++-v3/testsuite/ext/pb_ds/example/tree_order_statistics.cc)：
```cpp
#include <cassert>
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>

using namespace std;
using namespace pb_ds;
using namespace pb_ds;

// A red-black tree table storing ints and their order
// statistics. Note that since the tree uses
// tree_order_statistics_node_update as its update policy, then it
// includes its methods by_order and order_of_key.
typedef
tree<
  int,
  null_mapped_type,
  less<int>,
  rb_tree_tag,
  // This policy updates nodes' metadata for order statistics.
  tree_order_statistics_node_update>
set_t;

int main()
{
  // Insert some entries into s.
  set_t s;
  s.insert(12);
  s.insert(505);
  s.insert(30);
  s.insert(1000);
  s.insert(10000);
  s.insert(100);

  // The order of the keys should be: 12, 30, 100, 505, 1000, 10000.
  assert(*s.find_by_order(0) == 12);
  assert(*s.find_by_order(1) == 30);
  assert(*s.find_by_order(2) == 100);
  assert(*s.find_by_order(3) == 505);
  assert(*s.find_by_order(4) == 1000);
  assert(*s.find_by_order(5) == 10000);
  assert(s.find_by_order(6) == s.end());

  // The order of the keys should be: 12, 30, 100, 505, 1000, 10000.
  assert(s.order_of_key(10) == 0);
  assert(s.order_of_key(12) == 0);
  assert(s.order_of_key(15) == 1);
  assert(s.order_of_key(30) == 1);
  assert(s.order_of_key(99) == 2);
  assert(s.order_of_key(100) == 2);
  assert(s.order_of_key(505) == 3);
  assert(s.order_of_key(1000) == 4);
  assert(s.order_of_key(10000) == 5);
  assert(s.order_of_key(9999999) == 6);

  // Erase an entry.
  s.erase(30);

  // The order of the keys should be: 12, 100, 505, 1000, 10000.
  assert(*s.find_by_order(0) == 12);
  assert(*s.find_by_order(1) == 100);
  assert(*s.find_by_order(2) == 505);
  assert(*s.find_by_order(3) == 1000);
  assert(*s.find_by_order(4) == 10000);
  assert(s.find_by_order(5) == s.end());

  // The order of the keys should be: 12, 100, 505, 1000, 10000.
  assert(s.order_of_key(10) == 0);
  assert(s.order_of_key(12) == 0);
  assert(s.order_of_key(100) == 1);
  assert(s.order_of_key(505) == 2);
  assert(s.order_of_key(707) == 3);
  assert(s.order_of_key(1000) == 3);
  assert(s.order_of_key(1001) == 4);
  assert(s.order_of_key(10000) == 4);
  assert(s.order_of_key(100000) == 5);
  assert(s.order_of_key(9999999) == 5);

  return 0;
}
```
存取效率都可以达到O(log(n))，不足就是程序重启后数据会丢失。还是对所有的用户积分，没必要。
而且是有依赖，不方便扩展，实现不了复杂的需求，

### 方案四 实现排序树

大致实现思路如下：


　　我们可以把[0, 1,000,000)作为一级区间；再把一级区间分为两个2级区间[0, 500,000), [500,000, 1,000,000)，然后把二级区间二分为4个3级区间[0, 250,000), [250,000, 500,000), [500,000, 750,000), [750,000, 1,000,000)，依此类推，最终我们会得到1,000,000个21级区间[0,1), [1,2) … [999,999, 1,000,000)。这实际上是把区间组织成了一种平衡二叉树结构，根结点代表一级区间，每个非叶子结点有两个子结点，左子结点代表低分区间，右子结点代表高分区间。树形分区结构需要在更新时保持一种不变量，非叶子结点的count值总是等于其左右子结点的count值之和。


　　以后，每次用户积分有变化所需要更新的区间数量和积分变化量有关系，积分变化越小更新的区间层次越低。总体上，每次所需要更新的区间数量是用户积分变量的log(n)级别的，也就是说如果用户积分一次变化在百万级，更新区间的数量在二十这个级别。在这种树形分区积分表的辅助下查询积分为s的用户排名，实际上是一个在区间树上由上至下、由粗到细一步步明确s所在位置的过程。比如，对于积分499,000，我们用一个初值为0的排名变量来做累加；首先，它属于1级区间的左子树[0, 500,000)，那么该用户排名应该在右子树[500,000, 1,000,000)的用户数count之后，我们把该count值累加到该用户排名变量，进入下一级区间；其次，它属于3级区间的[250,000, 500,000)，这是2级区间的右子树，所以不用累加count到排名变量，直接进入下一级区间；再次，它属于4级区间的…；直到最后我们把用户积分精确定位在21级区间[499,000, 499,001)，整个累加过程完成，得出排名！


　　虽然，本算法的更新和查询都涉及到若干个操作，但如果我们为区间的from_score和to_score建立索引，这些操作都是基于键的查询和更新，不会产生表扫描，因此效率更高。另外，本算法并不依赖于关系数据模型和SQL运算，可以轻易地改造为NoSQL等其他存储方式，而基于键的操作也很容易引入缓存机制进一步优化性能。进一步，我们可以估算一下树形区间的数目大约为2,000,000，考虑每个结点的大小，整个结构只占用几十M空间。所以，我们完全可以在内存建立区间树结构，并通过user_score表在O(n)的时间内初始化区间树，然后排名的查询和更新操作都可以在内存进行。一般来讲，同样的算法，从数据库到内存算法的性能提升常常可以达到10^5以上；因此，本算法可以达到非常高的性能。


　　算法特点


　　优点：结构稳定，不受积分分布影响；每次查询或更新的复杂度为积分最大值的O(log(n))级别，且与用户规模无关，可以应对海量规模；不依赖于SQL，容易改造为NoSQL或内存数据结构。


　　缺点：算法相对更复杂。

### 方案五 实现跳表排序

skip list是链表的一种特殊形式，对链表的一种优化；保证INSERT和REMOVE操作是O(logn)，而通用链表的复杂度为O(n);
优点：实现较简单，效率基本上O(log(N))
缺点：当达到亿级别时的数据时，性能会急剧下降

### 方案六 利用redis特新实现
其实redis底层还是使用跳表实现排序的，只是将接口都封装好了，使用接口也比较完善，稳定。

redis的zset天生是用来做排行榜的、好友列表, 去重, 历史记录等业务需求。接口使用非常简单。接口非常丰富，基本上需要的实现都能满足，说明如下：

ZAdd/ZRem是O(log(N))，ZRangeByScore/ZRemRangeByScore是O(log(N)+M)，N是Set大小，M是结果/操作元素的个数。

ZSET的实现用到了两个数据结构：hash table 和 skip list(跳跃表)，其中hash table是具体使用redis中的dict来实现的，主要是为了保证查询效率为O(1) ，而skip list(跳跃表)主要是保证元素有序并能够保证INSERT和REMOVE操作是O(logn)的复杂度。

优点：基于redis开发，速度快；使用redis相关特性

缺点：当达到亿级别时的数据时，性能会急剧下降

来实现排行榜的方法很多，可以根据自己的具体需求，参考选用。

## 方案七 其实只需要TopN的排名，大于N的排名并不需要精确排名计算

基于此，假设我们游戏内只需要排前100名，这里我们只需要维护一个100大小的数组
1. 当元素A需要参与排序的时候，与数组中最小的积分进行比较，如果能进100名，那么将第100剔除，将A加入，并记录最小元素，这样就完成了积分上涨的情况
2. 还有一种就是已经在前100名中元素的积分发生变化下降，那么需要在前100名后找出可以进排行榜的元素，这种情况比较麻烦，可以使用最大堆保存剩下所有用户的数据，当需要找出能替换进入排行榜的元素就非常快logn，选择特定数据结构非常重要
3. 既然大于N的用户不需要精确排名，那么怎么样估算大概排名呢？一般做法是按照数值区间建立所若干个桶，比如我们预计要排名的那一个数据的最大值能到1W。我建立0-10， 10-100，100-1000， 1000-2000， 2000-5000， 5000-10000 这样6个桶，每个桶里面记录分值在这个桶对应的区间内，有多少个玩家。 比如
0-10， 10人
10-100，20人
100-1000，30人
1000-2000， 40人
2000-5000， 50人
5000-10000， 60人
那么如果一个玩家 是 1234分，那么他的排名就超过了 （10 + 20 + 30）/ （10 + 20 + 30 + 40 + 50 + 60）这个百分比的玩家（所以桶分的越细，后面的排名越精确）
实质就是按照分区间记录区间内元素个数，从而估算大概排名，因此数值区间越小，估算约精确。


