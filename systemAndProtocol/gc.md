

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [扩展](#扩展)
  - [mutator?](#mutator)
  - [FQA](#fqa)
    - [谈一谈golang的gc.](#谈一谈golang的gc)
    - [二：为什么清扫阶段不需要屏障了呢？](#二为什么清扫阶段不需要屏障了呢)
    - [三：golang的heap结构](#三golang的heap结构)
    - [四: **工作队列**相关的问题：并发标记的分工问题？写屏障记录集的竞争问题？](#四-工作队列相关的问题并发标记的分工问题写屏障记录集的竞争问题)
    - [五： CUP utilization](#五-cup-utilization)
    - [TODO: 并发GC如何缓解内存分配压力？](#todo-并发gc如何缓解内存分配压力)
- [附录](#附录)
  - [内存分配器](#内存分配器)
    - [heapArena](#heaparena)
    - [span](#span)

<!-- /code_chunk_output -->




为什么gc也要扫描栈，不是都在堆上面吗？
- 因为堆上的地址，可能保存在栈上某个变量里，所以需要扫描。

- 一个是插入写屏障
  - 破坏第一个条件
  - 记住插入，---黑色----白色===》 黑色----灰色
    ```go
    writePointer(slot, ptr):
 	  shade(ptr) //shade函数尝试改变指针的颜色-->改变ptr的颜色
	  *slot = ptr
	```

---

- 一个是删除写屏障
  - 破坏第二个条件。
  - 删除
  ```go
  writePointer(slot, ptr)
  	shade(*slot) //shade函数尝试改变指针的颜色-->改变slot的颜色--->注意这个是*slot,slot保存着它原先的保存的地址。
  	*slot = ptr
  ```
  - disadvantage:
    - which means that many stacks must be re-scanned during STW. The garbage collector first scans all stacks at the beginning of the GC cycle to collect roots.(第一次扫描所有的栈在收集根对象的时候，不能保证在栈后面不会有黑对象引用了白对象)

---

- 混合写屏障
  - what: 混合写屏障为了**消除栈的重扫**过程.
  ```go
  writePointer(slot, ptr):
      shade(*slot)
      if current stack is grey:
          shade(ptr)
      *slot = ptr
  ```
  - why: 

  - how: 通过几种方式来保证弱三色一致性
    - STW 扫描一次协程栈(扫一个Goroutine栈就变为黑色) + 创建对象默认黑色
      - 开启写屏障期间创建的所有对象默认都是黑色
    - 混合写屏障 + 两Goroutine之间交流

---	
>  - The hybrid barrier requires that objects be allocated black (allocate-white is a common policy, but incompatible with this barrier). 混合写屏障的时候要求新对象被分配为黑色（这里我猜是所有的对象，栈对象，或者堆对象也好）
>  - once a stack has been scanned and blackened, it remains black. 一旦栈被扫描（这个的扫描是第一次扫描，查找根对象的时候）和置为黑，那么它一直保持为黑
>    - 说明栈在第一次扫描的时候就会把栈上对象置为黑色？
>  - In the hybrid barrier, the two shades and the condition work together
>    - shade(*slot): 从heap到stack移动白色对象，染为灰色。（尝试如果它试图从heap中解开一个对象的链接，就会对其进行遮挡。）
>	  - shade(ptr): 从stack到一个黑色的heap对象。
>	  - Once a goroutine's stack is black, the shade(ptr) becomes unnecessary. shade(ptr) prevents hiding an object by moving it from the stack to the heap, but this requires first having a pointer hidden on the stack. Immediately after a stack is scanned, it only points to shaded objects, so it's not hiding anything, and the shade(*slot) prevents it from hiding any other pointers on its stack.
>  - 一个Goroutine写到另一个Goroutine:



  ```
  我就是觉得混合写屏障好像也没法 解决重新扫描栈的问题。我举个例子：
  现在有 A, B, C三个对象，A(黑色，栈上)，B（灰色，栈上），C（白色，堆上）；
  当前引用关系是：
  A（黑） -> nil
  B（灰） -> C（白）
  现在应用程序赋值修改，把A指向C：
  A（黑） -> C（白）
  B（灰） -> nil
  由于A，B是栈上的对象，栈上对象赋值这里可是没有写屏障的；那么岂不是黑色对象指向白色对象了，C会回收了，就悬挂指针了？？？
  ```

>   - Goroutine 栈扫描的过程需要 STW，所以你描述的这种状况是不存在的，**栈上的对象要么全白要么全黑**
>     - 你说的“栈上的对象要么全白，要么全黑“ ，这个只是对一个 goroutine 栈来说的（golang 暂停业务扫描栈也是一个一个来的）。如果场景是 A 在 Goroutine1，B在Goroutine2呢？这种情况就是A是黑色，B是白色或者灰色。这样会不会就有我说的原本那个问题呢？
>       - [The hybrid barrier assumes a goroutine cannot write to another goroutine's stack.(混合写屏障假设一个goroutine,不能写到另一个goroutine的栈)](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md#channel-operations-and-go-statements)
>     - 除了从一个Goroutine通过chan发送/生成一个新Goroutine
>	      - 如果两个Goroutine 栈只要有一个是灰色的，那么就会有```shade(ptr)```
>	      - 新生成的Goroutine栈都是黑色的（由前面的条件保证）,如果父Goroutine是灰色的，那么需使用```shade(ptr)```
>  





https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md


---


gc:

what:
gc主要是释放那些不再需要的分配在堆（heap）上的数据

why:
降低人的心智负担，从堆（heap）中申请的内存，不需要手动的释放。

how:
- 一般判断对象是否存活是使用：是把可达性近视的认为存活性.
  - 可把栈（stack），数据段（data segment? bss?）的数据对象作为root

- Mark-Sweep.
  - STW(stop the world)

- 三色标记法。
  - 黑色：当基于当前节点的追踪任务完成后，表示为存活数据，后续无需再追踪。
  - 灰色：基于当前节点展开的追踪还未完成。
  - 白色：

- conflict:
	- 当gc使用三色标记法与用户程序并发执行的时候，可能让黑色对象重新引用了白色对象，且无其他灰色对象引用了这个白色对象。导致白色对象错误回收。(用户程序和垃圾收集器可以在交替工作的)
	  - 强三色：黑色对象只能引用灰色对象。
	  - 弱三色：所有被黑色对象引用的白色对象都处于灰色保护状态.

- 读写屏障。
  - 读屏障：非移动式垃圾回收器中，天然不需要读屏障。
  - 写屏障：会在写操作中插入指令，把数据对象的修改通知到垃圾回收器。
    - 插入写屏障：
	```
	writePointer(slot, ptr):
   	 	shade(ptr) //shade函数尝试改变指针的颜色-->改变ptr的颜色
    	*slot = ptr
	```
	- 删除写屏障： 会在老对象的引用被删除时，将白色的老对象涂成灰色。
	```
	writePointer(slot, ptr)
    	shade(*slot) //shade函数尝试改变指针的颜色-->改变slot的颜色
    	*slot = ptr
	```
	- conflict: 只有插入写屏障，但是这需要对所有堆、栈的写操作都开启写屏障，代价太大。为了改善这个问题，改为**忽略协程栈**上的写屏障，只在标记结束阶段重新扫描那些被激活的栈帧。但是Go语言通常会有大量活跃的协程，这就导致第二次STW时重新扫描协程栈的时间太长。
		- 混合写屏障：将被覆盖的对象(老对象)标记成灰色并在当前栈没有扫描时将新对象也标记成灰色。
		```
		 writePointer(slot, ptr):
    		shade(*slot) //将老对象标记为灰色。
  			if current stack is grey: // The insertion part of the barrier is necessary while the calling goroutine's stack is grey.
   		    	shade(ptr)
  			*slot = ptr
		```
		- 既可**忽略当前栈帧的写屏障**。(不管是插入写屏障，还是删除写屏障.)
          - 这里模拟很容易，黑色关联白色，且没有其他灰色关联这个黑色；就会出现hiding object。
            - 模拟heap上触发**删除写屏障**。---》所以stack上一个black object关联heap上面的white object；这时候heap上的其他颜色object断开与它的连接。
            - 模拟heap上触发**插入写屏障**。
              - ![gc](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/gc.png)
		- 又不需要在第二次STW的时，重新扫描所有活跃G的栈帧。

堆（heap）上面的都有插入写屏障，不会发生hiding object。



## 扩展

### mutator?

“同GC并发执行的用户程序，源码与GC相关书籍中都称其为“mutator”（赋值器）”

### FQA

#### 谈一谈golang的gc.

- 一：Golang中GC的实现采用的是标记——清扫算法，支持增量与并发式回收。非完全并发操作
   - 标记清除，不是像原始的，STW（stop the world）, 一个是暂停的时间太长，二个是用户的程序需暂停。它采用的是基于三色标记的混合写屏障。
   - 增量和并发，一个是横向一个是纵向，gc与mutator交替与运行。所以这里也需要屏障技术。
     - 横向：增量垃圾收集 — 增量地标记和清除垃圾，降低应用程序暂停的最长时间；
     - 纵向：并发垃圾收集 — 利用多核的计算资源，在用户程序执行时并发标记和清除垃圾；
       - 因为增量和并发两种方式都可以与用户程序交替运行，所以我们需要使用屏障技术保证垃圾收集的正确性.

- 使用混合写屏障的原因是缩短gc暂停的时间。
  - 因为栈上使用写屏障，会导致耗时太多。但是如果栈上不使用写屏障，等到第二次STW重新扫描栈空间，goroutine数目多，需要扫描的stack耗时也多。
忽略协程栈的写屏障;其他的使用删除写屏障，插入写屏障。

- 如果没有任何STW的时间，也就是说垃圾回收程序与用户程序完全并发执行，其代价与实现难度可能都会高于短暂的STW。例如标记——清扫回收器中，若完全抛弃STW，那么垃圾回收开始的消息便很难准确及时地通知到所有线程，可能导致**某些线程**开启**写屏障**的动作有所延迟而无法保障双方执行的正确性。

![gc](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113105703.png)


#### 二：为什么清扫阶段不需要屏障了呢？

   - 当标记完成了，那么它白色的对象都是不可达的对象，是可以删除的对象，程序不可能再找到已经不可达的对象。所以放心的清除。

#### 三：golang的heap结构
  - ![20210113134838](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113134838.png)
  - 参见下面的内存分配器
    - 03. Golang中GC的三色标记
        -（1）着为灰色对应的操作就是把指针对应的```gcmarkBits```标记位置为1并**加入工作队列**；
        -（2）着为黑色对应的操作就是把指针对应的```gcmarkBits```标记位置为```1```。
        -（3）白色对象就是那些```gcMarkBits```中标记为0的对象。

知道标记在哪了，那么如果进行分工？
#### 四: **工作队列**相关的问题：并发标记的分工问题？写屏障记录集的竞争问题？

![20210121134453](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210121134453.png)

>这里的竞争问题，我怀疑是： 一个是gc从root Object到一直遍历，打标记(从上到下)，同时另一个用户程序也在更新Object的关联（比如从从一个灰色到白色节点断开）
  >>其中用户程序的写到每个P中的写屏障缓冲区。

- 前面提到了全局变量work中存储着全局工作队列缓存（work.full），其实每个P都有**一个本地工作队列（p.gcw）和一个写屏障缓冲（p.wbBuf）**。
- p.gcw中有两个workbuf：wbuf1和wbuf2，添加任务时总是从wbuf1添加，wbuf1满了就交换wbuf1和wbuf2，如果还是满的，就把当前wbuf1的工作flush到全局工作缓存中去。


知道分工了，不可能占用很多CPU进行gc,这样会限制用户程序。
#### 五： CUP utilization

   - GC默认的CPU目标使用率为25%，在GC执行的初始化阶段，会根据当前CPU核数乘以CPU目标使用率来计算需要启动的```mark worker```数量。
   - ![20210113134856](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113134856.png)


#### TODO: 并发GC如何缓解内存分配压力？
  - 借贷偿还机制。也可以偷。


## 附录

### 内存分配器

![gc_heap](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/gc_heap.png)

```go
//go:notinheap
type mheap struct {
//...    
	// arenas is the heap arena map. It points to the metadata for
	// the heap for every arena frame of the entire usable virtual
	// address space.
	//
	// Use arenaIndex to compute indexes into this array.
	//
	// For regions of the address space that are not backed by the
	// Go heap, the arena map contains nil.
	//
	// Modifications are protected by mheap_.lock. Reads can be
	// performed without locking; however, a given entry can
	// transition from nil to non-nil at any time when the lock
	// isn't held. (Entries never transitions back to nil.)
	//
	// In general, this is a two-level mapping consisting of an L1
	// map and possibly many L2 maps. This saves space when there
	// are a huge number of arena frames. However, on many
	// platforms (even 64-bit), arenaL1Bits is 0, making this
	// effectively a single-level map. In this case, arenas[0]
	// will never be nil.
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena
//...
}

//A heapArena stores metadata for a heap arena.
// 每一个heapArena管理一个heap arena.
type heapArena struct {
	bitmap [heapArenaBitmapBytes]byte
	spans [pagesPerArena]*mspan
	pageInUse [pagesPerArena / 8]uint8
	pageMarks [pagesPerArena / 8]uint8
	zeroedBase uintptr
}

	type mspan struct {
		next *mspan
		prev *mspan
		//...

		startAddr uintptr // 起始地址
		npages    uintptr // 页数
		freeindex uintptr

		allocBits  *gcBits
		gcmarkBits *gcBits
		allocCache uint64

	    //startAddr 和 npages — 确定该结构体管理的多个页所在的内存，每个页的大小都是 8KB；
	    //freeindex — 扫描页中空闲对象的初始索引；
	    //allocBits 和 gcmarkBits — 分别用于标记内存的占用和回收情况；
	    //allocCache — allocBits 的补码，可以用于快速查找内存中未被使用的内存；

		//...
	}
```
[mheap](https://github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go#L152)

[heapArena](https://github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go#L217)

#### heapArena

![20210120104547](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210120104547.png)
- mheap中每个arena对应一个HeapArena，记录arena的元数据信息。HeapArena中有一个**bitmap和一个spans**字段。
  - 1.bitmap
    - bitmap中每两个bit对应标记arena中一个指针大小的word，也就是说bitmap中**一个byte**可以标记**arena中连续四个指针大小**的内存。
      - 每个word对应的两个bit中，**低位bit用于标记是否为指针**，0为非指针，1为指针；**高位bit用于标记是否要继续扫描**，```高位bit为1```就代表扫描完当前word并不能完成当前数据对象的扫描。
  - 2.spans
    - spans是一个*mspan类型的数组，用于记录当前arena中每一页对应到哪一个mspan。(看这个mspan的结构可以知道，它有startAddr与npages,说明一个mspan管理多个page)

>基于HeapArena记录的元数据信息，我们只要知道一个对象的地址，
>>就可以根据HeapArena.bitmap信息扫描它内部是否含有指针；
>>也可以根据对象地址计算出它在哪一页，然后通过HeapArena.spans信息查到该对象存在哪一个mspan中。


#### span

- 而每个span都对应两个位图标记：mspan.allocBits和mspan.gcmarkBits。
  - （1）allocBits中每一位用于标记一个对象存储单元是否已分配。
    - ![20210113202836](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113202836.png)
  - （2）```gcmarkBits```中每一位用于标记**一个对象是否存活**。
    - ![20210113202858](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113202858.png)  



