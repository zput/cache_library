- kubernets中volume有那些?
- 为什么不要只使用docker一样的volume就可以了?
- 有了PV为什么需要pvc?
- 创建PV的种类有几类?

- kubernets的volume类型有哪些?
  - 本机
  - 网络存储
  - projected volume: secret/configMap
  - pvc 与pv体系
  
  
- 为什么不要只使用docker一样的volume就可以了?
  - 方便复用和共享, 
    - 销毁
    - 扩展
    - 共享-->多个pod共享.
    
- 有了PV为什么需要pvc?   
  - 面向接口,而不是面向实现.
  - PVC只用声明自己(size/accessMode),而不需要关心实现.
  
- 创建PV的种类有几类?  
  - static volume provisioning 
  - dynamic volume provisioning
  
static volume provisioning 
 使用阿里云文件存储(NAS)
 Cluster Admin
 1. 通过阿里云文件存储控制台,创建NAS文件系统和添加挂载点. 
 2. 创建PV对象, 将NAS文件系统大小,挂载点, 以及PV的access mode, reclaim policy等信息添加到PV对象中. 
 
 User: 
 1. 创建PVC对象, 声明存储需求. 
 2. 创建应用pod并通过在.spec.volumes中通过PVC声明volume,通过.spec.containers.volumeMounts声明container挂在使用该volume.



[aliyun](https://edu.aliyun.com/lesson_1651_18381?spm=5176.254948.1334973.27.2c12cad2XQ8GmG#_18381)

