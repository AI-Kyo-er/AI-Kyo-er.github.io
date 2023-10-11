---
title: The Solution of Data Lab of CSAPP
date: 2023-10-11 11:15:11
categories: 
- CS 
tags:
- 计算机系统基础

---
## 一、实验内容

本实验内容为 CMU 的 CSAPP 课程的第一个Lab内容，名称为 Data Lab。授课教师为WHU yili老师。

本实验中，你需要利用整型及浮点数的位表达形式，来解开一些人工谜题，一共有15个需要补充的函数，
除了在线评测之外。

实验的必要内容存在于本人的[github实验同名仓库](https://github.com/AI-Kyo-er/Data-Lab)。

## 二、报告要求

本报告要求学生把实验中实现的所有函数逐一进行分析说明，写出实现的依据，也就是推理过程，
可以是一个简单的数学证明，也可以是代码分析，根据实现中你的想法不同而异。

## 三、函数分析

### bitXor函数

**函数要求：**

函数名 | bitNor 
-|-------
参数 | int , int 
功能实现 | ~(x \| y) 
要求 | ~(x \| y) using only ~ and &  

**实现分析：**

由德摩根律：

~(x | y) = (~x) & (~y)

**函数实现：**

```C
int bitNor(int x, int y) {
  return  (~x) & (~y);
}
```

### copyLSB函数

**函数要求：**

函数名 | copyLSB
-|----------------------------
参数 | int
功能实现 | set all bits of result to least significant bit of x

**实现分析：**

通过左移位31，使得最不重要位成为最重要位，然后通过算术右移31位，使得最不重要位填满整个32 bits。

**函数实现：**

```C
int copyLSB(int x) {
  return (x << 31) >> 31;
}
```

### isEqual函数

**函数要求：**

函数名 | isEqual
-|----------------------------
参数 | int, int
功能实现 | return 1 if x == y, and 0 otherwise 

**实现分析：**

如果两个数相等，等价于bitwise相等，等价于互相异或等于零。

**函数实现：**

```C
int isEqual(int x, int y) {
  return !(x ^ y);
}
```

### bitMask函数

**函数要求：**

函数名 | bitMask
-|----------------------------
参数 | int, int
功能实现 | Generate a mask consisting of all 1's lowbit and highbit

**实现分析：**

即将全1的32-bits (~0) ，通过两端的零序列与之做与运算，屏蔽除了lowbit至highbit之间的 1。

**函数实现：**

```C
  int all_1 = ~0;

  return (all_1 << lowbit) & (all_1 + (1 << highbit << 1));
```

### bitMask函数

**函数要求：**

函数名 | bitCount
-|----------------------------
参数 | int
功能实现 | returns count of number of 1's in word

**实现分析：**

分治思想，将32 bits每2 bits / 4 bits / 8 bits / 16 bits划分，对于 $ 2^n $ ，将此 int 向右移位 n 位，
然后利用 00...00 (n 个) 11...11 (n 个) ... 的 mask 对其进行屏蔽后， 与x本身进行叠加，即得此 划分block内
的bit数， 最终通过 5 次移位叠加，得到最终的bit计数。

其中，mask的生成依靠对合适的 constant 进行移位。

**函数实现：**

```C
  int mask1 = 0x55, mask2 = 0x33, mask3 = 0x0F, mask4 = 0xFF, mask5 = 0xFF;
  mask1 |= mask1 << 8;
  mask1 |= mask1 << 16;

  mask2 |= mask2 << 8;
  mask2 |= mask2 << 16;

  mask3 |= mask3 << 8;
  mask3 |= mask3 << 16;

  mask4 |= mask4 << 16;

  mask5 |= mask5 << 8;

  x = (x & mask1) + ((x >> 1) & mask1);
  x = (x & mask2) + ((x >> 2) & mask2);
  x = (x & mask3) + ((x >> 4) & mask3);
  x = (x & mask4) + ((x >> 8) & mask4);
  x = (x & mask5) + ((x >> 16) & mask5);

  return x;
```

### bitMask函数

**函数要求：**

函数名 | tmax
-|----------------------------
参数 | void
功能实现 | return maximum two's complement integer 

**实现分析：**

考虑到最大数为1 后面跟 31个零，故对(1 << 31) 进行取反。

**函数实现：**

```C
  int tmax(void) {
    return ~(1 << 31);
  }
```

### isNonNegative函数

**函数要求：**

函数名 | isNonNegative
-|----------------------------
参数 | int
功能实现 | return 1 if x >= 0, return 0 otherwise 

**实现分析：**

通过右移31位，实现全为符号位，然后取否 或者 + 1 或者 & 1 都可。

**函数实现：**

```C
  int isNonNegative(int x) {
    return !(x >> 31);
  }
```

### addOK函数

**函数要求：**

函数名 | addOK
-|----------------------------
参数 | int, int
功能实现 | Determine if can compute x+y without overflow

**实现分析：**

针对符号位操作，如果两个加数的符号相反，则不会溢出，否则再判断和 和 某一加数的符号，如果相同则没有溢出。

**函数实现：**

```C
  int addOK(int x, int y) {
    int SameSign1 = (x ^ y);
    int SameSign2 = ((x + y) ^ y);

    return ((SameSign1 | ~SameSign2) >> 31)& 1;
  }
```

### rempwr2函数

**函数要求：**

函数名 | rempwr2
-|----------------------------
参数 | int, int
功能实现 | Compute x%(2^n), for 0 <= n <= 30

**实现分析：**

每有一个n，则通过mask屏蔽掉其他的位，最终得到余数结果。

其中，做特殊处理的是取余之后等于零的负数，由于补码不存在 -0，所以自行检测该情况进行置零。

**函数实现：**

```C
  int rempwr2(int x, int n) {
    int all_1 = (~0);
    int mask = all_1 << n;
    int ans = (((x ^ (x >> 31)) & mask) ^ x);
    return ans & (all_1 + (!(ans + (1 << n))));
  }
```
### isLess函数

**函数要求：**

函数名 | isLess
-|----------------------------
参数 | int, int
功能实现 | if x < y  then return 1, else return 0 

**实现分析：**

针对x 、y 、 x - y 的符号位操作，x正y负必定返回 0，x负y正必定返回 1，其他情况判断 x - y的符号。

**函数实现：**

```C
  int isLess(int x, int y) {
    int y_ = ~y;
    int ans = ((y_ + 1 + x) | (x & y_)) & (x | y_);
    return (ans >> 31) & 1;
  }
```

### absVal函数

**函数要求：**

函数名 | absVal
-|----------------------------
参数 | int
功能实现 | absolute value of x

**实现分析：**

整数保持不变，负数取反加一。

**函数实现：**

```C
  int absVal(int x) {
    int mask = x >> 31;
    return (x ^ mask) + (mask & 1);
  }
```

### isPower2函数

**函数要求：**

函数名 | isPower2
-|----------------------------
参数 | int
功能实现 | returns 1 if x is a power of 2, and 0 otherwise

**实现分析：**

注意到，只有一个位为1的int有专有性质： 其加上全1得到的是前面全零，后面全1的数，或 原数恰好为0。

**函数实现：**

```C
  int isPower2(int x) {
    int mask = ~(x >> 31);
    int ifPower2 = (x & (x + mask));
    return !(ifPower2 | !x);
  }
```

### float_neg函数

**函数要求：**

函数名 | float_neg
-|----------------------------
参数 | unsigned
功能实现 | Return bit-level equivalent of expression -f for floating point argument f.

**实现分析：**

直接返回转置符号位结果，特别的，如果exp全1，frac非全0，则返回本身。

**函数实现：**

```C
  unsigned float_neg(unsigned uf) {
    if((uf & 0x7fffffff) > 0x7f800000) return uf;
  
    return uf ^ 0x80000000;
  }
```

### float_half函数

**函数要求：**

函数名 | float_half
-|----------------------------
参数 | unsigned
功能实现 | Return bit-level equivalent of expression -f for floating point argument f.

**实现分析：**

对于一般情况，直接把exp减一。

特别的exp = 1，将frac右移，并在二分位填充1。exp = 0，将frac右移，并在二分位填充0。
（同时需要处理舍入问题）


**函数实现：**

```C
  unsigned float_half(unsigned uf) {
    int expMask = 0x7F800000, fracMask = 0x007FFFFF, signMask = 0x80000000;
    int exp = uf & expMask, frac = uf & fracMask, sign = uf & signMask;

    if (!(exp ^ expMask)) return uf;
  
    if (exp <= 0x00800000) {
      if ((frac & 0x3) == 0x3) frac += 1;
      frac >>= 1;
      frac |= exp >> 1;
      exp = 0;
    }
    else exp -= 0x00800000;

    return sign | exp | frac; 
  }
```

### float_i2f函数

**函数要求：**

函数名 | float_i2f
-|----------------------------
参数 | int
功能实现 | Return bit-level equivalent of expression (float) x

**实现分析：**

* sign： 直接拷贝int的符号位。
* exp：基准位 0x4B000000，每次左移（右移）int，exp -=（+=） 0x00800000。
* frac： int通过移位使其对其float的frac位，同时注意舍入。

**函数实现：**

```C
  unsigned float_i2f(int x) {
    int x_o = x;
    int sign = 0, exp = 0x4B000000;
    int count = -1;
    int mask = 0;
    int a = 0, b = 0;

    if (!x) return 0;
    if (x == 0x80000000) return 0xCF000000;

    if (x >> 31) {
      sign = 0x80000000;
      x = (~x) + 1;
      x_o = x;
    }

    while (x < 0x00800000) {
      x <<= 1;
      exp -= 0x00800000;
    }

    if (x >= 0x01000000) {
      while (x >= 0x01000000) {
        x >>= 1;
        count++;
        exp += 0x00800000;
      }

      mask = 0xFFFFFFFFU >> (31 - count);
      a = 1 << count;
      b = x_o & mask;
      if (b > a || (b == a && (x & 1))) x += 1;
    }
    if (x > 0x00FFFFFF) exp += 0x00800000;

    return sign | exp | (x & 0x007FFFFF);
  }  
```

## 四、实验总结

总体实验比较考察技巧性，在全然不顾及可读性的前提下，尽可能的化约计算操作数。

### 化约方法：
* 可以简化的等价逻辑运算：当大量 & | ~ ！ 出现时，往往可以通过布尔代数的公式进行等价简化。
* 可以改进的思路：问题考察的形参有多种方式检测是否符合条件，得到充要性条件以方便检测。譬如：对于float_neg中
NaN 的检测可以将检查「exp全一 and frac非全零」等价于检测除符号位外是否大于 0x7F800000。

### 时间花费
七个小时完成，两个小时优化操作数。