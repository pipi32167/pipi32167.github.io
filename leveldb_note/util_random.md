### Random类

一个简单的伪随机数工具类。

获取下一个随机数：

```C++
uint32_t Next() {
  static const uint32_t M = 2147483647L;  // 2^31-1
  static const uint64_t A = 16807;        // bits 14, 8, 7, 5, 2, 1, 0
  // We are computing
  //       seed_ = (seed_ * A) % M,    where M = 2^31-1
  //
  // seed_ must not be zero or M, or else all subsequent computed values
  // will be zero or M respectively.  For all other values, seed_ will end
  // up cycling through every number in [1,M-1]
  uint64_t product = seed_ * A;

  // Compute (product % M) using the fact that ((x << 31) % M) == x.
  seed_ = static_cast<uint32_t>((product >> 31) + (product & M));
  // The first reduction may overflow by 1 bit, so we may need to
  // repeat.  mod == M is not possible; using > allows the faster
  // sign-bit-based test.
  if (seed_ > M) {
    seed_ -= M;
  }
  return seed_;
}

```

获取一个均匀分布的[0,n-1]的随机数：

```C++
uint32_t Uniform(int n) { return Next() % n; }

```

以1/n的概率返回true：

```C++
bool OneIn(int n) { return (Next() % n) == 0; }

```

返回**倾斜**的随机数，随机范围为[0, 2^max_log-1]，但是小数字的返回概率更大：

```C++
uint32_t Skewed(int max_log) { return Uniform(1 << Uniform(max_log + 1)); }

```

