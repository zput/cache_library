
## TCP转移图

![20200927182536](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20200927182536.png)






- TCP与UDP的区别，
- TCP三次握手四次分手的作用是什么
  - 初始化资源，同步信息sequence号
    - 如果两次握手会发生什么？
      - 那么本来已经无效的数据，重新到达对方，导致错误，在这里如果客户端，第一个SYN信号在网络中，延迟，客户端重新发起了连接，然后发送数据，然后关闭，这个时候第一次SYN信号才到达，那么两个握手的话，服务端会认为客户端重新发起了连接导致，服务端白白消耗资源。
      - 防止已失效的**连接请求**报文段，又传送到了服务端。
  - 回收资源,通知对方，我不发送数据了
  
- 流量控制与拥塞控制
  - 如果区分两者，有一个拥塞避免的步骤，那么拥塞控制就是网络堵塞。
  - 流量控制就是滑动窗口。

- 流量控制
  - what：接收方对发送方的流量或者速度进行控制。
  - why：因为相对来说发送方总是希望快速的把数据发完的，而接收方的接收缓存是有限度的，这是他们的矛盾
  - how：利用滑动窗口来进行控制。
    - 1.连接建立的时候接收方通告发送方自己接收窗口的大小。
    - 2.利用三个指针来实现窗口的状态
      - P1  P2 P3
      - P3-p1等于窗口的大小。
      - 当发送方收到接收方的零窗口，发送方会启动持续计时器，来定时发送给接收方，看它的窗口是否变大了。
- 拥塞控制
  - what：对网络的某一个资源需求超过了网络所能提供的负荷，导致拥塞的情况发生。拥塞控制就是控制这种情况的发生。
  - why：如果不进行控制，网络上的阻塞会导致后面积压越多的阻塞。导致网络风暴。
  - how：慢开始 拥塞避免 快重传 快恢复
    - 前面两个是TCP tahoa的
    - 后面是升级的TCP reno，解决到丢包是，系统误以为是网络拥塞，导致网络传输效率下降。

 TODO: picture

- 超时重传时间：
  TODO：picture

- 黏包问题
  - what：当应用层发送方发送两个信息，但是接收方接收到的信息是两个信息的连接，他们连接到了一起
  - why：tcp是面向字节流的，数据无边界概念。
    - 发送发把两个信息组成一起发送。
    - 接收方把两个信息同时发送到应用层缓冲区。
  - how：发送数据之前先发送信息的大小，接收方接收的时候，先接收它的长度信息，然后根据长度接收数据


### 计时器

![clock_tcp](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/clock_tcp.png)



## 附录

### 一些问题

![tcp](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/tcp.png)


























