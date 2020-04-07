---
layout: post
title: LevelDB SkipList
date: 2020-04-02 01:49 +0800
categories: 数据库
tags: LevelDB
---

# 任务

1. 在skiplist.h中写入函数声明:

![image-20200326160557408](/../../media/2020-04-02-leveldb-skiplist/20200326160553864.png)

# 学习

1. LevelDb的Memtable中KV对是根据**Key大小有序存储**
   - 为了支持扫描操作
   - 无法使用Hash table
   - B+树与SkipList性能没有定论
2. LevelDb的Memtable类只是一个接口类，真正的操作是通过背后的**SkipList**来做的
   - 内存索引Memtable的核心数据结构是SkipList, 作用就是解决KV的快速插入和查询。

3. SkipList（跳表）数据结构是用于Memtable，Immutable Memtable表中，对于此二表的作用点此查看[Memtable作用](https://blog.csdn.net/H514434485/article/details/103546949)。

<!--more-->

## 一. 介绍

SkipList使用空间换时间的设计思路，通过构建多级索引来提高查询的效率，实现了基于链表的“二分查找”。
复杂度如下：

- 插入、删除、查找的时间复杂度都是O(logn)
- 空间复杂度是O(n)。

## 二. 结构

? Leveldb实现的SkipLisp初始化时head部分是12个指针点
![SkipList结构](/../../media/2020-04-02-leveldb-skiplist/2020030716464661.png)

## 三. 源码

### Node

```c
// Implementation details follow
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node { 
  explicit Node(const Key& k) : key(k) {}

  Key const key;
  /// 有内存屏障操作, 获取当前节点在指定level的下一个节点
  // Accessors/mutators for links.  Wrapped in methods so we can
  // add the appropriate barriers as necessary.
  Node* Next(int n) {
    assert(n >= 0);
    // Use an 'acquire load' so that we observe a fully initialized
    // version of the returned Node.
    return next_[n].load(std::memory_order_acquire);
  }

  /// 将当前节点在指定level的下一个节点设置为x
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.
    next_[n].store(x, std::memory_order_release);
  }

  /// 无内存屏障操作，相比于无内存屏障操作，性能损耗更小
  // No-barrier variants that can be safely used in a few locations.
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }

 private:
  /*
    next_[1]: 当前节点的每个等级的下一个结点
	  第2级 N1 N2
	  第1级 N1 N2
	  如果N1是本节点，则next_[x] 保存的是N2
	  next_[0]就是原始链表。

    atomic types are those for which reading and 
    writing are guaranteed to happen in a single instruction.
  */
  // Array of length equal to the node height.  next_[0] is lowest level link.
  std::atomic<Node*> next_[1]; /// 大小是一个Node
};

```

### SkipList

```c
/// SkipList
/// 内存管理
class Arena;

template <typename Key, class Comparator>
class SkipList {
 private:
  struct Node;

 public:
  // Create a new SkipList object that will use "cmp" for comparing keys,
  // and will allocate memory using "*arena".  Objects allocated in the arena
  // must remain allocated for the lifetime of the skiplist object.
  explicit SkipList(Comparator cmp, Arena* arena);

  SkipList(const SkipList&) = delete;
  SkipList& operator=(const SkipList&) = delete;

  // Insert key into the list.
  // REQUIRES: nothing that compares equal to key is currently in the list.
  void Insert(const Key& key);

  // Returns true iff an entry that compares equal to key is in the list.
  bool Contains(const Key& key) const;

  /// 迭代器，英文注释可直接看
  // Iteration over the contents of a skip list
  class Iterator {
   public:
    // Initialize an iterator over the specified list.
    // The returned iterator is not valid.
    explicit Iterator(const SkipList* list);

    // Returns true iff the iterator is positioned at a valid node.
    bool Valid() const;

    // Returns the key at the current position.
    // REQUIRES: Valid()
    const Key& key() const;

    // Advances to the next position.
    // REQUIRES: Valid()
    void Next();

    // Advances to the previous position.
    // REQUIRES: Valid()
    void Prev();

    // Advance to the first entry with a key >= target
    void Seek(const Key& target);

    // Position at the first entry in list.
    // Final state of iterator is Valid() iff list is not empty.
    void SeekToFirst();

    // Position at the last entry in list.
    // Final state of iterator is Valid() iff list is not empty.
    void SeekToLast();

   private:
    /// 当前迭代器关联的SkipList
    const SkipList* list_;
    /// 当前迭代器所指向的值
    Node* node_;
    // Intentionally copyable
  };

 private:
  /// 调表的层数, 最底层是第0层
  enum { kMaxHeight = 12 };

  /// 获取当前跳表有多少层
  inline int GetMaxHeight() const {
    return max_height_.load(std::memory_order_relaxed);
  }

  /// 新建一个Node
  Node* NewNode(const Key& key, int height);

  /* 
    返回需要插入值的随机高度, 比如说4
    则第0-3层都需要插入对应的Node
  */
  int RandomHeight();
  
  /// 等值判断
  bool Equal(const Key& a, const Key& b) const { return (compare_(a, b) == 0); }

  /// 判断当前key是否在Node *n后面
  // Return true if key is greater than the data stored in "n"
  bool KeyIsAfterNode(const Key& key, Node* n) const;

  // Return the earliest node that comes at or after key.
  // Return nullptr if there is no such node.
  //
  // If prev is non-null, fills prev[level] with pointer to previous
  // node at "level" for every level in [0..max_height_-1].
  Node* FindGreaterOrEqual(const Key& key, Node** prev) const;

  // Return the latest node with a key < key.
  // Return head_ if there is no such node.
  Node* FindLessThan(const Key& key) const;

  // Return the last node in the list.
  // Return head_ if list is empty.
  Node* FindLast() const;

  // Immutable after construction
  Comparator const compare_;
  Arena* const arena_;  // Arena used for allocations of nodes

  /// 跳表第0层的头指针, 指向第一个元素
  /// 也称skiplist的前置哨兵节点
  /// (这两种解释我比较喜欢第一种...)
  Node* const head_;

  // Modified only by Insert().  Read racily by readers, but stale
  // values are ok.
  std::atomic<int> max_height_;  // Height of the entire list

  /// 用于产生随机数
  // Read/written only by Insert().
  Random rnd_;
};

/* 
  产生一个新Node: 键是key, height表示key存在于多少层,
  最底层是第0层, 所以直接new, 剩下的层就是 (height - 1) 个指针指示,
  此处通过Arena获取对齐的内存, 提高CPU的访问速度.
*/
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node* SkipList<Key, Comparator>::NewNode(
    const Key& key, int height) {
      /// 一个实际Node值, 其它都是指针
  char* const node_memory = arena_->AllocateAligned(
      sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
  /// 在已经分配好内存的node_memory上构造一个Node对象
  return new (node_memory) Node(key);
}

/// 迭代器构造, 迭代器都是基于第0层进行操作的
template <typename Key, class Comparator>
inline SkipList<Key, Comparator>::Iterator::Iterator(const SkipList* list) {
  list_ = list;
  node_ = nullptr;
}

/// 当前迭代器是否有效
template <typename Key, class Comparator>
inline bool SkipList<Key, Comparator>::Iterator::Valid() const {
  return node_ != nullptr;
}

template <typename Key, class Comparator>
inline const Key& SkipList<Key, Comparator>::Iterator::key() const {
  assert(Valid());
  return node_->key;
}

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Next() {
  assert(Valid());
  /// 迭代器都是操作的第0层的数据
  node_ = node_->Next(0); 
}

/// 当前节点的前一个节点
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Prev() {
  // Instead of using explicit "prev" links, we just search for the
  // last node that falls before key.
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
}

/// 定位到≥target的位置
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Seek(const Key& target) {
  node_ = list_->FindGreaterOrEqual(target, nullptr);
}

/// 定位到第0层第一个节点值
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::SeekToFirst() {
  /// 迭代器操作的都是第0层，head是一个无值的Node，只是一个指针。
  node_ = list_->head_->Next(0);
}

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::SeekToLast() {
  node_ = list_->FindLast();
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
}

/// 生成要随机插入的层高，比如4，那就是[0...3]都要插入
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  //利用随机数实现每次有4分之一的概率增长高度。
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}

/// 判断key是否在节点node之后
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::KeyIsAfterNode(const Key& key, Node* n) const {//if(n->key<key || n==nullptr)return true
  // null n is considered infinite
  return (n != nullptr) && (compare_(n->key, key) < 0);
}

/*
  找到≥key的Node, 从最高层开始:
  1. 如果未找到对应的Node, 则返回的next是null
  2. 如果prev非null, 则讲每一层最近小于key的Noden地址保存入prev[level]
*/
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                              Node** prev) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {
      // Keep searching in this list
      x = next;
    } else {
      /// prev和next的方向不同(即, 是指向上一个Node的指针?)
      if (prev != nullptr) prev[level] = x;
      if (level == 0) {
        return next;
      } else {
        // Switch to next list
        level--;
      }
    }
  }
}

/*
  查找最近小于key的Node, 从最高层查起
  如果未找到, 就返回head_
*/
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindLessThan(const Key& key) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    assert(x == head_ || compare_(x->key, key) < 0);
    Node* next = x->Next(level);
    if (next == nullptr || compare_(next->key, key) >= 0) {
      if (level == 0) {
        return x;
      } else {
        // Switch to next list
        level--;
      }
    } else {
      x = next;
    }
  }
}

/// 从最高层开始, 定位到最后一个元素
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node* SkipList<Key, Comparator>::FindLast()
    const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (next == nullptr) {
      if (level == 0) {
        return x;
      } else {
        // Switch to next list
        level--;
      }
    } else {
      x = next;
    }
  }
}

/// SkipList构造, 以及一些值的初始化
template <typename Key, class Comparator>
SkipList<Key, Comparator>::SkipList(Comparator cmp, Arena* arena)
    : compare_(cmp),
      arena_(arena),
      head_(NewNode(0 /* any key will do */, kMaxHeight)),
      max_height_(1),
      rnd_(0xdeadbeef) {
  for (int i = 0; i < kMaxHeight; i++) {
    head_->SetNext(i, nullptr);
  }
}

/// 插入key
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  // TODO(opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
  /// 找到大于等于key的Node x, 并且记录每一层最近小于等于key的Node, 存入prev中
  Node* prev[kMaxHeight];
  Node* x = FindGreaterOrEqual(key, prev);

  /// 要么未找到这样的Node, 要么找到的是和插入的值相等的Node
  // Our data structure does not allow duplicate insertion
  assert(x == nullptr || !Equal(key, x->key));
  
  /// 使用随机数获取该节点的插入高度
  int height = RandomHeight();
  
  /// 大于当前skiplist 最高高度的话，将超过的部分的前置节点赋值为head_
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (nullptr), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since nullptr sorts after all
    // keys.  In the latter case the reader will use the new node.
    max_height_.store(height, std::memory_order_relaxed);
  }

  /// 产生一个新Node
  x = NewNode(key, height);
  /* 
    以下插入过程就是移动指针，从这里我们也明白了为什么要有prev[kMaxHeight]了
    1. 先设置x的后置指针
    2. 再设置小于x的key的最近节点的后置指针
    3. 此顺序防止断链
  */  
  for (int i = 0; i < height; i++) {
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));//set next of x
    prev[i]->SetNext(i, x);//set pre of x
  }
}

/// 跳表内是否存在此key
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}

}  // namespace leveldb

#endif  // STORAGE_LEVELDB_DB_SKIPLIST_H_

```

## 四. 总结

1. SkipList的所有操作都是从上到下去执行。

2. 随机层数为什么是%4？因为第x层节点数是第x-1层的1/4，换一个角度每个元素出现在每层的概率就是1/4，这样通过%4来决定节点一共出现在多少层。
3. leveldb用模板方式实现的SkipList，这样更通用。
4. leveldb的SkipList未实现del操作是因为元素删除也是插入，删除某个Key的Value在 Memtable 内是作为插入一条记录实施的，但是会打上一个 Key 的删除标记，真正的删除操作是Lazy的，会在以后的 Compaction 过程中去掉这个KV。
5. leveldb为什么使用SkipList来实现数据的插入查询呢？为什么不是红黑树或其它数据结构？
   SkipList按照区间查找数据效率比较高，而且实现起来也不是太复杂。

# 参考文章

1. [【leveldb】SkipList（七）](https://blog.csdn.net/H514434485/article/details/104716897)