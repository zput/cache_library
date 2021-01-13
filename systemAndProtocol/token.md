

what:
  - token是一个令牌，是一串字符串，当用户使用用户名/密码登录成功后，server返回一个token，用户后面的请求只需要带上token即可。

why:
  - 减轻服务器的压力，如果每次请求都需要用户名/密码，服务器需要频繁的与数据库打交道。使程序更加健壮。

how:
## 认证分类

![token](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210107193448.png)

### H2M

- 当密码泄漏也能防止伪装token获取信息。
  - 服务端只能当服务创造一个token，马上把token持久化到数据库中，来防止，即使入侵者根据对称加密密码完全构造了一个token，但是也能防止登录。

- 只有token泄漏，对称加密算法没泄漏。
  - 使用浏览器的指纹锁。（但是指纹锁也有可能泄漏吧）

- 自动刷新过期token。
  - 当7天过期，最后一小时自动生成新token，通过响应header返回给前端。

- 多端互踢。
  - 这个只能当重新登录的时候，把持久化中这个用户先前的token都删除掉。（第一条中就不满足,那些token就不能登录了。）

### M2M

![20210107194549](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210107194549.png)

OAuth2以及凭证技术JWT token。

##### OAuth2

what: 
  - OAuth 引入了一个授权层，用来分离两种不同的角色：**客户端**和**资源所有者**。......资源所有者同意以后，资源服务器可以向**客户端**颁发令牌。客户端通过令牌，去请求数据。
  - 通过授权服务器获取 access token 和 refresh token（refresh token 用于重新刷新access token），然后通过 access token 从资源服务器获取数据

how: 
	- 授权码（authorization-code）
	- 隐藏式（implicit）
	- 密码式（password）：
	- 客户端凭证（client credentials）


- refresh token 和 access token
  - access token被设计用来客户端和资源服务器之间交互，而refresh token是被设计用来客户端和授权服务器之间交互。
  - 需要access token的过期时间（TTL）应该尽量短。
  - refresh token被截获，系统依然是安全的
  	- 客户端拿着refresh token去获取access token时同时需要预先配置的 secure key，客户端和授权服务器之前始终存在安全的认证。


##### JWT

what:
  	- JSON Web Token
  	- JWT是一种包含令牌（self-contained token），或者叫值令牌（value token），我们以前使用关联到session上的hash值被叫做引用令牌（reference token）

why:
	- 设计了一种自包含令牌，令牌签发后**无需**从服务器存储中检查是否合法，通过解析令牌就能获取令牌的过期

how:
	- ![20210108125939](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210108125939.png)



## 一些概念

### session 和 cookie

- session
  - 1，session 在服务器端，cookie 在客户端（浏览器）
  - 2，session 默认被存在在服务器的一个文件里（不是内存）
  - 3，session 的运行依赖 session id，而 session id 是存在 cookie 中的，也就是说，如果浏览器禁用了 cookie ，同时 session 也会失效（但是可以通过其它方式实现，比如在 url 中传递 session_id）
  - 4，session 可以放在 文件、数据库、或内存中都可以。
  - 5，用户验证这种场合一般会用 session 因此，维持一个会话的核心就是客户端的唯一标识，即session id

- cookie 
  -甜点/甜心:重复登录的时候不想多次输入账号密码,放cookie自动帮你把用户名给填了，能够方便一下用户。给用户的一点甜头。


“如果禁用浏览器 cookie，如何实现用户追踪和认证？”
URL重写的技术来进行会话跟踪，即每次HTTP交互，URL后面都会被附加上一个诸如 sid=xxxxx(session_id) 这样的参数，服务端据此来识别用户。

### authentication AND authorization

认证是 authentication

授权则不同，授权是 authorization，指的是什么样的身份被允许访问某些资源

实现认证和授权的基础是需要一种媒介（credentials）来标记访问者的身份或权利


### HTTP Basic AUthentication、HAMC、OAuth2，以及凭证技术JWT token

令牌必须保密，泄漏令牌与泄漏密码的后果是一样的

单一的系统授权往往是伴随认证来完成的，但是在开放 API 的多系统结构下，授权可以由不同的系统来完成，例如 OAuth。授权技术是解决“我能做什么？”的问题

#### OAuth2(open authorization) 开放授权

本地环境可以使用对称加密算法，生产环境建议使用非对称加密算法

- 对称加密：
  - 加密和解密用的是同一个密码或者同一套逻辑的加密方式。
  - 两边都有密码，都能互相传递消息。
  - DES算法简介
	- DES算法为密码体制中的**对称密码**体制，又被称为美国数据加密标准。

- 非对称加密：这个密码也叫对称秘钥，其实这个对称和不对称指的就是加密和解密用的秘钥是不是同一个。(**RSA**)
  - 一边只有公钥，一边只有公私钥。
  - 这个只能公钥的一边传输信息给拥有公私钥的一边，换句话说公钥加密的信息，只能私钥才能解开。
  - 怎么互相传递消息呢？
    - 通过只有公钥的一端生成随机密码，发送给另一边，后面的交流都用这个对称加密密钥进行加密传输。

- 撤回token(人为token过期)
  – 高频，就不该用值token
    - 不用token那使用什么？
  – 低频，token 有效期不要太长，用黑名单机制拦了就好了，这个名单不会特别

## 附录

https://insights.thoughtworks.cn/api-2/?unapproved=58071&moderation-hash=d7f7e84bb91302883912edb8105a63da#comment-58071

