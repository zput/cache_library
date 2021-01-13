
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



“同GC并发执行的用户程序，源码与GC相关书籍中都称其为“mutator”（赋值器）”



- QA:谈一谈golang的gc.
- 一：Golang中GC的实现采用的是标记——清扫算法，支持增量与并发式回收。
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


- 二：为什么清扫阶段不需要屏障了呢？
        - 当标记完成了，那么它白色的对象都是不可达的对象，是可以删除的对象，程序不可能再找到已经不可达的对象。所以放心的清除。

- 三：golang的heap结构
    - ![20210113134838](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113134838.png)

    - 03. Golang中GC的三色标记
        -（1）着为灰色对应的操作就是把指针对应的gcmarkBits标记位置为1并**加入工作队列**；
        -（2）着为黑色对应的操作就是把指针对应的gcmarkBits标记位置为1。
        -（3）白色对象就是那些gcMarkBits中标记为0的对象。


问题**工作队列**相关的问题：并发标记的分工问题？写屏障记录集的竞争问题？
    - 前面提到了全局变量work中存储着全局工作队列缓存（work.full），其实每个P都有一个本地工作队列（p.gcw）和一个写屏障缓冲（p.wbBuf）。
    - p.gcw中有两个workbuf：wbuf1和wbuf2，添加任务时总是从wbuf1添加，wbuf1满了就交换wbuf1和wbuf2，如果还是满的，就把当前wbuf1的工作flush到全局工作缓存中去。


- 四： CUP utilization
    - GC默认的CPU目标使用率为25%，在GC执行的初始化阶段，会根据当前CPU核数乘以CPU目标使用率来计算需要启动的```mark worker```数量。
    - ![20210113134856](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210113134856.png)

问题04.
并发GC如何缓解内存分配压力？


























