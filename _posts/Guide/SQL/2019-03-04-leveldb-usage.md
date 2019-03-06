---
layout: post
title: leveldb的入门级使用
data: March 5, 2019 10:35 PM
excerpt: leveldb是Google开源的一款KV存储引擎.实现巧妙,代码风格友好.之后有阅读源码的想法,现在首先开始使用吧!本篇便是一份简单的leveldb使用教程
categories:
- DataBase
tags:
- LevelDB
toc: true
comments: true
---

## Headers

LevelDB本质上只是C/C++的KV存储库, 那么对于一个库, 首先来浏览它的头文件
编译安装之后, 头文件目录会在`/usr/include/leveldb`下

| header | description|
|--------|------------|
|db.h|核心头文件,提供DB句柄|
|status.h|LevelDB所有操作返回状态的抽象|
|options.h|LevelDB所有操作的选项设置, 尤其包括ReadOption & WriteOption|
|env.h| LevelDB与操作系统底层的基类, 可继承后在Options.env中修改|
|filter_policy.h|LevelDB过滤器的基类, 可继承后在Options.filter_policy中设置|
|comparator.h|KV比较函数, 可进行继承后设置|
|slice.h|LevelDB中Key的默认类型, 是二进制安全的, 类似于SDS|
|iterator.h|LevelDB中进行迭代的迭代器|
|write_batch.h|LevelDB的原子操作设施|
|table.h|较低层次的Table设置, 一般不用|
|table.builder.h|同上|
|cache.h|缓存相关, 除性能调优外一般不用|
|c.h|LevelDB的C接口|
|dumpfile.h|进行文件Dump的函数, 仅含DumpFile()|

## leveldb::Open

对于一款数据库, 我们开始使用, 就要从打开数据库开始. leveldb的打开十分简单, 我们先上代码

```cpp
#include <cassert>
#include <leveldb/db.h>
#include <leveldb/status.h>

leveldb::DB *db;                              // 数据库操作句柄
leveldb::Options options;                     // 设置选项
options.create_if_missing = true;             // 不存在的情况下创建
leveldb::Status status = leveldb::Open(options, "/path/to/db", &db);    // Open操作
assert(status.ok());                          // 断言操作后的状态
```

上面的代码十分简明清晰. 我们打开一个数据库可以通过这样的步骤:
1. 创建句柄
2. 设置选项
3. 打开数据库
4. 断言 (非必须, 可以用)

至此, 我们之后都可以通过 `db` 这个数据库句柄进行数据库的操作

## leveldb::Options

`leveldb::Options`是与数据库的打开设置息息相关的. 我们可以从源码中窥知一二

```cpp
// leveldb/src/include/options.h

struct Options {
  bool create_if_missing;        // 类似于 O_CREATE 选项 default: false
  bool error_if_exists;          // 与上面的选项同时设置会报错 default: false
  bool paranoid_checks;
  Env* env;                      // leveldb与操作系统底层的交互, default: Env::default()
  Logger* info_log;              // 设置日志级别
  size_t write_buffer_size;      // 设置写缓冲区大小, 可用来调优 defualt: 4096KB
  int max_open_files;            // 最大打开文件数, 可用来调优  default: 1000
  Cache* block_cache;            // 设置Cache, 可以明显提高性能 defualt: false
  size_t block_size;             // 设置block块大小 defualt: 4MB
  int block_restart_interval;
  CompressionType compression;   // 使用的压缩算法, 可以配合Google开源压缩算法使用
  const FilterPolicy* filter_policy;   // 设置过滤器, 如Bloom Filter, default: NULL
  Options();                      // 默认构造函数
};
```

我们每次在打开数据库前, 可以通过特定的`leveldb::Options`进行数据库的设置, 从而进行调优
如果, 不进行设置, 可以直接使用临时对象, 使用默认配置即可. 如:
`auto status = leveldb::Open(leveldb::Options(), "path/to/db/", &db);`

## leveldb::Status

众所周知, Google的编码规范中, 不建议使用异常. [Google C++ 编码规范](https://zh-google-styleguide.readthedocs.io/en/latest/contents/)

但是, 去使用UNIX/WIN Syscall时的错误码吗? 又显得太过粗糙

leveldb提供了`leveldb::Status`进行状态检测, 如下:

```cpp
// leveldb/src/include/status.h

class Status {
 public:
  C-tor ...
  
  Status getter ...
  
  // 判断状态的各类函数
  bool ok() const { return (state_ == NULL); }
  bool IsNotFound() const { return code() == kNotFound; }
  bool IsCorruption() const { return code() == kCorruption; }
  bool IsIOError() const { return code() == kIOError; }

  // if(!status.ok()), 可以输出错误, 类似于std::exception::what()
  std::string ToString() const;

 private:
  // 保存状态的是字符串, state_[0..3]为长度, state_[4]为Code, state_[5...]为message
  const char* state_;

  // 设置的状态位, 1, 2, 3, 4, 5
  enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
  };

  // 获取Code
  Code code() const {
    return (state_ == NULL) ? kOk : static_cast<Code>(state_[4]);
  }

  ...
};
```

使用`leveldb::Status`可以方便我们进行状态判别, 方式值得学习

## leveldb::Close

那么,如何关闭leveldb的数据库呢? 上面已经提到句柄这个词了.

很简单 `delete db;`即可. 

问题来了, 为什么不使用RAII呢?
个人推测有这方面的原因:
1. 作者个人编码风格影响
2. Google手册中对新特性一般
3. 正经的原因: 暴露句柄也容易操作, 而且leveldb是底层的KV库, 使用这可以自行封装, 即使暴露原生指针也是可以的

## leveldb::Put/Get/Delete

说到最关键的地方了, 作为KV库, 最本质的功能来了: 存取, 修改, 删除功能
是下面的用法:

```cpp
//interface

Status Put(const WriteOptions& options, const Slice& key, const Slice& value);
Status Get(cosnt ReadOptions& options, const Slice& key, std::string *value);
Status Delte(const WriteOptions& options, const Slice& key);
```

```cpp
#include <leveldb/db.h>

... open a database

const slice/std::string key(...);
std::string value;
auto status = db->Put(leveldb::WriteOptions(), key, value);
if (status.ok()) status = db->Get(leveldb::ReadOptions(), key, &value);
if (status.ok()) status = db->Delete(leveldb::WriteOptions(), key);
```

如上使用即可, 接口简单易用, 风格良好
在这部分中, 我们有遇到了两个新的Options, `leveldb::ReadOptions`, `leveldb::WriteOptions`
使用方法,同`leveldb::Options`, 进行设置, 使用即可, 默认设置使用临时对象即可.

*Q: 为什么DELETE, 也是使用`leveldb::WriteOptions`呢?*
*A: 实际上, PUT, DELETE操作仅仅只是置位的区别, 其他别无二致*

## leveldb::WriteBatch

我们在进行数据库的使用中, 有事务的概念, 事务具有原子性.
我们在进行某些操作的时候, **必须需要原子性, 那么, 原子性如何实现 ?**

`leveldb::WriteBatch`实现了原子性操作的控制

```cpp
#include <leveldb/db.h>
#include <leveldb/write_batch.h>
#include <leveldb/options.h>

open a database...

auto status = db->Put(leveldb::WriteOptions(), key, value);
if (status.ok()) {
    leveldb::WriteBatch batch;
    batch.Delete(key);
    batch.Put(leveldb::WriteOptions(), key_, value_);
    status = db->Write(leveldb::WriteOptions(), &batch);
}
```

我们可以理解为将操作缓存在`leveldb::WriteBatch`中, 之后一次性原子操作.

**WriteBatch除了有原子性, 还具有批量性操作加速的特点**


## leveldb::Iterator

因为数据库中存储的内容相当多, 所以我们必须有迭代措施. `leveldb::Iterator`便是为此而生

```cpp
// leveldb::Iterator

bool Valid() const;                   # 判断是否合法, 控制遍历条件
void SeekToFirst();                   # STL::Begin()
void SeekToLast()                     # STL::End()
void Seek(const Slice& target);       # 可以跳转到指定位置
void Next();                          # 后向移动, STL::Iterator++
void Prev();                          # 前向移动, STL::Iterator--
Slice key() const;                    # 返回节点Key
Slice value() const;                  # 返回节点Value
Status status() const;                # 返回节点Status
```

使用示例:

```cpp
auto iter = db->NewIterator(leveldb::ReadOptions());                 // (1)
for (iter->SeekToFirst(); iter->Valid(); iter->Next())
    std::cout << iter->key().ToString() << " " << iter->value().ToString() << std::endl;

// [start, limit)
auto iter = db->NewIterator(leveldb::ReadOptions());                 // (2)
for (iter->Seek(key) && iter->key() < limit; 
     iter->Valid(); 
     iter->Next())
    ...

auto iter = db->NewIterator(leveldb::ReadOptions());                 // (3)
for (iter->SeekToLast(); iter->Valid(); iter->Prev())
    ...
```

上面分别展示了 `(1)正向遍历`, `(2)指定位置遍历`, `(3)反向遍历`
**注意: 反向遍历要比正向遍历慢 !**

## leveldb::Snapshot

因为数据库实时在变动, 我们可能只需要其某一个时刻的状态即可, 快照机制是必须的.
当使用快照机制的时候, 所有操作限定在目标版本上操作

```cpp
leveldb::ReadOptions read_options;
read_options.snapshot = db->GetSnapshot();
...
...

db->ReleaseSnapshot(oread_options.snapshot);
```

其中我们要注意, 在长期不使用快照的时候, `leveldb::DB::ReleaseSnapshot()`释放快照
避免长期维护不使用的状态数据.

## leveldb::Slice

至此, 我们已经在很多接口中见过 `leveldb::Slice`了, 这个数据结构是leveldb中维护的特定串数据结构
某种意义上, 类似于SDS (Redis中的简单字符串)
其本质是, 一个`length`和**外部字符串**, 因此开销很低 (试试 std::move ? )

```cpp
class LEVELDB_EXPORT Slice {
 public:
  ........ // interface
 private:
  const char* data_;
  size_t size_;
};
```

对于`leveldb::Slice`支持和`std::string`的无缝转换 ,
(使用`leveldb::Slice::Constructor` + `leveldb::Slice::TOStirng()`)

使用`Slice`有一个很大的误区!

因为其使用的是外部字符串, 所以需要使用者, 自行保证生命期, 尤其是在`if-case`/`while`中

```cpp
{
    Slice slice;
    if (...) {
        slice = std::string("key");        // WARRRRRRRR !
	}
    db->Get(leveldb::ReadOptions(), slice, &val);
}
```

`Slice` 藉由 `std::string` 这样实现
```cpp
// Create a slice that refers to the contents of "s"
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) { }
```

所以在跳出作用域之后, `std::string`析构, `Slice`也会受到影响 !

## level::Comparator

众所周知, 作为key-value存储, 我们需要指定比较器, 比如:`std::map`
但是一旦我们的K-V不单纯, 有自行定义的类型时, 我们就需要自定义比较器, 用来满足KV的绝对弱序.

定义方法嘛, 和[platinum](https://github.com/Evil-crow/platinum/blob/master/src/protocol/parser.hpp)还是挺类似的

继承基类接口, 自定义实现即可 (但是platinum实现欠佳, 目前未统一接口, 不过思想到位了)

```cpp
#include <leveldb/comparator.h>

class OtherComparator : public leveldb::Comparator {

# 最核心的比较器接口
virtual int Compare(const Slice& a, const Slice& b) const = 0;

# 下面三个用来实现向后兼容性
virtual const char* Name() const = 0;
virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const = 0;
virtual void FindShortSuccessor(std::string* key) const = 0;
}
```

实现之后, `leveldb::Options`, 懂我意思了吧

```cpp
#include <leveldb/db.h>

leveldb::DB *db;
leveldb::Options options;
OthreComparator cmp;
options.comparator = &cmp;

leveldb::Open(options, "path/to/db", &db);
```

## leveldb::FilterPolicy

过滤器是KV中经常使用的设施, 比如`Google-BigTable`中使用了 `Bloom Filter`

使用过滤器有什么用呢? **可以加速查询, 减少硬盘查询的次数**

*在这一点上, Redis很快, 因为是基于内存的KV, 但是leveldb占用空间极小, 各有优劣*

那么, 如何做, 同`leveldb::Comparator`

```cpp
// 实现以下接口即可

class OthreFileterPolicy : public leveldb::FiuleterPolicy {
 public:
  virtual const char* Name() const = 0;

  virtual void CreateFilter(const Slice* keys, int n, std::string* dst)
      const = 0;

  virtual bool KeyMayMatch(const Slice& key, const Slice& filter) const = 0;
};

}
```

之后, 使用方法老套路

```cpp
leveldb::DB *db;
leveldb::Options options;
OthreFileterPolicy filter;
options.filter_policy = &filter;

leveldb::Open(options, "path/to/db", &db);
```

## leveldb::Env

同理, `Env`所管理的是与OS交互的文件操作. 我们可以认为设置行为模式, 同上理:

```cpp
class OtherEnv : public leveldb::Env {
    .. implementation of the Env interface ...
  };

OtherEnv env;
leveldb::Options options;
options.env = &env;
Status s = leveldb::DB::Open(options, ...);
```

## Performance

我们同时可以进行性能调整.有下面几个选项, 这几个选项都是可以在`leveldb::Options`中设置的

```cpp
leveldb::Options options;

options.block_size = xxx;                             # 设置block size
options.compression = leveldb::KNocompression;        # 设置压缩方式
options.cache = leveldb::NewLRUCache(100 * 1048576);  # 设置缓存
options.filter_policy = xxxxxx;                       # 设置过滤器
```

这是节个基础的调整性能设施的方法. 其他调优就可能需要从内核层面上入手了

## Concurrency

对于一款数据库而言, 并发性的处理十分关键, 其直接关系的数据库性能
对于leveldb而言.一个数据库在**同一时间内只能由一个进程打开，**
leveldb通过从**操作系统中获取锁的方式**防止误用。

在单个进程中，同一个`leveldb::DB`对象可以安全的由多个并发线程共享使用。

即，**不同的线程可以同时写入或获取迭代器**，

或在**没有任何外部同步的情况**在同一数据库上调用`leveldb::DB::Get`

但是其他对象（如`leveldb::Iterator`和`leveldb::WriteBatch`）可能需要外部同步。

如果两个线程共享这样的对象，它们**必须使用自己的协议锁**来保护自己的访问。(自行同步)

## Sync Writes

这里有一个关键的内容: **leveldb的写操作默认都是异步的**
在将操作推送到操作系统底层之后, write操作便会返回.

那么, 存在一个隐患: **Write返回之后, 宕机, 所有异步操作失效**

我们可以这样操作: 
```cpp
leveldb::WriteOptions options;
options.sync = true;

db->Put(options, ...);
```

即可实现同步写入, 但是**异步操作的性能十分高啊! ! !**

取下面这样两种折中方案:
1. 设有P次操作, 将其中N(0 < N <= M), M(N< M <= P)次操作使用同步手法写, 这样可以保证只有部分失效, 重启后,可以使用`CURRENT`文件进行恢复
2. 借用`leveldb::WriteBatch`, 在`db->Write()`中`options.sync = true`, 本次的同步开销将在每次操作中均摊

以上两种方法,均能获得还不错的性能

## 参考资料

[leveldb中文网doc](https://leveldb.org.cn/doc/)
[leveldb GitHub](https://github.com/google/leveldb)
[leveldb源码commits]()

---

leveldb代码十分精致 (真的像艺术品), 十分推荐阅读!
~~个人在找暑期实习, 时间可能会拖一点, 不过不会迟到!~~
另外[SSDB](http://ssdb.io/zh_cn/)也是挺有意思的项目, 我自己的项目也在做一个类似的, 不过可能就没有SSDB那么丰富,

那么, 下次leveldb源码阅读再见, 或者是 LevelNet架构解析再见了