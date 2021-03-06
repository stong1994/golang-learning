# map

*go version go1.12.6 windows/amd64*

#### 一、汇编命令查看调用函数
先创建一个文件`declar_map.go`，然后再创建一个`map`，代码如下：
```go
package main

var simpleMap = make(map[string]int)
```
查看汇编`go tool compile -S declar_map.go`  
发现生成了一堆花里胡哨的命令（我代码才两行啊）。截取前一小部分：
```
"".init.ializers STEXT size=92 args=0x0 locals=0x10
        0x0000 00000 (declar_map.go:3)  TEXT    "".init.ializers(SB), ABIInternal, $16-0
        0x0000 00000 (declar_map.go:3)  MOVQ    TLS, CX
        0x0009 00009 (declar_map.go:3)  MOVQ    (CX)(TLS*2), CX
        0x0010 00016 (declar_map.go:3)  CMPQ    SP, 16(CX)
        0x0014 00020 (declar_map.go:3)  JLS     85
        0x0016 00022 (declar_map.go:3)  SUBQ    $16, SP
        0x001a 00026 (declar_map.go:3)  MOVQ    BP, 8(SP)
        0x001f 00031 (declar_map.go:3)  LEAQ    8(SP), BP
        0x0024 00036 (declar_map.go:3)  FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0024 00036 (declar_map.go:3)  FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0024 00036 (declar_map.go:3)  FUNCDATA        $3, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
        0x0024 00036 (declar_map.go:3)  PCDATA  $2, $0
        0x0024 00036 (declar_map.go:3)  PCDATA  $0, $0
        0x0024 00036 (declar_map.go:3)  CALL    runtime.makemap_small(SB)
        0x0029 00041 (declar_map.go:3)  PCDATA  $2, $1
        0x0029 00041 (declar_map.go:3)  MOVQ    (SP), AX
        0x002d 00045 (declar_map.go:3)  PCDATA  $2, $-2
        0x002d 00045 (declar_map.go:3)  PCDATA  $0, $-2
        0x002d 00045 (declar_map.go:3)  CMPL    runtime.writeBarrier(SB), $0
        0x0034 00052 (declar_map.go:3)  JNE     71
        0x0036 00054 (declar_map.go:3)  MOVQ    AX, "".simpleMap(SB)
        0x003d 00061 (declar_map.go:3)  MOVQ    8(SP), BP
        0x0042 00066 (declar_map.go:3)  ADDQ    $16, SP
        0x0046 00070 (declar_map.go:3)  RET
        0x0047 00071 (declar_map.go:3)  LEAQ    "".simpleMap(SB), DI
        0x004e 00078 (declar_map.go:3)  CALL    runtime.gcWriteBarrier(SB)
        0x0053 00083 (declar_map.go:3)  JMP     61
        0x0055 00085 (declar_map.go:3)  NOP
        0x0055 00085 (declar_map.go:3)  PCDATA  $0, $-1
        0x0055 00085 (declar_map.go:3)  PCDATA  $2, $-1
        0x0055 00085 (declar_map.go:3)  CALL    runtime.morestack_noctxt(SB)
        0x005a 00090 (declar_map.go:3)  JMP     0

```
我们战略性的忽略其他汇编代码，看到`runtime.makemap_small(SB)`，也就是调用了`runtime`包下的`makemap_small`函数。  
**遗留问题1**：为什么是`small`?

#### 二、GO源码 — runtime.makemap_small
源码位置：`runtime/map.go:294`
```go
func makemap_small() *hmap {
	h := new(hmap)
	h.hash0 = fastrand()
	return h
}
```
`new`了一个`hmap`，然后对`hmap`的`hash0`属性初始化为随机数(`hmp`是`hashmap`的缩写)。  
查看`hmap`的结构
```go
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
```
一共有九个属性（`slice`才三个。。。），先重点记住这几个
- hash0: hash种子
- count： map的大小
- flags: map的状态
- B：bucket的数量为2^8
- buckets: bucket组成的数组
- oldbuckets: map扩容时，会把buckets前一半的数据放在这里，扩容完后，置为nil
- noverflow：overflow是溢出的bucket，noverflow是overflow bucket的大致数量，（没太懂为啥是大致）

#### 查找元素
增加代码
```go
func add() {
	simpleMap["a"] = 1
}
```
编译后发现`CALL    runtime.mapassign_faststr(SB)`，查看源码`runtime/map_faststr.go:12`
```go
func mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapaccess1_faststr))
	}
	if h == nil || h.count == 0 {
		return unsafe.Pointer(&zeroVal[0])
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	key := stringStructOf(&ky)
	if h.B == 0 {
		// One-bucket table.
		b := (*bmap)(h.buckets)
		if key.len < 32 {
			// short key, doing lots of comparisons is ok
			for i, kptr := uintptr(0), b.keys(); i < bucketCnt; i, kptr = i+1, add(kptr, 2*sys.PtrSize) {
				k := (*stringStruct)(kptr)
				if k.len != key.len || isEmpty(b.tophash[i]) {
					if b.tophash[i] == emptyRest {
						break
					}
					continue
				}
				if k.str == key.str || memequal(k.str, key.str, uintptr(key.len)) {
					return add(unsafe.Pointer(b), dataOffset+bucketCnt*2*sys.PtrSize+i*uintptr(t.valuesize))
				}
			}
			return unsafe.Pointer(&zeroVal[0])
		}
		// long key, try not to do more comparisons than necessary
		keymaybe := uintptr(bucketCnt)
		for i, kptr := uintptr(0), b.keys(); i < bucketCnt; i, kptr = i+1, add(kptr, 2*sys.PtrSize) {
			k := (*stringStruct)(kptr)
			if k.len != key.len || isEmpty(b.tophash[i]) {
				if b.tophash[i] == emptyRest {
					break
				}
				continue
			}
			if k.str == key.str {
				return add(unsafe.Pointer(b), dataOffset+bucketCnt*2*sys.PtrSize+i*uintptr(t.valuesize))
			}
			// check first 4 bytes
			if *((*[4]byte)(key.str)) != *((*[4]byte)(k.str)) {
				continue
			}
			// check last 4 bytes
			if *((*[4]byte)(add(key.str, uintptr(key.len)-4))) != *((*[4]byte)(add(k.str, uintptr(key.len)-4))) {
				continue
			}
			if keymaybe != bucketCnt { // 第一次for循环相等，然后keymaybe被赋值为i，所以for循环第二次进入goto，剩下的就不用遍历了吗
				// Two keys are potential matches. Use hash to distinguish them.
				goto dohash
			}
			keymaybe = i
		}
		if keymaybe != bucketCnt {
			k := (*stringStruct)(add(unsafe.Pointer(b), dataOffset+keymaybe*2*sys.PtrSize))
			if memequal(k.str, key.str, uintptr(key.len)) {
				return add(unsafe.Pointer(b), dataOffset+bucketCnt*2*sys.PtrSize+keymaybe*uintptr(t.valuesize))
			}
		}
		return unsafe.Pointer(&zeroVal[0])
	}
dohash:
	hash := t.key.alg.hash(noescape(unsafe.Pointer(&ky)), uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
	for ; b != nil; b = b.overflow(t) {
		for i, kptr := uintptr(0), b.keys(); i < bucketCnt; i, kptr = i+1, add(kptr, 2*sys.PtrSize) {
			k := (*stringStruct)(kptr)
			if k.len != key.len || b.tophash[i] != top {
				continue
			}
			if k.str == key.str || memequal(k.str, key.str, uintptr(key.len)) {
				return add(unsafe.Pointer(b), dataOffset+bucketCnt*2*sys.PtrSize+i*uintptr(t.valuesize))
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```
逻辑流程：  
- 如果`map`为`nil`或者大小为0，那么返回零值`zeroVal[0]`
- 如果当前map正在进行写操作，直接`panic`
- 如果`h.B==0`，即bucket的数量为1（2^0=1）
    - 如果字符串(key)的长度小于32，遍历桶中的cell，如果key相等，返回对应值的指针地址，否则返回零值
    - 如果字符串长度不小于32，遍历桶中的cell
        - 如果key的指针地址相等，直接返回对应值的指针地址
        - 如果key的指针地址不相等，判断key前四位和后四位的value值是否相等
            - 如果不等，继续遍历
            - 如果相等，说明很有可能key相等，通过hash计算key值是否相等。
                - m为map低B位全1，如B等于3，m为111
                - b为bucket的位置
                - 如果oldbucket不为空，说明正在扩容，需要调整m，m右移1位，同时调整b
                - 遍历b的溢出桶
                    - 遍历桶中的cell，如果相等，返回对应值得指针地址
                - 如果没有相等的key，返回零值
- m为map低B位全1，如B等于3，m为111
- b为bucket的位置
- 如果oldbucket不为空，说明正在扩容，需要调整m，m右移1位，同时调整b
- 遍历b的溢出桶
    - 遍历桶中的cell，如果相等，返回对应值得指针地址
- 如果没有相等的key，返回零值

其中`bmap`为bucket的结构
```go
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
注释说如果`tophash[0]`小于`minTopHash`，那么`tophash[0]`是bucket的扩容状态。  
因为不清楚`map`的类型，因此`bmp`还隐藏了其他字段，如key/value。为了节省内存空间，key放在了一起，value放在了一起。  

#### 添加元素
增加代码
```go
func add() {
	simpleMap["a"] = 1
}
```
编译后发现`CALL    runtime.mapassign_faststr(SB)`，查看源码`runtime/mapassign_faststr.go:202`
```go
func mapassign_faststr(t *maptype, h *hmap, s string) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(unsafe.Pointer(h), callerpc, funcPC(mapassign_faststr))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	key := stringStructOf(&s)
	hash := t.key.alg.hash(noescape(unsafe.Pointer(&s)), uintptr(h.hash0))

	// Set hashWriting after calling alg.hash for consistency with mapassign.
	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
	bucket := hash & bucketMask(h.B) // 根据key的哈希和已有的桶的个数来计算key应存入的桶的哈希
	if h.growing() { // 如果map正在扩容
		growWork_faststr(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize))) // 获取key应存入的桶
	top := tophash(hash) // 获取key的哈希的高8位

	var insertb *bmap // 要插入的桶
	var inserti uintptr // 要插入的桶中cell的index
	var insertk unsafe.Pointer // 

bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ { // 遍历cell
			if b.tophash[i] != top { // 如果第i个cell的的tophash（tophash对应一种状态）不等于key的哈希（高八位）
				if isEmpty(b.tophash[i]) && insertb == nil { // 如果第i个cell为空，并且insertb还没有被赋值，那么这里可以用来加载该key
					insertb = b // 记录要插入的桶和桶中cell的index
					inserti = i
				}
				if b.tophash[i] == emptyRest { // 如果剩下的桶都为空，那么结束循环
					break bucketloop
				}
				continue
			}
			k := (*stringStruct)(add(unsafe.Pointer(b), dataOffset+i*2*sys.PtrSize)) // 获取该索引的cell对应的string的len
			if k.len != key.len { // 如果与key的长度不一致，说明key不相等
				continue
			}
			if k.str != key.str && !memequal(k.str, key.str, uintptr(key.len)) { // 判断第i个cell的字符串内容是否一样
				continue
			}
			// 经过上边，得到结论：map中已经有该key了，那么更新该key的值
			// already have a mapping for key. Update it.
			inserti = i
			insertb = b
			goto done
		}
		// 能到这里，说明上边没有找到和key一样的cell，那么就去该桶的overflow找
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
    // 能到这里说明在桶及桶的所有溢出桶中都没有对应的key，那就分配一个cell
	// Did not find mapping for key. Allocate new cell & add entry.

    // 判断是否应该扩容，先决条件是该map没有正在扩容，然后map的数量超过了过载因子或者map的溢出桶太多
	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		// 扩容完后，h.B可能加1，所以根据key的哈希来定位桶的操作无效。所以从again开始从新定位桶
		goto again // Growing the table invalidates everything, so try again
	}
    // 判断是否找到了合适的地方添加该key，即桶中存在一个cell为空或者map的一个key与该key相等
    // 如果没有找到一个地方来添加该key，那么就新建一个溢出桶。
	if insertb == nil {
		// all current buckets are full, allocate a new one.
		insertb = h.newoverflow(t, b)
		inserti = 0 // not necessary, but avoids needlessly spilling inserti
	}
	// 更新插入cell的tophash（只用于新建的溢出桶，如果是在已存在的桶中插入该key，那么top=insertb.tophash[inserti&(bucketCnt-1)]）
	insertb.tophash[inserti&(bucketCnt-1)] = top // mask inserti to avoid bounds checks
    // 找到插入的位置的指针地址
	insertk = add(unsafe.Pointer(insertb), dataOffset+inserti*2*sys.PtrSize)
	// 对该地址进行赋值
	// store new key at insert position
	*((*stringStruct)(insertk)) = *key
	h.count++

done:
	// 找到该key的对应value的指针地址
	val := add(unsafe.Pointer(insertb), dataOffset+bucketCnt*2*sys.PtrSize+inserti*uintptr(t.valuesize))
	// 再次检查该map是否为写操作，如果不是，那么说明有其他操作更新了map，抛出异常
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	// 将该map状态的写状态标记为0，即停止写的状态，并返回value的指针地址
	h.flags &^= hashWriting
	return val
}

```
流程：
- 如果该map为空的话，直接panic
- 先判断该map是否在写，如果是的话，抛出异常。
- 计算该key的哈希值
- 标记该map为写状态
- 如果map刚初始化，即map还没有桶，那么就新建一个桶
- `bucket := hash & bucketMask(h.B)`定位key应存入桶的位置
- `top := tophash(hash)`获取key的hash的tophash，用来定位cell
- 获取到了桶，那么遍历桶中的cell，判断是否已存在该key，如果存在就更新，不存在就添加
    - 如果cell的tophash与key的top不等，那么判断cell是否为空，如果为空，标记这个桶和cell，意为如果map中不存在该key，那么可以把key存这里
    - 如果cell的tophash与key相等，那么判断该cell的key与我们要添加的key是否相同，如果相同，那么就更新该key的值
- 如果桶中没有改key，那么就去溢出桶寻找，重复上个动作，如果没有找到的话就去添加该key
- 先判断map是否需要扩容
    - 不需要扩容，就找到key应存入的桶，如果之前没有找到一个合适的桶，那么就新建一个溢出桶，然后用该桶的第1个cell来保存key，并更新该cell的tophash。
    - 如果需要扩容，扩容完后，需要重新寻找key的桶
- 找到要插入key的指针地址，对该地址赋值为key的值，然后map的count+1
- 找到该key对应的val的地址
- 再次验证map的状态，并更新map的状态为
- 返回val的地址
- 应该是在汇编层面对val的地址赋值，该函数中没有此操作。



- 定位key对应桶的位置：  
    `bucket := hash & bucketMask(h.B)`，`hash`为key的hash，`bucketMask(h.B)`即`1<<B - 1`，假设B为5，桶的个数为2^5=32个，那么`bucketMask(h.B)`
    即为`11111`，然后进行与操作，如果`hash`的低5位为`10010`,那么结果为`10010`也就是定位到了第18个桶。
- 获取对应桶的内存地址：  
    `b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))`  
    uintptr(h.buckets)为map的桶的地址值，bucket*uintptr(t.bucketsize)为该类型的桶的长度乘以桶的个数，相加就找到了对应桶的内存地址
- 获取对应的cell位置：  
    获取key的hash的tophash
    `top := tophash(hash)`
    ```go
    top := uint8(hash >> (sys.PtrSize*8 - 8))
    if top < minTopHash {
        top += minTopHash
    }
    ```
    `sys.PtrSize`为8，也就是hash右移56位，剩余8位，即取hash的高八位，如果小于最小值`minTopHash`，则加上一个最小值`minTopHash`  
    因为一个桶中有8个cell，通过高八位来定位cell的位置，为什么要大于`minTopHash`？？？？
- 找到要插入的key的位置：  
    `insertk = add(unsafe.Pointer(insertb), dataOffset+inserti*2*sys.PtrSize)      `
    `dataOffset`是map的指针偏移量，`inserti*2*sys.PtrSize`是该cell对应的指针偏移量，`unsafe.Pointer(insertb)`是要插入的桶的地址,相加得到了key的地址
- 找到value的位置：  
    `val := add(unsafe.Pointer(insertb), dataOffset+bucketCnt*2*sys.PtrSize+inserti*uintptr(t.valuesize))`  
    由于map中key放在一起，value放在一起，因此先要计算所有key的偏移量，然后在计算第i个value的偏移量    
- 判断是否要扩容:  
    `!h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B))`  
    先决条件是map没有正在扩容，然后判断map的数量是否超过了过载因子或者map是否有太多的溢出桶
    - 判断map的数量是否超过了过载因子  
        `count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)`  
        `loadFactorNum/loadFactorDen`为6.5，再乘以bucket的总数（`bucketShift(B)`），也就是当map中每个桶中平均的不空cell数量大于6.5，就要扩容。
        因为一个桶最多有8个cell，如果大于了6.5，说明桶中的cell将要被填满，查找效率和插入效率变低？
    - 判断map是否有太多溢出桶
        ```go
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
        B最大为15，`B&15`也就是等于`B`  
        `uint16(1)<<(B&15)`,1左移B位，如果现有的溢出桶的数量不小于“1左移B位”，那么就进行扩容。  
        “1左移B位”就是map中桶的数量，如果溢出桶的数量不小于桶的数量（平均每个桶都有一个溢出桶），联合上层的判断条件，每个桶中的元素数量小于6.5。
        那么说明了有太多的空的cell，影响了key的定位效率。因此要进行扩容。
- 扩容操作：  
    两种情况下需要扩容：1. 桶快满了（超过了过载因子）；2. 桶太空（由于频繁的删除导致）
    ```go
    func hashGrow(t *maptype, h *hmap) {
    	// If we've hit the load factor, get bigger.
    	// Otherwise, there are too many overflow buckets,
    	// so keep the same number of buckets and "grow" laterally.
    	// 定义一个变量来标志桶扩容的倍数，默认1倍，如果是由于桶太空导致的，就变为0倍
    	bigger := uint8(1)
    	if !overLoadFactor(h.count+1, h.B) {
    		bigger = 0
    		h.flags |= sameSizeGrow
    	}
  	    // 存储当前map的buckets
    	oldbuckets := h.buckets
    	// 创建一个新的buckets,如果`h.B+bigger`大于等于4的话，还会获得一个非nil的nextOverflow
    	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
    	// 获取状态标志符
    	flags := h.flags &^ (iterator | oldIterator) 
    	if h.flags&iterator != 0 {
    		flags |= oldIterator
    	}
  	    // 更新map
    	// commit the grow (atomic wrt gc)
    	h.B += bigger
    	h.flags = flags
    	h.oldbuckets = oldbuckets
    	h.buckets = newbuckets
    	h.nevacuate = 0
    	h.noverflow = 0
    	// 如果h.extra.overflow不为空，那么就把这个extra.overflow赋给extra.oldoverflow,extra.overflow置为nil
    	if h.extra != nil && h.extra.overflow != nil {
    		// Promote current overflow buckets to the old generation.
    		if h.extra.oldoverflow != nil {
    			throw("oldoverflow is not nil")
    		}
    		h.extra.oldoverflow = h.extra.overflow
    		h.extra.overflow = nil
    	}
  	    // 如果生成的nextOverflow不为空，那么就把nextOverflow赋给h.extra.nextOverflow
	    // 那么map的extra属性的作用就是存放overflow的，并且只是一个暂存的作用？？
    	if nextOverflow != nil {
    		if h.extra == nil {
    			h.extra = new(mapextra)
    		}
    		h.extra.nextOverflow = nextOverflow
    	}
    	// 上述代码只是分配了新的buckets和把之前的buckets赋给oldbuckets，并没有对新的buckets进行填充，
  	    // 这里代码的意思是实际的赋值实在growWork() and evacuate()
    	// the actual copying of the hash table data is done incrementally
    	// by growWork() and evacuate().
    }
    ```    
- 创建新buckets操作    
```go
// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    // 桶的数量
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.kind&kindNoPointers == 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}
    // 只有在b>4时才会不等，也就是在桶的数量较大时，会提前分配溢出桶
	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```    

- `buckets`迁移代码,每次最多迁移两个桶，防止一次性迁移过多导致的性能问题
```go
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

// runtime/map.go:1136
```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))) // 获取oldbucket
	newbit := h.noldbuckets() // 1向左移旧桶的B个位数，即扩容前的桶的最大数量
	if !evacuated(b) {  // 判断bucket是否扩容完，根据状态判断
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		// x，y分别是旧桶和新桶，先把旧桶的信息放在x里
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))
        // 如果map扩容，长度扩大为1倍（不是因为溢出桶太多导致的扩容），就把新桶直接给y
		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.v = add(y.k, bucketCnt*uintptr(t.keysize))
		}
        // 遍历桶及溢出桶
		for ; b != nil; b = b.overflow(t) {
			// 获取key和value的地址
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			// 遍历cell
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				// 如果key是指针类型，那么对key取值
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/value to bucket x or bucket y).
					hash := t.key.alg.hash(k2, uintptr(h.hash0))  // 计算key的哈希
					// t.key.alg.equal(k2, k2)什么时候为false? 大概只有在k2为NaN的时候吧
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.alg.equal(k2, k2) {
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
                // 当前cell的tophash设置为`evacuatedX + useY`，注意到这个值肯定是小于minTopHash的    
				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				// 当i为8时，创建溢出桶，并将i赋值为0
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy value
				}
				if t.indirectvalue() {
					*(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
				} else {
					typedmemmove(t.elem, dst.v, v)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or value arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		// 最后把溢出桶清除掉
		// Unlink the overflow buckets & clear key/value to help GC.
		if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}
    // 如果oldbucket和扩容的数量相等
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```
流程：  
- 参数：
    - maptype: map的类型
    - hmap：map实体
    - oldbucket
        看一下第三个参数是怎么来的
        ```
        hash := alg.hash(key, uintptr(h.hash0)) // 对key进行hash计算
        bucket := hash & bucketMask(h.B) // 计算hash的低B位（定位桶）
        growWork(t, h, bucket)
        evacuate(t, h, bucket&h.oldbucketmask()) // bucket再与扩容前的B相与，与第二步相似
        ```
        所以bucket就是对桶的定位，那么oldbucket就是对旧桶的定位。只针对一个桶？？？
- 获取旧桶的实体，通过状态判断该桶是否扩充完
    - 如果没扩充完
        - 创建xy来存新桶和旧桶，先把旧桶存入x
        - 如果该扩容不是由于溢出桶太多引起的，也就是由元素数量太多引起的，直接扩容为之前的2倍，那么就直接分配对应地址（新桶的地址）给y
        - 遍历之前的桶和它的溢出桶
            - 遍历桶中的cell
                - 如果key是指针类型，那么就获取其值
                - 如果该扩容是由于元素数量太多引起的，那么计算key的哈希，判断放在旧桶还是新桶
                - 如果该桶已经放满了8个cell，那么就新建一个溢出桶存放数据。
                - 复制值和value到桶中
#### 删除元素
增加代码
```go
var m = map[int]struct{}{1: struct{}{}}

func del()  {
	delete(m, 1)
}

```
编译后,得到关键代码:`CALL    runtime.mapdelete_fast64(SB)`  
这样我们就找到了源码入口. 即`runtime`下的`mapdelete_fast64`函数.  

`runtime/map_fast64.go:272`
```go
func mapdelete_fast64(t *maptype, h *hmap, key uint64) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racewritepc(unsafe.Pointer(h), callerpc, funcPC(mapdelete_fast64))
	}
	// 判断是否为空map
	if h == nil || h.count == 0 {
		return
	}
	// 判断该map是否处于写状态
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	
    // 计算key的哈希
	hash := t.key.alg.hash(noescape(unsafe.Pointer(&key)), uintptr(h.hash0))

	// 设置状态
	// Set hashWriting after calling alg.hash for consistency with mapdelete
	h.flags ^= hashWriting

	// 获取key所在的桶的位置,如果map处在扩容状态,对两个桶进行扩容
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork_fast64(t, h, bucket)
	}
	// 获取key所在的桶
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
search:
	// 遍历该桶及其溢出桶
	for ; b != nil; b = b.overflow(t) {
		// 遍历桶中的key,因为是uint64,因此每个key占8个字节,所以每次加8
		for i, k := uintptr(0), b.keys(); i < bucketCnt; i, k = i+1, add(k, 8) {
			if key != *(*uint64)(k) || isEmpty(b.tophash[i]) {
				continue
			}
			// 如果key为指针,则清空
			// Only clear key if there are pointers in it.
			if t.key.kind&kindNoPointers == 0 {
				memclrHasPointers(k, t.key.size)
			}
			// 桶的地址 + key开始的地址偏移量 + key占用的大小 + 当前key的索引(即value的索引值)*每个值的占得大小 = key对应的value的地址
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*8+i*uintptr(t.valuesize))
			// 如果value为指针,则清空
			if t.elem.kind&kindNoPointers == 0 {
				memclrHasPointers(v, t.elem.size)
			} else {
				memclrNoHeapPointers(v, t.elem.size)
			}
			// 重置ceil的tophah
			b.tophash[i] = emptyOne
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// 如果这是桶中最后一个元素,但是其溢出桶中还有元素,那么直接将map的元素的数量减一并退出循环
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				// 如果这不是桶中的最后一个元素,且该元素的高位元素不为空,那么直接将map的数量减一并退出循环
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			// 上面if else表达的意思是查看该元素后是否还有元素，如果有的话，直接将map的数量减一。如果没有的话，需要做emptyRest标记，即下边的操作
			
			// 下边的for循环表达的是，将当前的ceil所在桶的最后一个溢出桶，往低位ceil遍历（因为已知高位ceil为空），如果ceil为空，则标记为emptyRest，碰到不为空的，终止循环
			for {
			    // 标记当前ceil以及其高位ceil都为空。在该for循环的最后一行，判断了当前ceil为空，因此设置为 emptyRest(当前ceil及高位ceil都为空)
				b.tophash[i] = emptyRest
				if i == 0 {
				    // 如果是key定位到的桶,即key没有在溢出桶，退出当前循环
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// 获取到该链表桶中的最后一个溢出桶
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				// 如果当前ceil不为空,则退出当前循环
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}

	// 再次检查状态:状态被改变了,说明该map被其他操作进行了更改
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	// 更新状态
	h.flags &^= hashWriting
}
```
#### 更新元素
增加代码
```go
func update() {
	var simpleMap = map[string]int{"a": 1}
	simpleMap["a"] = 2
}
```
编译后查看结果
```
0x00a7 00167 (update_map.go:4)  CALL    runtime.mapassign_faststr(SB)
0x00e2 00226 (update_map.go:5)  CALL    runtime.mapassign_faststr(SB)
```
其中go代码中第四行为定义以及赋值操作，第五行为更新操作，但调用的为同一个函数，在上边已经对该函数分析过。  
然后我们发现增加元素、查找元素、更新元素调用的是同一个函数。而我上边查找元素的源码分析错了，分析的是`mapaccess1_faststr`这个函数。。。这个函数还去清楚
在哪里用到了。但是看代码也是对key的定位。

#### 遍历元素
增加代码
```go
func iterator() {
	m := map[int]int{1: 1, 2: 2, 3: 3}
	for v := range m {
		fmt.Printf("%d\t", v)
	}
}
```
编译后得到
```
...
// 0x0162 00354 (rand_map.go:7)    CALL    runtime.mapiterinit(SB)
...
//  0x01fc 00508 (rand_map.go:7)    CALL    runtime.mapiternext(SB)
...
```
`mapiterinit`函数中有这样一段代码
```go
// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
```
代表每次遍历的时候都从一个随机的bucket开始.因此即使不扩容,遍历map获得的元素的顺序也是随机的.


#### 总结
1. map的结构体hmap有9个属性，比较重要的几个为：元素个数、map状态标志、buckets的对数、溢出桶的近似数、哈希值（用来计算key的哈希），指向buckets数组的地址
2. 一个bucket中最多有8个cell，通过key的hash值得低B位来决定存储哪个桶。（B为桶的个数的对数）
3. bucket的结构体为bmap,有5个属性：8个cell的tophash组成数组，8个key组成的数组、8个值组成的数组、溢出桶的地址（将key放一起，value放一起，通过内存对齐可以节省内存空间）

    ![](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61pib1iaeK6CYYicjtlSl0HrycEvYofWxQWP0fnXSqqfwRFKt8HSJ7HP2qic0mqfEv9w82B0Qvpg1OJNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
4. key的哈希的高8位决定key存入哪个cell，即tophash。当桶已满，创建一个溢出桶来存放数据。
    如果一个key的哈希是：`10010111|000011110110110010001111001010100010010110010101010│01010`，并且B为5，那么哈希的后5位为10，前8位为151，那么就会存入第10个桶，设置tophash为151，方便查找。（由于可能存在tophash重复的情况，因此查找时先判读tophash，再判断key的值）
5. 触发map扩容，有两种情况
    - 桶中的元素过多：桶中的已占用的cell的平均值超过过载因子，源码中为6.5
    - 溢出桶过多：大概为溢出桶的数量超过非溢出桶的数量
6. map遍历随机原因（类比为空的房子太多，需要减少房子，和人太多，需要按种类分类）：
    - map添加时需要按照key的哈希的后B位和前8位来确定其位置，并且扩容时还会更改位置。
    - 获取key的value时，会随机选择一个桶作为起始点遍历所有的桶，因此map的遍历是随机的。
7. go中，通过哈希查找表实现map，用链表法解决哈希冲突。
8. 扩容的过程是渐进的，防止一次性搬迁的key过多，引发性能问题。触发扩容的时机是添加新元素，bucket搬迁的时机则发生在赋值、删除期间，每次最多搬迁两个bucket
9. 增改查的操作用的是同一个函数，而删除是另一个函数。增删改调用的函数返回的是value的地址，应该会在汇编层面对其进行赋值操作。
10. map的随机性指的是遍历时的随机性+扩容时重新计算哈希导致可能导致元素顺序的改变.开发者大概是为了让随机性更"明显",所以才在每次遍历的时候随机一个桶开始遍历.
#### 参考文章
- [深度解密Go语言之map-码农桃花源](https://mp.weixin.qq.com/s/2CDpE5wfoiNXm1agMAq4wA)
- [如何实现随机取一个map的key](https://lukechampine.com/hackmap.html)