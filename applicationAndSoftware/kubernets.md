

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [安装](#安装)
- [CRD](#crd)
  - [crd's exercise](#crds-exercise)
- [一个目标：容器操作](#一个目标容器操作httpscloudtencentcomdeveloperarticle1415035)
  - [实践](#实践httpszhuanlanzhihucomp363978095)
- [四个网络关系:container,pod,service,internet](#四个网络关系containerpodserviceinternet)
  - [容器和容器之间的网络](#容器和容器之间的网络)
  - [Pod与Pod之间的网络](#pod与pod之间的网络)
    - [同一个Node中的Pod之间的一次通信](#同一个node中的pod之间的一次通信)
    - [不同Node中的Pod之间通信](#不同node中的pod之间通信)
  - [Pod与Service之间的网络](#pod与service之间的网络)
    - [netfilter](#netfilter)
    - [iptables](#iptables)
    - [IPVS](#ipvs)
    - [Pod到Service的一个包的流转](#pod到service的一个包的流转)
    - [Service到Pod的一个包的流转](#service到pod的一个包的流转)
  - [Internet与Service之间的网络](#internet与service之间的网络)
    - [k8s流量到Internet](#k8s流量到internet)
    - [Internet到k8s](#internet到k8s)
  - [从其他观点出发:](#从其他观点出发)
    - [七层负载均衡](#七层负载均衡)
      - [service](#service)
      - [Headless Services](#headless-serviceshttpskubernetesiodocsconceptsservices-networkingserviceheadless-services)
- [vxlan协议格式](#vxlan协议格式)
- [CRD](#crd-1)
    - [需要更近一步探寻的](#需要更近一步探寻的)

<!-- /code_chunk_output -->


## 安装

https://github.com/easzlab/kubeasz


## CRD

https://blog.csdn.net/aixiaoyang168/article/details/81875907

https://github.com/kubernetes-sigs/kubebuilder

https://book.kubebuilder.io/quick-start.html


https://kuboard.cn/install/install-kubernetes.html#%E6%A3%80%E6%9F%A5%E7%BD%91%E7%BB%9C

https://www.bookstack.cn/read/source-code-reading-notes/kubernetes-kubelet_garbage_collect.md


### crd's exercise

```sh
[root@iZf8z14idfp0rwhiicngwqZ k8s-cronjob]# make undeploy
/root/code/go_tmp/awesomeProject/example/k8s-cronjob/bin/kustomize build config/default | kubectl delete -f -
namespace "k8s-cronjob-system" deleted
customresourcedefinition.apiextensions.k8s.io "cronjobs.batch.tutorial.kubebuilder.io" deleted
serviceaccount "k8s-cronjob-controller-manager" deleted
role.rbac.authorization.k8s.io "k8s-cronjob-leader-election-role" deleted
clusterrole.rbac.authorization.k8s.io "k8s-cronjob-manager-role" deleted
clusterrole.rbac.authorization.k8s.io "k8s-cronjob-metrics-reader" deleted
clusterrole.rbac.authorization.k8s.io "k8s-cronjob-proxy-role" deleted
rolebinding.rbac.authorization.k8s.io "k8s-cronjob-leader-election-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "k8s-cronjob-manager-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "k8s-cronjob-proxy-rolebinding" deleted
configmap "k8s-cronjob-manager-config" deleted
service "k8s-cronjob-controller-manager-metrics-service" deleted
deployment.apps "k8s-cronjob-controller-manager" deleted
```



https://feisky.gitbooks.io/kubernetes/content/introduction/cluster.html

## [一个目标：容器操作](https://cloud.tencent.com/developer/article/1415035)

- ![20210412200447](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210412200447.png)
  - kubectl：客户端命令行工具，作为整个系统的操作入口。
  - kube-apiserver：以 REST API 服务形式提供接口，作为整个系统的控制入口。
  - kube-controller-manager：执行整个系统的后台任务，包括节点状态状况、Pod 个数、Pods 和Service 的关联等。
  - kube-scheduler：负责节点资源管理，接收来自 kube-apiserver 创建 Pods 任务，并分配到某个节点。
  - etcd：负责节点间的服务发现和配置共享。
  - kube-proxy：运行在每个计算节点上，负责 Pod 网络代理。定时从 etcd 获取到 service 信息来做相应的策略。
  - kubelet：运行在每个计算节点上，作为 agent，接收分配该节点的 Pods 任务及管理容器，周期性获取容器状态，反馈给 kube-apiserver。
  - DNS：一个可选的 DNS 服务，用于为每个 Service 对象创建 DNS 记录，这样所有的 Pod 就可以通过 DNS 访问服务了。


https://mp.weixin.qq.com/s?__biz=MzI0MDQ4MTM5NQ==&mid=2247505452&idx=2&sn=0f50c8087960b13c0b7762a6611c3a96&chksm=e918b330de6f3a26e46d7bb8e96b8da7b0daf1c2761cac76117118406ee8a82aa8e2d8fbbbd1&scene=21#wechat_redirect


### [实践](https://zhuanlan.zhihu.com/p/363978095)

1. kubernetes 启动后，无论是 master 节点 亦或者 node 节点，都会将自身的信息存储到 etcd 数据库中
2. 创建 nginx 服务，首先会将安装请求发送到 master 节点上的 apiServer 组件中
3. apiServer 组件会调用 scheduler 组件来决定应该将该服务安装到哪个 node 节点上。这个时候就需要用到 etcd 数据库了，scheduler会从 etcd 中读取各个 node 节点的信息，然后按照一定的算法进行选择，并将结果告知给 apiServer
4. apiServer 调用 controllerManager 去调度 node 节点，并安装 nginx 服务
5. node 节点上的 kubelet 组件接收到指令后，会通知docker，然后由 docker 来启动一个 nginx 的pod
6. pod 是 kubernetes 中的最小操作单元，容器都是跑在 pod 中

7. 以上步骤完成后，nginx 服务便运行起来了，如果需要访问 nginx，就需要通过 kube-proxy 来对 pod 产生访问的代理，这样外部用户就能访问到这个 nginx 服务



- 首先我们创建一个 deployment
- 然后创建一个 service 来让外界能够访问到我们 nginx 服务


```sh
kubectl -n develop create deployment nginx --image=nginx:1.14-alpine
kubectl -n develop get deploy|grep nginx
kubectl -n develop expose deploy nginx --port=80 --target-port=80 --type=NodePort
kubectl -n develop get svc|grep nginx
```


```sh
[root@master ~]# kubectl create deployment nginx --image=nginx:1.14-alpine 
deployment.apps/nginx created

[root@master ~]# kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           31s


[root@master ~]# kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort 
service/nginx exposed

[root@master ~]# kubectl get svc 
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
nginx        NodePort    10.110.224.214   <none>        80:31771/TCP   5s
```

## 四个网络关系:container,pod,service,internet

https://network.51cto.com/art/201908/601109.htm

https://cloud.tencent.com/developer/article/1415035

K8S 网络设计原则如下：

每个 Pod 都拥有一个独立 IP 地址，Pod 内所有容器共享该 IP 地址。
集群内所有 Pod 都在一个直接连通的扁平网络中，可通过 IP 直接访问。
即：所有容器之间无需 NAT 就可以直接互相访问;所有 Node 和所有容器之间无需 NAT 就可以直接互相访问;容器自己看到的 IP 跟其他容器看到的一样。

### 容器和容器之间的网络

### Pod与Pod之间的网络

#### 同一个Node中的Pod之间的一次通信

#### 不同Node中的Pod之间通信


### Pod与Service之间的网络

#### netfilter
#### iptables
#### IPVS
#### Pod到Service的一个包的流转
#### Service到Pod的一个包的流转

### Internet与Service之间的网络
#### k8s流量到Internet
#### Internet到k8s







### 从其他观点出发:

#### 七层负载均衡

各层负载均衡 | 含义 | kubernets里面表现形式
-------|----|----------------
二层负载均衡 | 基于MAC地址的二层负载均衡 |\
三层负载均衡 | 基于IP地址的负载均衡 | \
四层负载均衡 | 基于IP+端口的负载均衡 | k8s 原生的 kube-proxy 方式
七层负载均衡 | 基于URL等应用层信息的负载均衡:在这种情况下，更类似于一个代理服务器。它和前端的客户端以及后端的服务器会分别建立 TCP 连接。所以从这个技术原理上来看，七层负载均衡明显的对负载均衡设备的要求更高，处理七层的能力也必然会低于四层模式的部署方式 |  Ingress:这是一个基于七层的方案



- k8s关于服务的暴露主要是通过NodePort方式，通过绑定主机的某个端口，然后进行Pod的请求转发和负载均衡，但这种方式有下面的缺陷：
  - Service可能有很多个，如果每个都绑定一个Node主机端口的话，主机需要开放外围的端口进行服务调用，管理混乱。
  - 无法应用很多公司要求的防火墙规则。

- 理想的方式是通过一个外部的负载均衡器，绑定固定的端口，比如80；然后根据域名或者服务名向后面的Service IP转发。
  - Kubernetes给出的方案就是Ingress

pod (po), service (svc), replicationcontroller (rc), deployment (deploy), replicaset (rs)

##### service

> Service可以看作是一组提供相同服务的Pod对外的访问接口。借助Service，应用可以方便地实现服务发现和负载均衡。 service默认只支持4层负载均衡能力，没有7层功能。（可以通过Ingress实现）

- service的类型：
  - ClusterIP：默认值，k8s系统给service自动**分配的虚拟IP**，只能在**集群内部**访问。
  - NodePort：将Service通过指定的**Node上的端口**暴露给外部，访问任意一个 NodeIP:nodePort都将路由到ClusterIP。
  - LoadBalancer：**在NodePort的基础上**，借助```cloud provider```创建一个外部的负载均衡器，并将请求转发到 :NodePort，此模式只能在云服务器上使用。
  - ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 spec.externlName 设定）。

- Service 是由**kube-proxy**组件，加上**iptables**来共同实现的.
  - kube-proxy通过iptables处理Service的过程，需要在宿主机上设置相当多的iptables规则，如果宿主机有大量的Pod，不断刷新iptables规则，会消耗大量的CPU资源。
  - IPVS模式的service，可以使K8s集群支持更多量级的Pod。











https://blog.csdn.net/Aimee_c/article/details/106964337




##### [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)

- 无头服务(No ClusterIP)：此前，每一个service都会有一个service名称，解析的结果都是cluster ip，请求由cluster ip DNAT到后端的pod上。
  - 因此cluster ip的解析结果只会有一个ip。但是，现在可以去掉中间这层cluster ip(既不会动态分配也不会手动指定)，而后的解析将会解析到pod ip上，而pod的ip是取决于集群大小。
    - 这类的service就是无头service。直达后端pod。
    - ![20210413170236](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210413170236.png)







## vxlan协议格式

 VXLAN 全称是 Virtual eXtensible Local Area Network，虚拟可扩展的局域网。它是一种 overlay 技术，通过三层的网络来搭建虚拟的二层网络，其帧格式：


![20210413150934](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210413150934.png)


















## CRD




https://git.forchange.cn/linweiqi/sandbox-debug-bridge/-/blob/master/main.go

https://git.forchange.cn/runtime/test/sandbox_test/-/issues/1#5gatewayoperator


https://stackoverflow.com/questions/47848258/what-is-the-difference-between-a-kubernetes-controller-and-a-kubernetes-operator

It's only an Operator if it's got: controller pattern + API extension + single-app focus.

An Operator is a controller. It's just that when the controller adds new k8s objects to store configuration for a component like prometheus or memcached, they use the term Operator. **A controller** normally **just watches** and **reacts** to native k8s objects.



- The list of controller in the Control-plane:
  - Deployment
  - ReplicaSet
  - StatefulSet
  - DaemonSet
  - etc

- From the Google Search, I found out that there are K8s Operators such as
  - etcd Operator
  - Prometheus Operator
  - kong Operators





#### 需要更近一步探寻的


```go
	sb := &corev2alpha1.Sandbox{}
  why ? new(corev2alpha1.Sandbox)



```






















