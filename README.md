# 分代索引(Generational indices)算法
> https://lucassardois.medium.com/generational-indices-guide-8e3c5f7fd594
记录一下自己的推导

## 问题: HashMap Vs Vector
经常遇到这样的需求: 
- 有一组数据, 其Element类型是T
- T的key是自定义的, 可以是字符串, 也可以是数字
- Collection\<T\>作为底层数据集, 每个Element都可以被上层数据集引用
- Collection\<T\>可以实现CRUD
通常, 为了方便, 我会直接将Collection看成HashMap, 确实可以带来很大方便\
但由于key其实完全可以自定义, 完全可以使用自增id(u32), 使用HashMap, 无论时间还是空间都并不是最佳方案

第一选择自然是Vector, 但立即遇到如下两个问题: 
1. 当删除Element时, 需要同步删除上层数据集的引用, 否则引用会悬空\
解决: 将Element数据类型设为 Option\<T\>, 并不删除上层数据集引用\
此方案可行

2. 先删除index=1处的Element, 再在Vector尾部insert, 会造成空间浪费
对于频繁、批量的删除操作, 这会造成Vector内部大量空洞(None)\
解决: 我们可以记下这些空洞的位置, 然后每次insert时直接填充空洞\
此方案又遇到以下问题(ABA问题):\
假设外部引用指向index=1, 删除此值后变为None, 再插入新值, 显然引用指向的值是错的\
解决此问题, 需要引用 generation 概念\

## 第一次尝试: 引用generation概念
```rust
struct GenVec {
  data: Vec<GenEntry<T>>,
  len: u32,
}

struct GenEntry<T> {
  value: Option<T>,
  generation: u32,
}

// 每次insert都返回key
struct Key {
  index: usize,
  pub generation: u32,
}
```
1. 第一次insert 100
```
data: [{value: Some(100), generation: 0}, len: 1]
返回 key{index:0, generation: 0}
```

2. 第二次insert 200
```
data: [
  {value: Some(100), generation: 0},
  {value: Some(200), generation: 0}
  , len: 2]
返回 key{index:1, generation: 0}
```

3. 第一次remove key{index:0,generation:0}
```
data: [
  {value: None, generation: 1},  // generation++
  {value: Some(200), generation: 0}
  , len: 2]
返回 key{index:1, generation: 0}
```

4. 此时get key{index:0,generation:0}, 应返回None
5. 此时get key{index:0,generation:1}, 应返回Some(100)

注意generation何时+1?\
如果在insert时+1, 则generation的初值为1, 不美观\
所以这里改成了remove时+1, 从而generation的初值为0\

貌似完美\

但: \
6. 第3次insert 300 
此时需要先找到空洞, 即 index = 0\
在当前的数据结构下, 需要对data从0开始遍历, 对于百万级的数组, 插入操作时间复杂度太高, 也抵消了相对于HashMap的好处\

## 第二次尝试: 维护一个空洞队列
```rust
struct GenVec {
  data: Vec<GenEntry<T>>,
  holes: Vec<u32>, // 删除时, 将删除的索引保存在此处
  len: u32,
}

struct GenEntry<T> {
  value: Option<T>,
  generation: u32,
}

// 每次insert都返回key
struct Key {
  index: usize,
  pub generation: u32,
}
```
这个方案有个小问题:\
删除index时, 在holes里保存index时, 要避免重复保存\
插入index时, 在holes里需要删除index(如果存在)\

这两种情况都可能需要遍历holes, 如果holes很长, 明显不合理\
根本原因是: holes与elements不满足"单一数据源"\

## 第三次尝试: 利用data内的空洞元素, 就地形成链表
```
比如: {
  data: [100, 200,300,400,500],
  free_head: 5
}

删除1: {
  data: [100, {next_free: 5}, 300, 400,500],
  free_head: 1,
},

删除3: {
  data: [100, {next_free: 5}, 300, {next_head: 1},500],
  free_head: 3,
},

形成的链表是: 3->1->5

而且是在删除过程中自然形成的
注意free_head始终保持最新删除的元素index, 链表顺序是 新->旧
(如果链表顺序是旧->新, 会导致很多麻烦)
```

新的数据结构:\
```rust
struct GenVec {
  data: Vec<GenEntry<T>>,
  free_head: u32, //最新删除元素的index, 也是holes链表的表头
  len: u32,
}

// 将entry改成enum
enum Entry<T> {
  Free{next_free: usize},
  Occupied{value:T}
}

struct GenEntry<T> {
  value: Entry<T>, // 如果采用Option<T>, 则None无法记录next_free, 只好改成enum
  generation: u32,
}

// 每次insert都返回key
struct Key {
  index: usize,
  pub generation: u32,
}

```

算法见代码