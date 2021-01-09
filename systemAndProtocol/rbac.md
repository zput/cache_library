
## rbac

what:
  	- role-based access control(基于角色的访问控制)
	- 类型：
	  - 第一种：最基本的rbac,就是单单的【user<--->role<--->resource】。
	    - 其中role可以继承，但是资源不能够继承。
	  - 第二种：RBAC with resource roles: both users and resources can have roles (or groups) at the same time.
	    - 这个是资源也有继承体系。比如【张三】有【develop资源】【write权限】；   <--->   且【develop.data资源】继承至【develop资源】;那么【张三】有【develop.data资源】写权限。
	  - 第二种：RBAC with domains/tenants: users can have different role sets for different domains/tenants. 

why:
	- ACL (Access Control List); 
	  - 用户和资源量一大，ACL就会变得异常繁琐。想象一下，每次新增一个用户，都要把他需要的权限重新设置一遍是多么地痛苦。

how:
	- 如何设计一个rbac?
		- RBAC模型：用户-角色-权限。所以最基本的我们应该具备用户、角色、权限这三个内容。
		- 如何将三者关联起来?
		  - 角色作为枢纽，关联用户、权限。所以创建一个角色，并为这个角色赋予相应权限，最后将角色赋予用户。
	- casbin
	"r": "request_definition",
	"p": "policy_definition",
	"g": "role_definition",
	"e": "policy_effect",
	"m": "matchers",
![rpme](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210109111146.png)
![20210109111441](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210109111441.png)
### 用户组的概念

用户组和用户角色之间的区别。用户角色是平级的，用户组是存在多级管理关系的。
![20210109105319](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210109105319.png)


### 实践

[一个小例子](https://github.com/zput/zput_rbac)

## 附录

https://zhuanlan.zhihu.com/p/148353743
https://casbin.org/docs/zh-CN/get-started
http://www.woshipm.com/pd/1150093.html
https://wen.woshipm.com/question/detail/88fues.html






























