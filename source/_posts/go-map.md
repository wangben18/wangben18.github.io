---
title: Go map
date: 2023-9-27 19:43:41
tags:
  - Go
  - map
categories:
  - Go
  - map
---
## 哈希表
哈希表的查询效率贼高，时间复杂度是 O(1)，这是咋实现的呢？

Go 的哈希表结构就是 map，这次我们就来探索一下它底层是咋回事～

## 3，2，1，上源码！
``` Go
const (
	// Maximum number of key/value pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits
)
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

这个就是 Go map 底层结构的部分源码了，业务代码中的每一个 map 底层对应的就是一个 hmap

咋？一下子给的太多受不了？没事，咱一个个来看！

### 先看 hmap
hmap 有个字段 buckets，这个字段的结构是一个指针，指向了一个数组，这个数组里的元素是啥呢？

### bmap
其实就是上面源码中的 bmap 结构，这个 bmap 里放的就是你存进 map 里的 key value 了，每个 bmap 里都可以保存 bucketCnt 个键值对

啥？你说这个 bmap 里不就一个 tophash 字段，哪有什么 key value？

#### bmap 里 key value 在哪
3，2，1 上源码！
``` Go
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))

v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
```
key value 就是这么通过指针计算获取的，key value 数据就存在 tophash 的后边

### 怎么知道数据放在哪个 bmap，难道是按顺序放吗
不是哦！按顺序一个个查过去和直接用数组有啥区别。。。

这时候就要说到 **hash** 值了

``` Go
hash := alg.hash(key, uintptr(h.hash0))
```

Go 就是用 hmap.hash0 和你存进 map 的 key 进行哈希运算，相等的 key 每次运算得出的 **hash** 值也相等


这个 **hash** 值可太有用了，map 就是用这个才实现出 O(1) 的时间复杂度

### hash 值咋用
再看下一段源码，这段源码是 map 在决定数据放到哪个 bmap 时用的
``` Go
bucket := hash & bucketMask(h.B)
```
好家伙，用 hash 值和一个玩意做了个与运算

#### bucketMask(h.B) 是啥
先看下 bucketMask 是啥
``` Go
// bucketMask returns 1<<b - 1, optimized for code generation.
func bucketMask(b uint8) uintptr {
	return bucketShift(b) - 1
}
```
ok，bucketMask 做的操作就是把 1 按二进制左移 b 位后再减去 1，比如当传进来的 b 是 4，1 用二进制表示是 0b1，左移 4 位后就是 0b10000，再减去 1 就是 0b1111

#### h.B 是啥
h.B 就是 hmap 里的 B 字段，记录了 buckets 数组的长度信息

hmap.buckets 数组的长度是 2 的 hmap.B 次方，如果 B 是 4，那么 buckets 数组的长度是 2 的 4 次方就是 16

#### 做 & 运算有啥意义
现在我们已经知道 **hash** 是哈希运算得出的值，类型为 uintptr，他在 64 位机器上是 64 个 0 和 1 组成的

**bucketMask(h.B)** 是掩码，返回值类型也是 uintptr



好了，看到这里已经知道 map 基本的数据存取是咋做的了。

但是如果数据越来越多，现有的 bmap 放不下了该怎么办？

这就得讲 map 的扩容了

### 扩容
到这写得有点累了，下回再更新，抱一丝～