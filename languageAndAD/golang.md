
### 1. new And make


- what: allocates AND initializes
  - new 
    - (allocates memory).
  - make
    - (allocates and initializes an object of type slice, map, or chan (only)).
- why:

- how:
  - 区别从函数定义就可以看出来:```func new(Type) *Type``` AND ```func make(t Type, size ...IntegerType) Type```
    - 入参值
      - make 只能用来分配及初始化类型为 slice、map、chan 的数据(slice, map, or chan (only))。new 可以分配任意类型的数据；
    - 输出值
      - new 分配返回的是指针，即类型 *Type。make 返回引用，即 Type；
      - new 分配的空间被清零。make 分配空间后，会进行**初始化**；

[外部参考](http://c.biancheng.net/view/5722.html)
#### example

```go
package main
import (
 "fmt"
)
func main() {
 var i *int
 *i=10
 fmt.Println(*i)
}
```

这个例子会打印出什么？0还是10?。以上全错，运行的时候会painc，原因如下：
```
panic: runtime error: invalid memory address or nil pointer dereference
```

>从这个提示中可以看出，对于**引用类型的变量，我们不光要声明它**，还要为它**分配内容**空间，否则我们的值放在哪里去呢？这就是上面错误提示的原因。

对于值类型的声明不需要，是因为已经默认帮我们分配好了。

要分配内存，就引出来今天的new和make。

### 2. 值类型和引用类型的区别

- 2.1. go语言中没有引用类型。
  - [the concept "reference type" has been totally removed from Go spec since Apr 3rd, 2013](https://github.com/golang/go/commit/b34f0551387fcf043d65cd7d96a0214956578f94)


- 2.2. 引用关系是通过指针和各种内置类型的某些可能的内部结构建立的
  - [内部结构的实现定义](https://gfw.go101.org/article/value-part.html)
	```go
	// 映射类型
	type _map *hashtableImpl // 目前，官方标准编译器是使用哈希表来实现映射的。
	// 通道类型
	type _channel *channelImpl
	// 函数类型
	type _function *functionImpl
	type _slice struct {
		elements unsafe.Pointer // 引用着底层的元素
		len      int            // 当前的元素个数
		cap      int            // 切片的容量
	}
	type _string struct {
		elements *byte // 引用着底层的byte元素
		len      int   // 字符串的长度
	}
	type _interface struct {
		dynamicType  *_type         // 引用着接口值的动态类型
		dynamicValue unsafe.Pointer // 引用着接口值的动态值
	}
	type _interface struct {
		dynamicTypeInfo *struct {
			dynamicType *_type       // 引用着接口值的动态类型
			methods     []*_function // 引用着动态类型的对应方法列表
		}
		dynamicValue unsafe.Pointer // 引用着动态值
	}
	```

- Question
  - ![20201216201044](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201216201044.png)
  - 但是为什么使用```%p```来打印的时候这些常规认为的“引用类型”,可以打印，答应我们认为的值类型的时候，出错。
    - 从下面的gdb例子中可以看出，我们能打印这些常规认为的“引用类型”,是因为```Slice: %p	address of 0th element in base 16 notation, with leading 0x```, 它实际打印的是第0个元素的地址。而```&test```是打印的它实际的```struct的地址```。
        ```go
        package main
        import (
        	"fmt"
        	"unsafe"
        )
        func testS(test []int) {
        	// test:0xc000078150
        	fmt.Printf("---test:%p\n", test)
        	// &test:0xc000098028
        	fmt.Printf("&test:%p\n", &test)

        	fmt.Println()
        }
        func main() {
        	s := []int{1, 2}
        	fmt.Println(unsafe.Sizeof(s))
        	fmt.Printf("s---:%p\n", s)
        	fmt.Printf("&s[0]---:%p\n", &s[0])
        	fmt.Printf("&s:%p\n", &s)
        	testS(s)
        }
        ```
    - ![20201216194723](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201216194723.png)

[Package fmt](https://golang.org/pkg/fmt/)
[There Are No Reference Types in Go](https://www.tapirgames.com/blog/golang-has-no-reference-values)
[About the terminology "reference type" in Go](https://github.com/go101/go101/wiki/About-the-terminology-%22reference-type%22-in-Go)

### 计算golang中类型的大小的方式

- struct会不会有填充的（padding）的bit?
  - 会的，就像下方的例子中，T.B就是一个bit,但是它后面填充了7个bit.
- 类似sizeof的函数？
  - ```unsafe.Sizeof()```
- 如果是分配在heap中，还可以使用什么方式来评估？
  ```go
  	runtime.ReadMemStats(&m1)// 好像一个snapshot,可以评估当前heap的空间.
	  t := T{}
	  runtime.ReadMemStats(&m2)
  ```

```go
package main

import (
	"fmt"
	"runtime"
	"strconv"
	"unsafe"
)

// Unlike C, there is no _#pragma pack_ in Go, the real memory allocation
// depends on its implementation.
type T struct {
	B  uint8 // is a byte
	I  int   // it is int32 on my x86 32 bit PC
	P  *int  // it is int32 on my x86 32 bit PC
	S  string
	SS []string
}

var p = fmt.Println

// In this case, the `t := T{}` can not measured by this method.
func memUsage(m1, m2 *runtime.MemStats) {
	p("Alloc:", m2.Alloc-m1.Alloc,
		"TotalAlloc:", m2.TotalAlloc-m1.TotalAlloc,
		"HeapAlloc:", m2.HeapAlloc-m1.HeapAlloc)
}

func main() {
	// Here is a tricky to get pointer size
	const PtrSize = 32 << uintptr(^uintptr(0)>>63)
	p("PtrSize=", PtrSize)
	p("IntSize=", strconv.IntSize)

	var m1, m2, m3, m4, m5, m6 runtime.MemStats
	runtime.ReadMemStats(&m1)
	t := T{}
	runtime.ReadMemStats(&m2)
	p("sizeof(uint8)", unsafe.Sizeof(t.B),
		"offset=", unsafe.Offsetof(t.B))
	p("sizeof(int)", unsafe.Sizeof(t.I),
		"offset=", unsafe.Offsetof(t.I))
	p("sizeof(*int)", unsafe.Sizeof(t.P),
		"offset=", unsafe.Offsetof(t.P))
	p("sizeof(string)", unsafe.Sizeof(t.S),
		"offset=", unsafe.Offsetof(t.S))

	// Slice is a structure of Pointer, Len and Cap.
	// Detail [here](http://blog.golang.org/go-slices-usage-and-internals)
	p("sizeof([]string)", unsafe.Sizeof(t.SS),
		"offset=", unsafe.Offsetof(t.SS))

	// We can see the this structure is 4 + 4 + 4 + 8 + 12 = 32 bytes
	// There are 3 padding bytes of first t.B expanded to 4 bytes.
	p("sizeof(T)", unsafe.Sizeof(t))

	// We will see 0 bytes, because it is on stack, so sizeof is the
	// proper method to tell how much memory allocated.
	memUsage(&m1, &m2)

	// Even string assignment is in stack.  also is zero
	runtime.ReadMemStats(&m3)
	t2 := "abc"
	runtime.ReadMemStats(&m4)
	memUsage(&m3, &m4)

	// map will alloc memory in heap
	runtime.ReadMemStats(&m5)
	t3 := map[int]string{1: "x"}
	runtime.ReadMemStats(&m6)
	memUsage(&m5, &m6)

	fmt.Println(t2, t3) // prevent compiler error
}
```




### golang中同步的方式

- what:
  - 并发同步是指如何控制若干并发计算（在Go中，即协程），从而
    - 避免在它们之间产生数据竞争的现象；
    - 避免在它们无所事事的时候消耗CPU资源。
  - 并发同步有时候也称为数据同步。
- why:
  - 避免在它们之间产生数据竞争的现象；
  - 避免在它们无所事事的时候消耗CPU资源。
- how:
  - 通道用例大全
    - ![20201217212124](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201217212124.png)
  - 如何优雅地关闭通道
  - [sync标准库包中提供的并发同步技术](https://gfw.go101.org/article/concurrent-synchronization-more.html)
    - sync.WaitGroup（等待组）类型: ：Add(delta int)、Done()和Wait()
	- sync.Once类型: Do(f func())方法
	  - 一个sync.Once值被用来确保一段代码在一个并发程序中被执行且仅被执行一次
    - sync.Mutex（互斥锁）和sync.RWMutex（读写锁）类型: 实现了```sync.Locker```接口类型。 所以这两个类型都有两个方法：Lock()和Unlock()，
	  - 用来保护一份数据不会被多个使用者同时读取和修改。
	  - *sync.RWMutex类型还有两个另外的方法：RLock()和RUnlock()，
	    - 用来支持多个读取者并发读取一份数据但防止此份数据被某个数据写入者和其它数据访问者（包括读取者和写入者）同时使用
    - sync.Cond类型:提供了一种有效的方式来实现多个协程间的**通知**。
	  - 成员： sync.Locker类型的名为L的字段
	  - 成员函数： Wait()、Signal()和Broadcast()。
	    - 每个Cond值维护着一个先进先出等待协程队列。 对于一个可寻址的Cond值c，
		  - c.Wait()必须在c.L字段值的锁被成功获取的时候调用；否则，c.Wait()调用将造成一个恐慌。 一个c.Wait()调用将
			- 1. 首先将当前协程入到c所维护的等待协程队列；
			- 2. 然后调用c.L.Unlock()释放c.L的锁；
			- 3. 然后使当前协程进入阻塞状态；（当前协程将被另一个协程通过c.Signal()或c.Broadcast()调用唤醒而重新进入运行状态。）
			  - 一旦当前协程重新进入运行状态，c.L.Lock()将被调用以试图重新获取c.L字段值的锁。 此c.Wait()调用将在此试图成功之后退出。
	      - 一个c.Signal()调用将唤醒并移除c所维护的等待协程队列中的第一个协程（如果此队列不为空的话）。
		  - 一个c.Broadcast()调用将唤醒并移除c所维护的等待协程队列中的所有协程（如果此队列不为空的话）。

  - [sync/atomic标准库包中提供的原子操作技术](https://gfw.go101.org/article/concurrent-atomic-operation.html)
    - ```T```---可以换成--->内置```int32、int64、uint32、uint64和uintptr```类型
	  ```go
	  func AddT(addr *T, delta T)(new T)
	  func LoadT(addr *T) (val T)
	  func StoreT(addr *T, val T)
	  func SwapT(addr *T, new T) (old T)
	  func CompareAndSwapT(addr *T, old, new T) (swapped bool)
	  ```
  - 我们也可以利用网络和文件读写来做并发同步，但是这样的并发同步方法使用在一个程序进程内部时效率相对比较低。 一般来说，这样的方法多适用于多个进程之间或多个主机之间的并发同步。《Go语言101》中不介绍这样的并发同步方法。


为了更好地理解各种并发同步技术，推荐在适当的时候（具有一定的Go并发编程经验时）阅读Go中的内存顺序保证一文。

- Go中的内存顺序保证
  - 内存顺序
    - what: 调整指令执行顺序，从而使得指令执行顺序和代码中指定的顺序不太一致。 指令顺序也称为内存顺序。
      - ![20201217193316](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201217193316.png)
	- why:
	- how:
	  - 通道操作相关的顺序保证:
	    - 1. 一个通道上的第n次成功发送操作的开始发生在此通道上的第n次成功接收操作完成之前，无论此通道是缓冲的还是非缓冲的。
		  - **如果是缓冲，一边是发送，数据进入到通道中，当另一边接收的时候，肯定在前一边之后。**
		- 2. **一个容量为m通道上的第n次成功接收操作的开始发生在此通道上的第n+m次发送操作完成之前**。 特别地，如果此通道是非缓冲的（m == 0），则此通道上的第n次成功接收操作的开始发生在此通道上的第n次发送操作完成之前。
		- 3. 一个通道的关闭操作发生在任何因为此通道被关闭而从此通道接收到了零值的操作完成之前。
      - 事实上， 对一个非缓冲通道来说，其上的第n次成功发送的完成和其上的第n次成功接收的完成应被视为同一事件。
	    - 好像他们就是保证当阻塞发送接收的时候，**他们可以当成一个事件，同时发生的。**
      - example:
	    - ![20201217195138](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201217195138.png)



### golang逃逸分析

- what:
  - 逃逸分析是一种确定指针动态范围的方法，可以分析在程序的哪些地方可以访问到指针。
    - 如果一个子程序分配一个对象并返回一个该对象的指针，该对象可能在程序中的任何一个地方被访问到——这样指针就成功“逃逸”了。
  - go消除了堆和栈的区别
    - go在一定程度消除了堆和栈的区别，因为go在编译的时候进行逃逸分析，来决定一个对象放栈上还是放堆上，不逃逸的对象放栈上，可能逃逸的放堆上。
- why:
  - 不逃逸的对象放栈上，可能逃逸的放堆上:
    - gc的压力: 最大的好处应该是减少gc的压力，不逃逸的对象分配在栈上，当函数返回时就回收了资源，不需要gc标记清除。
    - 因为逃逸分析完后可以确定哪些变量可以分配在栈上，栈的分配比堆快，性能好
  - 同步消除，如果你定义的对象的方法上有同步锁，但在运行时，却只有一个线程在访问，此时逃逸分析后的机器码，会去掉同步锁运行。
- how:
  - ```-gcflags '-m -l'```	 
  - ![20201217210216](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201217210216.png)
    - 从内部函数开始分析，它的好处的是，如果底部的函数中都没有逃逸，那么最上层的变量就直接放入栈里面就行了，不需要放堆里。
	  - 这里的objS没有逃逸，而m却逃逸了，这是因为go的逃逸分析不知道objS和m的关系，逃逸分析不知道参数M是```struct S```的一个成员，所以只能把它分配给堆。

[Golang逃逸分析](https://cloud.tencent.com/developer/article/1165660)
[GoLang-逃逸分析](https://www.jianshu.com/p/ad9dbc81a0aa)


