
what: 是golang内部自己实现的调度器，由’‘G’’,“M”,“P"用来调度goruntine，被称为"GMP模型”。

why:


- 单进程时代不需要调度器
  - 1.单一的执行流程，计算机只能一个任务一个任务处理。
  - 2.进程阻塞所带来的CPU时间浪费。

- 多进程/线程时代有了调度器需求
  - 1.解决了阻塞的问题
  - 2.CPU有很大的一部分都被浪费在进程调度
  - 设计会变得更加复杂，要考虑很多同步竞争等问题，如锁、竞争冲突等。

- 协程(用户线程)来提高CPU利用率(浪费在进程调度的CPU)
  - 协程和线程 
    - 1-1: 操作系统调度;
    - N-1: 用户程序调度; N个协程绑定1个线程，优点就是协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速;
    - N-M: 用户程序调度; 

>这个也就说明了```N-M```的基础，用户线程的各自栈空间其实就是放在公共的堆（heap）上。
>>每个系统线程都有一个唯一的m0, g0与之对应，想想为什么？
>>>每个线程有自己的栈空间(而这个g0就在这个栈上)。但是是与其他的线程公用的~~代码段~~，**数据段，堆空间**,所以当创建其他的goroutine的时候，把它的协裎栈在堆上，所以它可以被其他的M调用。


how:



## FQA

### 说一说GMP的底层实现。

- ![GMP大全](https://raw.githubusercontent.com/zput/myPicLib/master/GO/goroutine%E8%B0%83%E5%BA%A6.png)

#### 从程序加载到main函数执行的过程。

- 两条主线 + GMP经典图：
  - 那个经典的GMP在这一阶段实际是怎么表示的，(g0,m0,allp[0]),
  - 代码加载到内存中的code segment, 从```入口-->runtime.main-->main.main```.
  - 从全局变量的角度，全局变量
    - ```(g0,m0,sched)```;
    - ```(allg,allm,allp)```;
    - ```sched(midle, pidle, runq(全局queue))```.
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

















