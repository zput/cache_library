

- what:
  - Go-Micro是框架，不是服务，但是使用它来编写微服务;
    - 主要是解决微服务（注册/发现-通信-可靠性）
  - Micro是基于Go-Micro编写，面向Go-Micro服务治理与生态的工具集，它包含很多服务、工具.
    - Go-Micro与Micro，两个项目的关联.

--- 
--- 

- why:
  - 当面向多服务的时候，需要关心单体服务不需要关心的方面：
    - 服务的注册与发现
    - 服务之间的通信
    - 服务的可靠性
    - 部署

服务的注册与发现：支持Etcd、MSDN、Consul、ZK、Eureka...
服务间通信：HTTP、TCP、UDP、MQ（同步），HTTP、MQ（异步）
服务可靠性：TTL/Interval、基于Wrapper的限流、熔断等特性

--- 
---

- how:
  - 微服务提供服务能力的三种方式：RPC、OpenAPI、HTTP
    - Go-Micro方案：Srv、Web、API
      - SRV：内部RPC服务
      - API：对外API服务
        - API: 业务核心逻辑内聚在SRV，API则负责统一业务入口，并将不同SRV的能力聚合。
          - ![20210206151326](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206151326.png)
      - Web：对外HTTP服务
        - 基于Go-Micro开发Web应用
          - 支持服务发现与心跳检测
          - 支持自定义Handler
          - 支持静态文件
            - ![20210206150615](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206150615.png)
      - **最上面的Gateway：是Micro（通过go-micro框架编码写成的工具集）**
        - micro api （网关）
          - HTTP网关
          - 暴露内部RPC、Web、API服务
          - 基于Go-Micro编写，本身也是Go-Micro应用
            - ![20210206151529](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206151529.png)
          - ![20210206152505](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206152505.png)
            - 来了一个请求怎么知道它的type是:event还是api?
            - 如果知道type == api
              - ![20210206153440](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206153440.png)

 ---  

  - 核心组件
    - ![20210206145528](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206145528.png)
      - Service
      - 中间的client/server
        - Client：发送RPC请求与广播消息
        - Server：接收RPC请求与消费消息
      - 底部组件
        - Broker：异步通信组件
        - Codec：数据编码组件
        - Registry：服务注册组件
        - Selector：客户端均衡器
        - Transport：同步通信组件
    - ![20210206151013](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206151013.png)






1. 框架API
2. ENV 环境变量
3. CLI 命令行参数
同时声明：1<2<3


https://github.com/stack-labs/starter-kit
https://raw.githubusercontent.com/stack-labs/learning-videos/master/docs/micro-api/README.md


### Handler处理器

| - | 类型 | 说明
----|----|----
1 | rpc | 通过RPC向go-micro应用转送请求，只接收GET和POST请求，GET转发`RawQuery`，POST转发`Body`
2 | api | 与rpc差不多，但是会把完整的http头封装向下传送，不限制请求方法
3 | http或proxy | 以反向代理的方式使用**API**，相当于把普通的web应用部署在**API**之后，让外界像调api接口一样调用web服务
4 | web | 与http差不多，但是支持websocket
5 | event | 代理event事件服务类型的请求
6 | meta* | 默认值，元数据，通过在代码中的`Endpoint`配置选择使用上述中的某一个处理器，默认RPC

- `rpc`或`api`模式同样可以使用`Endpoint`定义路由。

```go
api.Endpoint{
	// 接口方法，一定要在proto接口中存在，不能是类的自有方法
	Name: "Example.Call",
	// http请求路由，支持POSIX风格
	Path: []string{"/example"},
	// 支持的方法类型
	Method: []string{"POST", "GET"},
	// 该接口使用的API转发模式
	Handler: rpc.Handler,
}
```

### Router路由

<img src="https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210206154251.png" width="75%">

- router过程
	- endpoint
		- 自定义路由
	- resolver
		- 路径规则

### Resolver请求映射

> 这里的Resolver类型都是`micro`中默认的`go-micro/api/resolver/micro`

`rpc`类型需要服务名称`go.micro.api.greeter` + 方法名`Greeter.Hello`

请求路径    |    后台服务    |    接口方法
----    |    ----    |    ----
/foo/bar    |    go.micro.api.foo    |    Foo.Bar
/foo/bar/baz    |    go.micro.api.foo    |    Bar.Baz
/foo/bar/baz/cat    |    go.micro.api.foo.bar    |    Baz.Cat
/v1/foo/bar    |    go.micro.api.v1.foo    |    Foo.Bar
/v1/foo/bar/baz    |    go.micro.api.v1.foo    |    Bar.Baz
/v2/foo/bar    |    go.micro.api.v2.foo    |    Foo.Bar
/v2/foo/bar/baz    |    go.micro.api.v2.foo    |    Bar.Baz


`proxy`类型只需要服务名称，用于服务发现，将http请求转发到对应的服务

请求路径    |    服务    |    后台服务路径
---    |    ---    |    ---
/foo    |   go.micro.api.foo	|   /foo
/foo/bar	|   go.micro.api.foo	|   /foo/bar
/greeter    |    go.micro.api.greeter    |    /greeter
/greeter/:name    |    go.micro.api.greeter    |    /greeter/:name



