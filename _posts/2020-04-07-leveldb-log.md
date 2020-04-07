---
layout: post
title: LevelDB Log
date: 2020-04-07 19:27
categories: 数据库
tags: [LevelDB, source code]
---

# 任务

# 学习

## 概念
log文件在LevelDB中的是 WAL(write-ahead-log), 主要作用是系统故障恢复时, 能够保证不丢失数据, 因为将记录写入内存的memtable前面会先写入log文件, 这样即使系统故障, memtable中的数据没有及时dump到磁盘的SSTable文件中, LevelDB也可以根据log文件(redo log)恢复内存的memtable数据结构的内容, 防止数据丢失.

### 作用

每执行一次`put()`或者`delete()`，都会产生一条record保存在log文件中, 写操作流程:
先成功地append到磁盘上的log日志中 -> 再更新内存memtable -> skiplist查找位置并插入key-value.
这样做的优点是:

1. 将随机写IO变为顺序append, 提高写磁盘速度;
2. 防止在节点down机导致内存数据丢失，造成数据丢失，这对系统来说是个灾难。

<!--more-->

### 格式

LevelDB 对于一个 log 文件，会把它切割成以 **32K** 为单位的**物理 Block**

- 每次读取都是以一个 Block 作为基本单位
- 写日志时, 出于一致性考虑, 没有以block为单位写, 即每次更新都对log文件进行IO, 每次更新写入作为一个record -> 避免写放大

下图展示的 log 文件由 3 个 Block 构成，所以从物理布局来讲，一个 log 文件就是由连续的 32K 大小 Block 构成的.

![log-block-record](/../../media/2020-04-07-leveldb-log/log-block-record.png)

 每条record的结构如下图

![record-structure](/../../media/2020-04-07-leveldb-log/record-structure.png)

1. **checksum**

   占4个字节, 记录的是 “type” 和 “data” 字段的CRC校验，为了避免处理不完整或者是被破坏的数据，当 LevelDB 读取记录数据时候会对数据进行校验，如果发现和存储的 checksum 相同，说明数据完整无破坏，可以继续后续流程。

2. **length**

   记录record 内保存的 data 长度(小端对齐, [变长编码](https://blog.csdn.net/H514434485/article/details/104910650))，2个字节

3. **type**

   指出了每条记录的逻辑结构和 log 文件物理分块结构之间的关系:

   - FULL, 表示当前记录完整存储在一个物理block中
   - FIRST, 表示这是一条记录的第一部分
   - MIDDLE，一条记录的中间部分。
   - LAST，一条记录的最后一部分。

4. **data**

   记录的是 Key:Value 数值对.

**Eg.** 以图log-block-record为例子具体说明, record的type:

- 目前存在3条record, 分别是 A(10K) , B(80K) , C(12K) , log的逻辑布局如上图所示;
- A 是图中蓝色区域, 因为 10K < 32K ,  A 能存储在一个物理block中, 所以type为FULL;
- B 是图中黄色区域, 
  - block 1 因为放入了 A, 还剩下22K 不足以放下 B, 但能放下 header(7Byte), 所以block的剩余部分放入 B 的开头一部分, type为FIRST;
  - B 还有 58K 没有存储, 只能依次放在 block2 中, 但 block2 只有 32K, 仍然放不下 B 的剩余部分, 所以 block 2 全部用来放 B, type为 MIDDLE;
  - B 剩余 26K 能完全放入 block3 中, type为LAST
- C 大小只有 12K , block3 的余下空间完全能放下C, 所以type为FULL.

从这个小例子可以看出逻辑记录和物理Block之间的关系，LevelDB一次物理读取为一个Block，然后根据类型情况拼接出逻辑记录，供后续流程处理。

总体构造格式如图: 

![log-block-record-construct](/../../media/2020-04-07-leveldb-log/log-block-record-construct.png)



## 源码

日志文件的切换是在写KV记录之前会进行`MakeRoomForWrite`来决定是否切换新的日志文件，所以在写入的过程中是不需要关注文件切换的。接下来介绍Log模块的读写流程及结构。

### 文件结构

- log_format.h：描述Log格式及Record类型。
- log_reader.h、log_reader.cc：读模块实现。
- log_writer.h、log_writer.cc：写模块实现。



### 格式信息



1. 一共有四种Record类型。
2. 每个Block为32KB
3. 每个Record头大小为4 + 2 + 1 = 7个字节。

```c
namespace log {

enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,

  kFullType = 1,

  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
static const int kMaxRecordType = kLastType;

static const int kBlockSize = 32768;

// Header is checksum (4 bytes), length (2 bytes), type (1 byte).
static const int kHeaderSize = 4 + 2 + 1;

}  // namespace log
```



### 写流程

#### 类关系图

![writer-class-relation](/../../media/2020-04-07-leveldb-log/writer-class-relation.png)

#### 分析

log_writer.cc 中`Writer::AddRecord()`, 对于 log 的每次写入作为 record 添加:

- 如果当前 block 剩余的 size 小于 recorder header 长度, 填充 trailer(即`\x00`), 开始下一个block
- 根据当前 block 剩余size 和写入 size,  确定 record type
- 写入 record, (`Writer::EmitPhysicalRecord()`)
- 循环 1-3, 直到写入操作完成
- 根据 option 指定的sync决定是否做 log 文件的强制 sync



####  源码



##### log_writer.h



```c
// Copyright (c) 2011 The LevelDB Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file. See the AUTHORS file for names of contributors.

#ifndef STORAGE_LEVELDB_DB_LOG_WRITER_H_
#define STORAGE_LEVELDB_DB_LOG_WRITER_H_

#include <stdint.h>

#include "db/log_format.h"
#include "leveldb/slice.h"
#include "leveldb/status.h"

namespace leveldb {

class WritableFile;

namespace log {

class Writer {
 public:
  /// 实例一个Writer，传入的参数*dest要为空，且在写期间，*dest要保持存活
  // Create a writer that will append data to "*dest".
  // "*dest" must be initially empty.
  // "*dest" must remain live while this Writer is in use.
  explicit Writer(WritableFile* dest);

  // Create a writer that will append data to "*dest".
  // "*dest" must have initial length "dest_length".
  // "*dest" must remain live while this Writer is in use.
  Writer(WritableFile* dest, uint64_t dest_length);

  Writer(const Writer&) = delete;
  Writer& operator=(const Writer&) = delete;

  ~Writer();
  /// 写一个record到文件中
  Status AddRecord(const Slice& slice);

 private:
 /// 实际写
  Status EmitPhysicalRecord(RecordType type, const char* ptr, size_t length);

  /// log文件(用来写入磁盘)
  WritableFile* dest_;
  /// 位于当前block的哪个位置
  int block_offset_;  // Current offset in block

  /// 提前计算好的type对应CRC值, 减少使用过程中的计算
  // crc32c values for all supported record types.  These are
  // pre-computed to reduce the overhead of computing the crc of the
  // record type stored in the header.
  uint32_t type_crc_[kMaxRecordType + 1];
};

}  // namespace log
}  // namespace leveldb

#endif  // STORAGE_LEVELDB_DB_LOG_WRITER_H_
```



##### log_writer.cc



```c
// Copyright (c) 2011 The LevelDB Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file. See the AUTHORS file for names of contributors.

#include "db/log_writer.h"

#include <stdint.h>

#include "leveldb/env.h"
#include "util/coding.h"
#include "util/crc32c.h"

namespace leveldb {
namespace log {

/// 计算RecordType的CRC32值
static void InitTypeCrc(uint32_t* type_crc) {
  for (int i = 0; i <= kMaxRecordType; i++) {
    char t = static_cast<char>(i);
    type_crc[i] = crc32c::Value(&t, 1);
  }
}

Writer::Writer(WritableFile* dest) : dest_(dest), block_offset_(0) {
  InitTypeCrc(type_crc_);
}

Writer::Writer(WritableFile* dest, uint64_t dest_length)
    : dest_(dest), block_offset_(dest_length % kBlockSize) {
  InitTypeCrc(type_crc_);
}

/// 指定默认析构函数
Writer::~Writer() = default;

/* 
  1. block: 32KB, record header: 7Byte
  2. 写record流程:
     假设block剩余可写长度为leftover, 要写入的data长度为left, 分以下情况:
     a. leftover >= left + 7, 则data和header都可被放入block, record Type定义为kFullType
     b. left + 7 >= leftover >= 7, 则header可放入block, data不可. 则:
      - 在当前页生成一个Type为kFirstType的record, block剩余空间保存data前面(leftover-7)字节的内容, 可为0 
      - 在下一个blcok中, 如果data剩余长度小于32KB,则生成Type为kLastType的record, 否则生成Type
        为kMiddleType的record
      - 依此类推, 直到data被完全保存.
    c. leftover < 7, 在当前block中填充0
*/
Status Writer::AddRecord(const Slice& slice) {
  /// 添加记录数据
  const char* ptr = slice.data();
  /// 记录记录的数据部分的长度
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  /* 
    如果slice数据为空，仍然会写一次，只是长度为0，
    读取的时候会对此种情况进行处理。
  */
  Status s;
  bool begin = true;
  do {
    /// 记录当前块剩余容量
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0);
    /// 如果剩余容量小于record header长度(7KB)
    if (leftover < kHeaderSize) {
      // Switch to a new block
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        static_assert(kHeaderSize == 7, "");
        /// 用"0"填充
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      /// 开始一个新块, 块内偏移置零
      block_offset_ = 0;
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    /// 记录当前块除了记录头还有的剩余空间
    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;

    /// 判断块能否容下当前记录
    /// kzerotype: 说明块的内容不对
    RecordType type;
    const bool end = (left == fragment_length);
    if (begin && end) {
      type = kFullType;
    } else if (begin) {
      type = kFirstType;
    } else if (end) {
      type = kLastType;
    } else {
      type = kMiddleType;
    }

    /// 将fragment_length长度的数据写入log
    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    /// 记录数据剩余长度
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);
  return s;
}

/// 实际写实现
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr,
                                  size_t length) {
  assert(length <= 0xffff);  // Must fit in two bytes
  assert(block_offset_ + kHeaderSize + length <= kBlockSize);

  // Format the header
  /*
    CheckSum: CRC验证码, 4个字节
    length: record长度, 2个字节
    type: record类型(log record和block的关系),1个字节
  */
  char buf[kHeaderSize];
  /// 取length低八位
  buf[4] = static_cast<char>(length & 0xff);
  /// 取length第9位到第16位
  buf[5] = static_cast<char>(length >> 8);
  /// 设置RecordType
  buf[6] = static_cast<char>(t);

  /// 用CRC填充buf前4个字节
  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, length);
  crc = crc32c::Mask(crc);  // Adjust for storage, 高位和低位颠倒一下
  EncodeFixed32(buf, crc);

  /// 将header写入缓存
  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  /// 将记录内容写入缓存
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, length));
    /// 将缓存数据刷新进内核
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  /// 更新块内偏移量
  block_offset_ += kHeaderSize + length;
  return s;
}

}  // namespace log
}  // namespace leveldb
```

### 读流程

#### **类关系图**

![reader-class-relation](/../../media/2020-04-07-leveldb-log/reader-class-relation.png)

#### 源码

##### log_reader.h

##### log_reader.cc

# 参考链接

1. [【leveldb】Log（五）](https://blog.csdn.net/H514434485/article/details/103899828)
2. [leveldb源码分析之读写log文件](http://luodw.cc/2015/10/18/leveldb-08/)
3. [Leveldb: Log](https://wiesen.github.io/post/leveldb-storage-log/)
4. [LevelDB: Read the Fucking Source Code](http://www.grakra.com/2017/06/17/Leveldb-RTFSC/)