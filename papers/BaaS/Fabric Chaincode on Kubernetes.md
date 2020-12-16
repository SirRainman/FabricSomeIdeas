# Kubernetes上使用外部链码

在kubernetes上运行链码容器时遇到的缺点：

1. the dependency of the Docker daemon in the peer leaded to some configurations that might not be acceptable in production environments, 
   1. such as the use of the Docker Socket of the worker nodes in the peer to deploy the chaincode containers, hence making these containers out of Kubernetes domain, 
   2. or the use of Docker-in-Docker (DinD) solutions, which required these containers to be privileged.

# 1 为什么链码容器的编排是BaaS系统的难点之一？

## 1.1 现有的Fabric链码容器的启动方式

chaincode 目前是以 Docker 容器的方式运行在 peer 容器所在的宿主机上，peer 容器需要调用 Docker 引擎的接口来构建和创建 chaincode 容器

Fabric启动链码容器的方式：

1. 将docker.sock文件volume进容器里
2. 在容器里调用容器外的docker daemon来创建新的容器。
3. peer节点接收到实例化链码的请求，共识通过后，连接CORE_VM_ENDPOINT参数配置的远程Docker，发送部署链码容器消息
4. peer节点创建并启动链码容器（链码容器的启动是peer通过设置CORE_VM_ENDPOINT=unix:///var/run/docker.sock，然后在peer容器里调用容器外的docker daemon来创建新的容器）



这种方法的弊端：

1. peer通过使用docker socket方式进行启动链码容器，这种方法启动的链码容器是独立于kubernetes系统外的，脱离了现有的容器编排系统。
   1. 虽然它仍在 Flannel 网络上，但却无法获得 peer节点的IP地址。这是因为创建该容器的 Docker 引擎使用宿主机默认的 DNS 解析来 peer 的域名，所以无法找到。
2. 安全性和可移植性。
   1. peer节点的权力太大
   2. 可移植性并不好



## 1.2 如何解决Fabric原生启动链码容器带来的弊端？

Kubernetes中启动链码容器：

1. K8S中运行一个Docker Pod，然后将链码在这个Docker环境中启动进程
2. 使用docker in docker的方式启动链码容器，即链码容器运行在一个容器之中

这种方法的弊端：

1. 虽然这种方法使得链码容器运行在了kubernetes中，可以由kubernetes进行管理，但是这种方法实质上还是使得链码容器运行在了另外容器之中，kubernetes系统是没有办法直接管理到具体的链码容器的。
2. 链码容器的存活，链码增多，dind负担也将增加，崩溃的可能性增大。
3. 此外，当链码容器因为某些原因崩溃后，kubernetes是感知不到的。
4. 虽然可以通过持久化存储实现dind重启后Pod重启，然而fabric虽然通过心跳机制感知到了chaincode容器的崩溃，但是k8s不能知道dind里嵌套的后一个docker的崩溃，所以fabric **无法重启chaincode**，k8s也**无法重启Pod**。



## 1.3 如何更好的解决这个问题？

chaincode运行在专有的运行环境之中：

1. fabric的链码实例化源码改动，让其支持k8s启动；
   1. 需要fabric源码作出改动，修改peer节点创建链码容器的过程。
2. 从dind入手，拦截创建链码容器的命令，然后通过k8s api创建pod，这其实相当于做了一个**代理**。
3. 实现外部链码构建器，baas系统直接启动外部链码容器，peer节点直接和kubernetes系统内的链码容器进行沟通，不会给peer容器暴露docker socket。



# 2 构建外部链码容器

通过构建外部链码容器能够解决的问题：

1. 外部链码容器的build/launch 和peer解耦合
2. 更安全，peer节点只需要做自己应该做的事情
3. 不用依赖Kubernetes CRI，避免了dind带来的问题
4. 当peer发现chaincode容器崩溃时，可以及时的处理





























































