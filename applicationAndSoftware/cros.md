## cors(cross-origin request Sharing)

### what:什么是跨域？

- 我有一个域名a.com和一个域名b.com
- 我在a.com上有一个接口a.com/api，会返回一些数据
- 我想在b.com域名下的一个页面上访问a.com/api得到数据
- 浏览器阻止了我

### why: 为什么不让我跨域？

- 因为在web交互的环境中，只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。

- 跨站请求伪造（CSRF: Cross-site request forgery）
  - 1. 支付宝的转账操作是一个post请求，大概是https://alipay.com/api/withdraw/?to_user=XXX&amout=1000.
  - 2. 我写了一段ajax(Asynchronous Javascript And XML)的post请求代码，请求连接是上面的url.
  - 3. 然后我把这段代码嵌入我的网站a.com你不久前登陆过支付宝.
  - 4. 浏览器里保存了alipay.com域名的cookie.
  - 5. 我让你访问a.com，打开页面，于是在你不知情的情况下发出了post请求，你的钱就被转到我的账号里了.

### how: 解决跨域问题:

- 发请求前设置一下document.domain的值.
  - 父子域名
- JSONP---json with padding.
  - 在非父子域关系的情况下，如developer.mozilla.org和production.mozilla.org（或者a.com和b.com），就是被浏览器当作两个不同的域名的，一般就会使用JSONP了.
- 跨域资源共享（CORS）.
  - 简单请求---GET和POST。
  - 预检请求---PUT和DELETE等，或者请求时添加了CORS安全的header之外的header（如自定义的）。
  - 带cookie的请求---要求响应头里```Access-Control-Allow-Credentials```为true，且```Access-Control-Allow-Origin```不能是通配符，防止后端犯错。



[进一步详细博客](https://zput.github.io/2018/05/24/tool/cros/)










