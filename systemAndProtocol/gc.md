
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
  - 黑色：当基于当前节点的追踪任务完成后，表示为存活数据，后续无需在追踪。
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
		- 既可忽略当前栈帧的写屏障。
		- 又不需要在第二次STW的时，重新扫描所有活跃G的栈帧。

