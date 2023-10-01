---
title: Go map 源码分析
date: 2023-9-27 19:43:41
tags:
  - Go
  - Hash Table
  - Source Code
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
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
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

### 怎么知道数据放在哪个 bmap
这时候就要说到 **hash** 值了

``` Go
hash := alg.hash(key, uintptr(h.hash0))
```

Go 就是用 hmap.hash0（创建 map 时生成的随机数） 和存进 map 的 key 进行哈希运算，相等的 key 和 hash0 每次运算得出的 **hash** 值也相等


这个 **hash** 值可太有用了，map 就是用这个才实现出 O(1) 的时间复杂度

### hash 值咋用
再看下一行源码是 map 在决定数据放到哪个 bmap 时用的
``` Go
bucket := hash & bucketMask(h.B)
```

#### bucketMask(h.B) 是啥
``` Go
// bucketMask returns 1<<b - 1, optimized for code generation.
func bucketMask(b uint8) uintptr {
	return bucketShift(b) - 1
}
```
bucketMask 做的操作就是把 1 按二进制左移 b 位后再减去 1，比如当传进来的 b 是 4，1 用二进制表示是 0b1，左移 4 位后就是 0b10000，再减去 1 就是 0b1111，名符其实，就是掩码

#### h.B 是啥
h.B 就是 hmap 里的 B 字段，记录了 buckets 数组的长度信息。hmap.buckets 数组的长度是 2 的 hmap.B 次方

如果 B 是 4，那么 buckets 数组的长度是 2 的 4 次方就是 16

与掩码做 & 运算的结果只会落在 0 ~ 掩码这个闭区间里，在这个例子中也就是 0 ~ 15，这恰好是 buckets 数组下标的取值范围。这就是掩码的作用，相当于取模

就这样，从 key 可以映射到 buckets 下标位置，也就知道数据该存放到哪个 bmap 了，正是因为这个映射的存在，所以不需要遍历每一个 bucket 去比较，才把时间复杂度给打成了 O(1)

### bmap 里的 tophash 干啥用
tophash 是一个数组，一个 bmap 里可以放多个键值对，所以当有多个数据放进同一个 bmap 时就需要挨个比较了，看到底是哪个数据

下面是从 map 查数据的一段源码中截出来的
``` Go
for i := uintptr(0); i < bucketCnt; i++ {
    if b.tophash[i] != top {
        if b.tophash[i] == emptyRest {
            break bucketloop
        }
        continue
    }
    k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
    if t.indirectkey() {
        k = *((*unsafe.Pointer)(k))
    }
    if t.key.equal(key, k) {
        e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
        if t.indirectelem() {
            e = *((*unsafe.Pointer)(e))
        }
        return e
    }
}
```

tophash 数组元素里存的是 hash 高位的一部分，就是用于快速比较，因为小，所以快

当然比较完 tophash 还是得再比较一遍 key 来确保是要取的值，因为不同 key 的 hash 值有小概率的可能是相等的（何况更短的 tophash）

好了，看到这里已经知道 map 基本的数据存取是咋做的了。

但是如果数据越来越多，现有的 bmap 放不下了该怎么办？

这就得讲 map 的扩容机制了

### 扩容
先看下 bmap 中还藏着的一个值
``` Go
func (b *bmap) overflow(t *maptype) *bmap {
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-goarch.PtrSize))
}
```
这个 overflow 也是像 key value 那样通过指针计算拿到的，实际位置是放在 key value 的后边，是 bmap 的最后一部分

overflow 是一个 bmap 指针，它就是为数据溢出做准备的

当要新增一个键值对，而它的目标 bmap 已经满了的时候，就会把一个新的 bmap 通过 overflow 与旧 bmap “连”起来，将新键值对放进新 bmap 里

``` Go
// The current bucket and all the overflow buckets connected to it are full, allocate a new one.
newb := h.newoverflow(t, b)
inserti = &newb.tophash[0]
insertk = add(unsafe.Pointer(newb), dataOffset)
elem = add(insertk, bucketCnt*uintptr(t.keysize))
```

取数据时，遍历完当前 bmap 没找到的话要继续通过 overflow 去取溢出 bmap，就像遍历链表一样

``` Go
for ; b != nil; b = b.overflow(t) {
    for i := uintptr(0); i < bucketCnt; i++ {
        ...
    }
}
```

到这就有问题了，要是数据一直这么增长下去，overflow bmap 链表越来越长，map 的查询时间复杂度岂不是和遍历链表一样是 O(n)

所以当数据太多时，需要一个把 buckets 数组变长的机制，让 hash 发挥更大作用，将时间复杂度继续打下来，维持在 O(1)

以下这段是从 map 存数据源码中截取出，目的就是检测当前是否需要扩容

``` Go
// If we hit the max load factor or we have too many overflow buckets,
// and we're not already in the middle of growing, start growing.
if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
    hashGrow(t, h)
    goto again // Growing the table invalidates everything, so try again
}
```

#### 翻倍扩容

overLoadFactor 指示数据是否太多，需要翻倍扩容

``` Go
const (
	// Maximum average load of a bucket that triggers growth is 6.5.
	// Represent as loadFactorNum/loadFactorDen, to allow integer math.
	loadFactorNum = 13
	loadFactorDen = 2
)
// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
// bucketShift returns 1<<b, optimized for code generation.
func bucketShift(b uint8) uintptr {
	// Masking the shift amount allows overflow checks to be elided.
	return uintptr(1) << (b & (goarch.PtrSize*8 - 1))
}
```

hmap.count 是 map 中实际存储数据的数量，当 count 超过 buckets 数组长度的 6.5 倍后，就会触发翻倍扩容

还有一种情况会导致查询效率下降，而且数据量不大无法触发翻倍扩容

先是新增数据导致生成许多 overflow bmap 后又删除了很多数据，导致各个 bmap 中有很多空位置，数据密度下降了，查询时在一个 bmap 里没两个数据就得访问下一个 overflow bmap，这时候就得重新整理一下数据，提高数据密度

#### 等量扩容

tooManyOverflowBuckets 指示 overflow bmap 是否太多，需要整理（等量扩容）

``` Go
// tooManyOverflowBuckets reports whether noverflow buckets is too many for a map with 1<<B buckets.
// Note that most of these overflow buckets must be in sparse use;
// if use was dense, then we'd have already triggered regular map growth.
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}
```

hmap.noverflow 是大约有多少 overflow bmap

``` Go
// incrnoverflow increments h.noverflow.
// noverflow counts the number of overflow buckets.
// This is used to trigger same-size map growth.
// See also tooManyOverflowBuckets.
// To keep hmap small, noverflow is a uint16.
// When there are few buckets, noverflow is an exact count.
// When there are many buckets, noverflow is an approximate count.
func (h *hmap) incrnoverflow() {
	// We trigger same-size map growth if there are
	// as many overflow buckets as buckets.
	// We need to be able to count to 1<<h.B.
	if h.B < 16 {
		h.noverflow++
		return
	}
	// Increment with probability 1/(1<<(h.B-15)).
	// When we reach 1<<15 - 1, we will have approximately
	// as many overflow buckets as buckets.
	mask := uint32(1)<<(h.B-15) - 1
	// Example: if h.B == 18, then mask == 7,
	// and fastrand & 7 == 0 with probability 1/8.
	if fastrand()&mask == 0 {
		h.noverflow++
	}
}
```

B 小于 16 时，noverflow 是确切值，大于等于 16 时就是通过随机数按概率控制每次是否自增 1，为啥要搞这么麻烦呢？

为了降低 hmap 的内存占用量，noverflow 定为了 uint16 类型，为啥是 uint16 不是 uint32？要省这么点内存？因为 uint16 恰好能内存对齐

B 小于 16 时，当 noverflow 大于等于 buckets 数组（2 的 B 次方）长度就会触发等量扩容

B 大于等于 16 时，当 noverflow 大于等于 2 的 15 次方就会触发等量扩容

以上我们知道了 map 有两个扩容触发机制，那么触发后具体是怎么做的呢？

#### 渐进式扩容
触发后会调用 hashGrow() 开始扩容
``` Go
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}
```

这里做的是扩容的初始化步骤，根据扩容要求（等量扩容还是翻倍扩容）确定 B 是否加 1，将 hmap.oldbuckets 指针指向老的 buckets 数组，hmap.buckets 则指向崭新创建的新数组

实际的 bmap 拷贝操作是在后续的 growWork() 和 evacuate() 中做的

hmap.nevacuate 标记的是下一个要迁移的 oldbucket 下标位置，所以被初始化置为 0

因为 buckets 已经是崭新的数组了（没有实际数据），所以 hmap.noverflow 也被重置为 0

growWork() 会在 hmap 赋值和删除 key 时被调用，具体时机是在 hash 值计算完确定好是哪个目标 bucket 但未做实际操作前，调用前先判断当前 hmap 是否正在扩容

``` Go
...
    if h.growing() {
        growWork(t, h, bucket)
    }
...
```

``` Go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

evacuate() 调用一次只会迁移一个 oldbucket

growWork() 会先确保已经把当前要用的 oldbucket 迁移掉，然后再根据 nevacuate 多 evacuate 一个 oldbucket

``` Go
// evacDst is an evacuation destination.
type evacDst struct {
	b *bmap          // current destination bucket
	i int            // key/elem index into b
	k unsafe.Pointer // pointer to current key storage
	e unsafe.Pointer // pointer to current elem storage
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
					hash := t.hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```

evacuate() 比较长，其主要做的就是迁移一个 oldbucket 和它后面的所有 overflow bmap 到新的 buckets 目标位置，并清理掉旧的 overflow bmap

如果是翻倍扩容的话，oldbucket 的新位置就有两种可能（举个例子：比如原来 B 是 3，现在要迁移的 oldbucket 是 1，那么原先放在 oldbucket 的 key 的 hash 值末 3 位就是 0b001 ，现在 B 是 4 了，那 hash 就要多比较 1 位，多出来的这位要么是 1 要么是 0，也就是说现在的 buckets 下标要么是 0b0001 要么是 0b1001）


根据 nevacuate 迁移后还要更新 nevacuate 的值，指向下一个非空的 oldbucket，遇到空的 oldbucket 就跳过（为了保证当前操作的时间复杂度，还限制了最多只跳 1024 个 oldbucket）

还要判断迁移是否完成，迁移完了就做收尾工作

``` Go
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```

好了 Go map 源码暂时就分析到这了，还有 makemap 和遍历过程没分析，有兴趣的话自己看源码去吧
