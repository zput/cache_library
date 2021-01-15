
## FQA


- 谈一谈GMP
我们在C++一般的运行一个线程就是调用系统函数创建一个系统线程。
1-1 N-1 N-M(如果说1-1是)
预先创建一堆线程池，然后用户代码里面接收请求，来一个请求handle一个。


每个系统线程都有一个唯一的m0, g0与之对应，想想为什么？
每个线程有自己的栈空间(而这个g0就在这个栈上)。但是是与其他的线程公用的**代码段，数据段，堆空间**,所以当创建其他的goroutine的时候，把它的协裎栈在堆上，所以它可以被其他的M调用。

### 说一说GMP的底层实现。

#### 从程序加载到main函数执行的过程。
    - 两条主线 plus GMP经典图：
      - 那个经典的GMP在这一阶段实际是怎么表示的，(g0,m0,allp[0]),
      - 代码加载到内存中的code segment, 从```入口--》runtime.main--》main.main```.
      - 从全局变量的角度，全局变量```(g0,m0,sched) --- (allg,allm,allp)```.
        - ```sched(midle, pidle, runq(全局queue))```
    - 进行```mcall()--》schedule()```.
    ![20210114101912](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210114101912.png)


> 从上面就引出了创建新G，与调度问题：```newproc(),与mcall()--->schedule()```

#### 新的goroutine创建的过程newproc()。

  - 创造一个新**goroutine**[```go myfunc()```]:
    - 它的最终结果一定是，就像是goexist()函数调用这个```myfunc函数一样```。
    - 创造一个新的goroutine前会在全局里面找看是否有```gidle的goroutine```.
    - 最后会```runput```到队列中（可能是本地，可能是全局）。

    ![20210114142644](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210114142644.png)

   ![c05b9cda-5f30-4f90-9cd5-379a28005a86](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/c05b9cda-5f30-4f90-9cd5-379a28005a86.jpg)

   ![newproc的过程](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/748394d1-2bc0-4cbc-b8c4-ef019908a993.jpg)


#### 调度的过程schedule()。


![fb2d3bf7-d5f6-4421-b191-b087cc0ddf5a](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/fb2d3bf7-d5f6-4421-b191-b087cc0ddf5a.jpg)

![539b9267-7355-49e1-95a5-5e8b1392c19e](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/539b9267-7355-49e1-95a5-5e8b1392c19e.jpg)

![9b452c91-7377-4707-8596-a2f2ff8704e5](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/9b452c91-7377-4707-8596-a2f2ff8704e5.jpg)



- 抢占又分为：函数抢占，和信号抢占
    - ![stack Preempt](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210114145818.png)
    - ![async(signal) Preempt](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/e274fe0e-275c-4072-aefc-da53922e36e7.jpg)
    - 










