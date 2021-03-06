### 0.builder.go

注意，如果使用了strings.Builder ，很多操作之前都会判断是否可以copy，被赋值之后就无法进行copy。

```
func (b *Builder) copyCheck() {
	if b.addr == nil {
		// This hack works around a failing of Go's escape analysis
		// that was causing b to escape and be heap allocated.
		// See issue 23382.
		// TODO: once issue 7921 is fixed, this should be reverted to
		// just "b.addr = b".
		b.addr = (*Builder)(noescape(unsafe.Pointer(b)))
	} else if b.addr != b {
		panic("strings: illegal use of non-zero Builder copied by value")
	}
}
```

### 	1. replace.go

字符串替换

主要由提供字符串替换功能

```
// Replace returns a copy of the string s with the first n
// non-overlapping instances of old replaced by new.
// If old is empty, it matches at the beginning of the string
// and after each UTF-8 sequence, yielding up to k+1 replacements
// for a k-rune string.
// If n < 0, there is no limit on the number of replacements.
func Replace(s, old, new string, n int) string {}

```

```
// ReplaceAll returns a copy of the string s with all
// non-overlapping instances of old replaced by new.
// If old is empty, it matches at the beginning of the string
// and after each UTF-8 sequence, yielding up to k+1 replacements
// for a k-rune string.
func ReplaceAll(s, old, new string) string {
   return Replace(s, old, new, -1)
}
```

通过内置的trie-tree快速查找替换。



### 2. strings.go

使用了Buffer升级。存储空间改动维护，业务类代码

#### Index(s, substr string) int

会根据substring的长度选择不一样的算法

其中

```bytealg.MaxLen```和cpu有有关系，定义在/bytealg/cpu/

如果失败次数过多，就会使用RabinKarp算法：

`BruteForce`算法: 暴力匹配， 匹配substring 和 s[i: i+len(substr)]. 当 substr长度比较小的时候。

[`RabinKarp`算法](https://zh.wikipedia.org/wiki/%E6%8B%89%E5%AE%BE-%E5%8D%A1%E6%99%AE%E7%AE%97%E6%B3%95): 

```
n = t.length
m = p.length
h = dm-1 mod q
p = 0
t0 = 0
for i = 1 to m
    p = (dp + p[i]) mod q
    t0 = (dt0 + t[i]) mod q
for s = 0 to n - m
    if p = ts
        if p[1.....m] = t[s + 1..... s + m]
            print "pattern found at position" s
    If s < n-m
        ts + 1 = (d (ts - t[s + 1]h) + t[s + m + 1]) mod q
```

在上面算法中，滑动窗口+hash，其中hash(s[i,i+len(substr)])  可以直接获取 hash(s[i+1, i+len(substr)+1])的值。

由于是hash值，准确率和q的选取有关系，go使用的是

```
// PrimeRK is the prime base used in Rabin-Karp algorithm.
const PrimeRK = 16777619
```

### 3. builder.go

```go
// String returns the accumulated string.
func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf))
}
```

对比bytes

```
// String returns the contents of the unread portion of the buffer
// as a string. If the Buffer is a nil pointer, it returns "<nil>".
//
// To build strings more efficiently, see the strings.Builder type.
func (b *Buffer) String() string {
   if b == nil {
      // Special case, useful in debugging.
      return "<nil>"
   }
   return string(b.buf[b.off:])
}
```

#### [unsafe.Pointer](https://pkg.go.dev/unsafe#Pointer)

文档内说明了pointer能做什么

```
- A pointer value of any type can be converted to a Pointer.
- A Pointer can be converted to a pointer value of any type.
- A uintptr can be converted to a Pointer.
- A Pointer can be converted to a uintptr.
```

本质上是uint指针, unsafer.Point 可以在需要的时候任意转换。可以将任意指针转换为pointer再转换为其他指针。（内存要保证变换后的不大于变换前）

(1) 将 *T1 转换为指向 *T2 的指针

T2的内存要小于等于T1的内存

```
func Float64bits(f float64) uint64 {
	return *(*uint64)(unsafe.Pointer(&f))
}
```

```
var s string
hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // case 1
hdr.Data = uintptr(unsafe.Pointer(p))              // case 6 (this case)
hdr.Len = n
```

#### 参考

> https://stackoverflow.com/questions/41326506/in-rabinkarp-algorithm-why-compare-hash-first
>
> https://www.programiz.com/dsa/rabin-karp-algorithm
>
> https://brilliant.org/wiki/rabin-karp-algorithm/