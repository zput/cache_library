
what: 是golang内部自己实现的调度器，由’‘G’’,“M”,“P"用来调度goruntine，被称为"GMP模型”。

why:

- 单进程时代不需要调度器
  - 1.单一的执行流程，计算机只能一个任务一个任务处理。
  - 2.进程阻塞所带来的CPU时间浪费。

- 多进程/线程时代有了调度器需求
  - 1.解决了阻塞的问题
  - 2.CPU有很大的一部分都被浪费在进程调度
  - 设计会变得更加复杂，要考虑很多同步竞争等问题，如锁、竞争冲突等。

- 协程(用户线程)来提高CPU利用率(减少CPU浪费在进程调度上)

> - 为什么。
>   - 线程和进程有很多相同的控制权。线程有自己的信号掩码，可以分配CPU时间，可以放入cgroups，可以查询它们使用了哪些资源。所有这些控制都增加了一些功能的开销，而这些功能对于Go程序如何使用goroutine来说是根本不需要的，而且当你的程序中有10万个线程时，它们很快就会增加。
>   - Go调度器可以做出只在它知道内存是一致的点进行调度的决定。
> - 如何进行调度。
>   - 一种是N:1，即在一个操作系统线程上运行几个用户空间线程。这样做的好处是上下文切换非常快，但不能利用多核系统的优势。
>   - 另一种是1:1，一个执行线程匹配一个OS线程。它可以利用机器上所有的核心，但是上下文切换很慢，因为它要通过OS进行切换。
>   - M:N调度器。你可以得到快速的上下文切换，你可以利用系统中所有的核心。这种方法的主要缺点是它增加了调度器的复杂性。
> - 摆脱上下文(在这里P就是上下文)？
>   - ~~不行。我们使用上下文的原因是，如果正在运行的线程由于某些原因需要阻塞，我们可以将它们移交给其他线程。~~
>   - 以前就只有一个全局的P，也可以运行。必须要有P（上下文），是有什么值保存在里面？  

> - why:
>   - threads get a lot of the same controls as processes. Threads have their own signal mask, can be assigned CPU affinity, can be put into cgroups and can be queried for which resources they use. All these controls add overhead for features that are simply not needed for how Go programs use goroutines and they quickly add up when you have 100,000 threads in your program.
>   - Go调度器可以做出只在它知道内存是一致的点进行调度的决定。
> - how:
>   - One is N:1 where several userspace threads are run on one OS thread. This has the advantage of being very quick to context switch but cannot take advantage of multi-core systems.
>   - Another is 1:1 where one thread of execution matches one OS thread. It takes advantage of all of the cores on the machine, but context switching is slow because it has to trap through the OS.
>   -  M:N scheduler. You get quick context switches and you take advantage of all the cores in your system. The main disadvantage of this approach is the complexity it adds to the scheduler.
> - get rid of contexts?
>   - Not really. The reason we have contexts is so that we can hand them off to other threads if the running thread needs to block for some reason.

---


>这个也就说明了```N-M```的基础，用户线程的各自栈空间其实就是放在公共的堆（heap）上。
>>每个系统线程都有一个唯一的m0, g0与之对应，想想为什么？(g0的栈空间与其他的不同，它是放在系统线程的栈空间，**应该是进程的栈空间？**TODO,线程的栈空间)
>>>每个线程有自己的栈空间(而这个g0就在这个栈上)。但是是与其他的线程公用的~~代码段~~，**数据段，堆空间**,
>>>>所以当创建其他的goroutine的时候，把它的协裎栈在堆上，所以它可以被其他的M调用。


how:

- Go调度本质是把大量的goroutine分配到少量线程上去执行，并利用多核并行，实现更强大的并发。
  - 通过这一点去记住，把大量goroutine分配到小量线程去尽快执行
    - 复用
    - 并发
    - 防止饥饿
    - 

> - 调度器的有两大思想：[<sup>1</sup>](#zxc-anchor-1)
>   - 复用线程：协程本身就是运行在一组线程之上，不需要频繁的创建、销毁线程，而是对线程的复用。在调度器中复用线程还有2个体现：
>     - **work stealing**，当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。
>     - **hand off**，当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行。
>   - 利用并行：GOMAXPROCS设置P的数量，当GOMAXPROCS大于1时，就最多有GOMAXPROCS个线程处于运行状态，**这些线程可能分布在多个CPU核上同时运行**，使得并发利用并行。另外，GOMAXPROCS也限制了并发的程度，比如GOMAXPROCS = 核数/2，则最多利用了一半的CPU核进行并行。
>     - golang并发和并行：Rob Pike一直在强调Go是并发，不是并行，因为Go做的是在一段时间内完成几十万、甚至几百万的工作，而不是同一时间同时在做大量的工作。并发可以利用并行提高效率，调度器是有并行设计的。
> - 调度器的两小策略：
>   - 抢占：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，**这就是goroutine不同于coroutine的一个地方**。
>   - 全局G队列：在新的调度器中依然有全局G队列，但功能已经被弱化了，当M执行work stealing从其他P偷不到G时，它可以从全局G队列获取G。






<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [FQA](#fqa)
  - [g0,m0,gM](#g0m0gm)
  - [说一说GMP的底层实现。](#说一说gmp的底层实现)
    - [从程序加载到main函数执行的过程。](#从程序加载到main函数执行的过程)
    - [新的goroutine创建的过程newproc()。](#新的goroutine创建的过程newproc)
    - [调度的过程schedule()。](#调度的过程schedule)
- [archive](#archive)
- [附录](#附录)

<!-- /code_chunk_output -->



## FQA


### g0,m0,gM

> 我们知道g0,m0是开始就创建的，那么后面创建了gM，它是新建一个M，还是使用m0？
>> 从图中可以看到是使用m0:```https://zput.github.io/go-goroutine/part1.%E8%BF%9B%E5%85%A5main%E5%87%BD%E6%95%B0%E4%B9%8B%E5%89%8D%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96/1.%E8%BF%9B%E5%85%A5main%E5%87%BD%E6%95%B0%E4%B9%8B%E5%89%8D%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96.html#mstart```
>> 一个M对应一个系统线程，那么你想在程序开始的时候就会进行又创建一个系统线程？

### 说一说GMP的底层实现。

- ![GMP大全](https://raw.githubusercontent.com/zput/myPicLib/master/GO/goroutine%E8%B0%83%E5%BA%A6.png)

#### 从程序加载到main函数执行的过程。

- 两条主线 + GMP经典图：
  - 那个经典的GMP在这一阶段实际是怎么表示的，(g0,m0,allp[0]),
  - 代码加载到内存中的code segment, 从```入口-->runtime.main-->main.main```.
  - 从全局变量的角度，全局变量
    - ```(g0,m0,sched)```; --->>> ```sched(midle, pidle, runq(全局运行goroutine queue))```.
    - ```(allg,allm,allp[0,...])```;
- 进行```mcall()-->schedule()```.

![20210114101912](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210114101912.png)


> 从上面就引出了创建新G，与调度问题：```newproc(),与mcall()--->schedule()```

#### 新的goroutine创建的过程newproc()。

  - 创造一个新**goroutine**[```go myfunc()```]:
    - 它的最终结果一定是: 就像是goexist()函数调用这个```myfunc函数一样```。
    - 创造一个新的goroutine前会在全局里面找看是否有```gidle的goroutine```.
    - 最后会```runput```到队列中（可能是本地，可能是全局）。

    ![20210114142644](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210114142644.png)

   ![c05b9cda-5f30-4f90-9cd5-379a28005a86](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/c05b9cda-5f30-4f90-9cd5-379a28005a86.jpg)

   ![newproc的过程](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/748394d1-2bc0-4cbc-b8c4-ef019908a993.jpg)


#### 调度的过程schedule()。

TODO
[TEXT runtime·gogo(SB), NOSPLIT|NOFRAME, $0-8](https://github.com/golang/go/blob/fa18f224c378f5831210077944e5df718efb8df5/src/runtime/asm_arm64.s#L126)


- 1. 程序到达调度的时机;
- 2. 选取下一个Goroutine;
  - 全局：globrunqget()
    - 从全局里面拿不是每次只拿一个，而是拿多个，多余一个的放入本地运行G队列(```runqput(_p_ *p, gp *g, next bool)```)。
  - 本地P：runqget()
  - 从其他的P中偷：findrunnable()
    - sched.gcwaiting
    - local runq
    - global runq
    - netpoll
    - steal from other P
- 3. 切换Goroutine cpu执行选出的Goroutine.

抢占的分类

![fb2d3bf7-d5f6-4421-b191-b087cc0ddf5a](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/fb2d3bf7-d5f6-4421-b191-b087cc0ddf5a.jpg)

![539b9267-7355-49e1-95a5-5e8b1392c19e](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/539b9267-7355-49e1-95a5-5e8b1392c19e.jpg)

![9b452c91-7377-4707-8596-a2f2ff8704e5](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/9b452c91-7377-4707-8596-a2f2ff8704e5.jpg)


- 抢占又分为：函数抢占，和信号抢占
    - ![stack Preempt](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210114145818.png)
    - ![async(signal) Preempt](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/e274fe0e-275c-4072-aefc-da53922e36e7.jpg)
    - 

## archive

- 谈一谈GMP
我们在C++一般的运行一个线程就是调用系统函数创建一个系统线程。
1-1 N-1 N-M(如果说1-1是)
预先创建一堆线程池，然后用户代码里面接收请求，来一个请求handle一个。




## 附录

<div id="zxc-anchor-1"></div>

- [1] [我是大彬: Go调度器系列](https://mp.weixin.qq.com/s/epWDv9Nrmgg4ySFlk1gGKg)

- [2] [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit)

https://lessisbetter.site/2019/04/14/golang-scheduler-4-explore-source-code/


https://speakerdeck.com/kavya719/the-scheduler-saga?slide=103


https://www.cnblogs.com/CodeWithTxT/p/11370215.html#:~:text=%E4%B8%BA%E4%BA%86%E6%9B%B4%E5%90%88%E7%90%86%E7%9A%84%E5%88%A9%E7%94%A8,%E5%88%86%E5%9C%B0%E5%88%A9%E7%94%A8CPU%E3%80%82

