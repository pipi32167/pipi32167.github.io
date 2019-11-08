## Arena类

一个内存管理类，用于内存批量申请，避免多次调用malloc/realloc/new/delete。

当需要堆内存的时候，每次分配一块4096字节的内存，然后慢慢从这块内存上取，直到剩余空间不足时，再分配出新的块出来。

使用方式如下：

```C++
{
	leveldb::Arena arena;
  //分配内存
  char* buff = arena.Allocate(sizeof(Foo)); 
  //在内存上调用构造函数
  auto *pFoo = new(buff) Foo; 
  // do something
  //显式调用析构函数做清理工作
  pFoo->~Foo();
} //这里Arena类的析构函数会将之前分配的内存都释放掉
```

类的内部有几个成员变量：

```C++
// 当前内存块指针
char* alloc_ptr_;
// 当前内存块的剩余空间
size_t alloc_bytes_remaining_;
// 管理创建出来的内存块列表，最后在Arena的析构函数里一次性全部释放掉
std::vector<char*> blocks_;
// 内存使用量，奇怪的是这里使用了atomic来保证读写是原子操作，但是其他成员的访问却没有相关保护
std::atomic<size_t> memory_usage_;
```

构造函数很简单，就是将内部成员设置为空指针和0：

```C++
Arena::Arena()
    : alloc_ptr_(nullptr), alloc_bytes_remaining_(0), memory_usage_(0) {}

```

析构函数将分配的内存块全部释放掉：

```C++
 Arena::~Arena() {
  for (size_t i = 0; i < blocks_.size(); i++) {
    delete[] blocks_[i];
  }
}
```

分配内存函数，注意到这里采用了内联函数标记进行优化：

```C++
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);
  // 内存块剩余空间足够。
  if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  // 内存块剩余空间不足，需要分配一块新的内存。
  return AllocateFallback(bytes);
}
```

分配内存函数的对齐版本，返回的内存块大小是2的幂，至少是8的倍数：

```C++
char* Arena::AllocateAligned(size_t bytes) {
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  static_assert((align & (align - 1)) == 0,
                "Pointer size should be a power of 2");
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  size_t needed = bytes + slop;
  char* result;
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
  return result;
}
```

当剩余空间不足时，分配一块新的内存：

```C++
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    // kBlockSize是块的大小，默认设置为4096。
    // 当对象大于块大小的1/4时，为了避免浪费太多空间，单独分配指定大小的新块。
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // 将内存块指针指向新创建的块，但是当前块的剩余空间都被浪费掉了。
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

单独分配一个新块：

```C++
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  // 更新内存用量
  memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
  return result;
}
```