### map
> map定义：由一组key，value组成的抽象的数据结构，并且key唯一。

+ map的操作（增删改查）
+ 增加add/insert一个key，value
+ 删除remove/delete一个key，value
+ 修改Reassign某个key对应的value
+ 查询lookup某个key对应的value

#### 数据结构
> 哈希查找表hash table, 搜索树search tree

+ 哈希查找表用一个hash函数将key分配到不同的桶中，这样主要开销在哈希计算，数组常数的获取上。性能很高。
+ 哈希表碰撞问题，分配使用的方法：链表法，开放地址法。

#### 简单描述
1. 结构体为hmap，用buckets存放一些bmap桶（2的指数倍）
2. bmap是一种8个格子的桶（一定只有8个个字），每个格子存放一对key-value
3. bmap中overflow用于连接下一bmap（溢出桶）
4. hmap存有oldbuckets，用于存放老数据（用于扩容时）
5. mapextra用于存放 非指针数据（用于优化存储和访问）,内部含有overflow，oldoverflow，本质还是bmap数组

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
    count     int 
    // 元素个数，调用 len(map) 时，直接返回此值
    // # live cells == size of map.  Must be first (used by len() builtin)
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
	// If both key and value do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and value do not contain pointers.
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
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit more complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

#### map 查询
1. 计算key的hash(如果cpu支持aes，使用aes hash，否则memhash)
2. 根据最后的B确定来源于那个桶bucket
3. 根据key的前8位确定在那个格子，（bmap存放每个key对应的tophash，就是key的前8位）
4. 比较key的完整hash，匹配成功获取对应的value
5. 如未找到，到overflow继续寻找
6. 如一直未寻找到key，返回key对应的类型的零值，不是nil

### 2种方式
```go

value1 := caseMap["one"]

value2, ok := caseMap["one"]

```

#### map 增加/插入
1. 计算key的hash
2. 根据B确定哪个桶
3. 根据前8位获取格子是否存在，如没有未满，存储
4. 如果满了，创建一个bmap作为溢出桶连接在overflow

#### map 扩容
*触发扩容条件*
1. 装载因子超过阈值 > 6.5
> loadFactor := count / (2^B) count:map元素个数，2^B:桶的数量
2. overflow的bucket数量过多(溢出桶太多，链表很长)
3. 当前map不能正在扩容

#### 扩容过程
1. 渐进式扩容，不能一次性全部key迁移完成，每次最多迁移2个bucket
2. 实质是分配新的bucket，将旧bucket放到oldbuckets上。增删改触发。全部迁移后，是否oldbucket。
3. 扩容影响查询和新增，查询从oldbuckets，新增，更新要从新的桶进行
4. 新bucket是扩容器2倍，分为X part，Y part。bucket的key会落到这2个桶（X,Y）,判断最后B提前一位，确定落入那个桶part。

#### map 遍历
> 遍历所有bucket以及后面overflow bucket

#### map 删除
1. 根据key的hash找到对应bucket
2. 判断是否扩容中，直接触发一次搬迁。
3. 找到对应位置，清楚key，value值


