# 思考问题的方法

1. 目前解决该问题的技术路线？
   1. Docker（目前在用
   2. Docker Swarm
   3. Kubernetes 
2. 各种技术路线的深度分析
   1. 技术难点
   2. 优势
   3. 缺陷
   4. 每个技术路线最终可以实现的效果
3. 目前最好的技术路线的缺陷是什么？
4. 自己使用什么样的技术路线？
5. 该技术路线解决目前的痛点问题
   1. 该技术路线的缺点？

# 一、大宗商品BaaS需要解决问题？

需要解决的问题：

1. 综合数据平台等各个参与方， 因为安全或者是隐私的愿意， 可能会不允许在他们的服务器上部署区块链节点
2. 帮助每一位使用该平台的用户，根据自己的需要，在自己的组织内实现节点的快速部署与管理，同时可以进行一些区块链服务调用，如channel的创建和加入，链码的安装与调用。
3. 可以让用户专注于快速创建、管理和监控区块链网络，不需要过多的关注底层的硬件资源。



技术难点：

1. Fabric Java SDK和该区块链BaaS平台的逻辑关系（调用关系）是什么？Fabric Java sdk 部署在kubernetes 的容器中嘛？每个组织只部署一个嘛？
   1. 我个人感觉对于每一个组织都应该部署一个Java sdk把？不然只有一个Java sdk来接受用户对区块链的请求是不是有点中心化的思想在里面了？
   2. 如果每个组织都部署一个fabric sdk 那么 他们和kubernetes sdk的逻辑调用关系是什么啊？
   3. 因为每个组织都有自己的区块链节点，因此他们对于自己的区块链进行操作时，是不是需要有自己的fabric sdk来提供服务，如果是这样的话，fabric sdk 是不是运行在容器中比较好？
4. 怎么把kubernetes对用户的权限控制 和 fabric 对用户的权限控制结合起来？？
   1. kuberntes可以通过权限管理技术，限制一个用户的权限，即创建peer节点、orderer节点的权限。
   2. fabric 对用户的权限管理是怎么实现的？即一个用户在区块链中可以干什么事情，不可以干什么事情，fabric是怎么限定用户的权限的？
   3. 怎么把kubernetes对用户的权限管理 和 fabric 的权限管理结合在一起？
3. 在fabric-samples中crypto-config文件夹充当的是什么角色？
   1. CA在fabric中扮演的是什么角色？他和证书与密钥是什么关系？
      1. CA：区块链节点类型之一，英文名称Certificate Authority，数字证书颁发机构，负责组织内部成员的register和enroll等，为该组织的区块链用户生成和颁发数字证书。
   2. 怎么使用fabric ca？
   3. 区块链中所有的节点都去ca完成注册并拿到证书吗？
   4. fabric ca 和 fabric msp的联系是什么？
4. 如何实现用户的身份管理功能？
   1. 如何提供保存/管理用户身份与公钥、地址之间的映射关系？同时这些映射关系可被颁发机构生成数字证书，满足法律效力？
   2. 用户的身份管理 和 私钥管理 之间的关系是什么？
   3. 当系统启动起来时，Fabric Java SDK 的证书从哪里获取？
   4. 用户的证书 和 kubernetes中实现用户的身份认证的证书有什么关系？
   5. 阿里是怎么利用TEE来管理用户的密钥的？
5. 怎么把动态的向区块链网络中添加一个组织/节点？
   1. 动态的加入一个组织的过程中发生了什么事情？生成证书 区块配置文件 区块链网络配置文件
   2. 添加组织/节点的工作肯定是由kubernetes来完成，那么怎么吧eyfn中实现的步骤（生成配置文件），与kubernetes 所管理的容器 结合起来呢？
   3. 问题的关键还是在于新增节点的证书等配置文件应该怎么配置？
   4. 怎么动态的添加一个peer？
      1. peer加入到网络中的时候，怎么和ca结合在一起？？
   5. kubernetes管理各个peer orderer后，sdk怎么找到他们？
6. kubernetes的用户（管理员）和fabric的用户（组织内的用户）他们之间的联系和区别是什么？
7. 分析请求/资源是如何实现调度的，怎么为fabric sdk的负载均衡模块服务？怎么实现负载均衡？负载均衡要和kubernetes组件结合在一起嘛？
8. 怎么解决云上的数据隐私问题？
   1. 本来区块链就是一个去中心化的网络，放在云上带来的中心话的问题应该怎么解决？



---

# 二、BaaS 系统架构

![image-20201013224631047](http://qhczuqqky.hn-bkt.clouddn.com/img/image-20201013224631047.png)

## 1 系统架构

1. 应用层 / 业务层：向应用开发者 和 平台管理员提供不同的操作管理能力
2. 核心层：完成系统的 资源编排、系统监控、数据分析 和 权限管理 区块链网络服务 等重要功能
3. 底层驱动：访问 和 管理 多种物理资源

## 2 BaaS 各组件的功能

a) 对用户提供

1. 权限/证书管理：不同的用户具备不同的 数据/请求 权限（权限管理属于fabric还是kubernetes部分？权限管理管理的是什么权限？管理区块链网络的权限），通过带认证的接口，用户根据自己的权限可以访问区块链网络，安装智能合约并进行调用。
2. 节点管理：管理区块链网络节点，一键式快速创建和部署生产级区块链环境，简化区块链的部署流程和应用配置。
3. 通道管理：可以创建&删除通道，管理通道的生命周期（事实上是区块链的生命周期的管理工作）
4. 智能合约管理：管理智能合约的生命周期，完成一键式的智能合约的部署与调用。
5. 区块链服务：fabric Java sdk 的安装与调用，使用fabric Java sdk提供区块链服务（fabric sdk应该可以调用kubernetes sdk的把？Fabric Java SDK 和 Kubernetes Java SDK 是同级关系嘛？这两个SDK部署在哪里？也是部署在容器中嘛？）
6. 监控服务：监控区块链网络的具体工作情况
   1. 区块链监控服务：可以通过 Fabric explorer 等类似的区块链监控工具，也可以通过fabric sdk完成实现具体的功能。
   2. 区块链的使用情况包括：区块数目情况，交易数目情况



b) 对管理员提供

1. 用户管理：（这里的用户管理管理的是什么用户？这里的用户实际上指的就是可以创建节点的某一组织内的用户）
2. 资源管理：容器节点的编排功能（计算资源、存储资源、网络资源这些是否支持动态扩容？？），使用kubernetes Java sdk 提供容器编排的服务
3. 节点监控服务：监控区块链网络中各个节点是否在正常的工作。
   1. 信息采集组件 - 部署：通过kubernetes部署在sdk业务层、oderer共识节点、peer节点账本存储层。
   2. 信息采集组件 - 采集：可以完成采集节点的系统信息（cpu memory network disk）、节点的使用状态（节点访问量、访问耗时 和 节点的健康状态）、业务使用情况（业务访问量、成功率 和 耗时分布）。



3) 平台提供

1. 高性能：区块链节点具备处理高并发的交易的能力（需要对fabric源码进行修改工作，将源码编译成镜像，把镜像交由kubernetes进行管理工作）

2. 可扩展硬件资源：可以跨区域的扩展BaaS平台的硬件资源（Kubernetes可以扩展硬件资源吗？）

3. 资源/请求调度：调度不均匀的用户请求/资源请求（这个资源的请求调度应该通过什么模块完成？）

4. 安全保障：

   1. 节点工作的安全性
   2. 节点中用户数据的安全性

   

---

# 三、技术路线

![image-20201013224723975](http://qhczuqqky.hn-bkt.clouddn.com/img/image-20201013224723975.png)

## 1 Kubernetes

动态的添加/管理一些容器，可以使用kubernetes java client项目来完成容器的编排工作，通过kubernetes java client客户端可以完成对kubenetes原生资源对象（pod、node、namespace、servcie、deployment）和自定义资源对象（如：cluster）的增删改查或事件监听（watch）。



Kubernetes Java client 通过运用什么技术完成容器的编排工作？

Kubernetes Java client 调用RESP API完成对分布式容器集群的编排工作。其中REST API 是 Kubernetes 的基础架构，kubernetes组件之间的所有操作和通信，以及外部用户命令都是 API Server 处理的 REST API 调用。因此，Kubernetes 平台中的所有资源被视为 API 对象，并且在API中都有对应的定义项。



Kubernetes 如何实现对用户的身份进行认证？

[Kubernetes身份认证策略](https://jimmysong.io/kubernetes-handbook/guide/authentication.html)。

Kubernetes 使用客户端证书、bearer token、身份验证代理或者 HTTP 基本身份验证等身份认证插件来对 API 请求进行身份验证。当有 HTTP 请求发送到 API server 时，插件会尝试将以下属性关联到请求上：

- 用户名：标识最终用户的字符串。常用值可能是 `kube-admin` 或 `jane@example.com`。
- UID：标识最终用户的字符串，比用户名更加一致且唯一。
- 组：一组将用户和常规用户组相关联的字符串。
- 额外字段：包含其他有用认证信息的字符串列表的映射。



## 2 Fabric 

[fabric java sdk 使用的简单介绍](https://uzshare.com/view/822302)

使用Fabric Java SDK完成和区块链节点的交互工作。可以通过该Fabric Java SDK完成对channel和chaincode的生命周期的管理工作，同时可也以完成执行用户的链码，查询区块和查询交易等信息。

channel

1. 创建channel
2. 加入channel

chaincode

1. 打包链码
2. 安装链码
3. 查询是否安装成功
4. 在某组织内同意/审批链码的安装
5. 查询链码是否经过同意/审批
6. 提交链码
7. 测试执行链码



## 3 数据流图

![image-20201104192948002](http://haoimg.hifool.cn//img/image-20201104192948002.png)

1. 用户注册、登陆
   1. kubernetes sdk 负责
   2. 完成对用户赋予权限的过程
2. 区块链节点管理：用户添加主机、添加组织、添加peer节点、添加orderer节点
   1. kubernetes sdk 负责
   2. 这个平台应该只有一个，他负责为每个用户提供创建区块链节点的功能，因为每个用户的权限不同，所以并不是每个用户都有创建区块链节点的功能。
3. 区块链服务初始化：创建channel、peer加入channel、管理chaincode的生命周期
   1. fabric sdk 负责
   2. 根据每个用户所拥有的权限，他们根据每个组织的fabric sdk所提供区块链服务，进行一些操作
4. 区块链服务调用：合约上链后，执行只能合约，并发起交易
   1. fabric sdk 负责
   2. 负责提供与区块链交互的服务
5. 区块链网络状态监控：
   1. fabric explorer等区块链浏览器提供的区块链监控功能
   2. 可以查看区块链网络中现存的区块状态、交易状态
6. 区块链网络节点的监控：
   1. kubernetes相应的监控组件 负责
   2. 监控节点的运行状态（cpu 内存 存储 带宽）



# 四、BaaS 系统的评测指标-技术难点

## 1 性能指标

区块链节点和应用（智能合约）对交易的处理速度是否达到要求，即对交易的入链/查询的实时性的要求

可以做的点：

1. 根据大宗商品区块链应用场景的需要，可以对peer节点进行改造
   1. 将peer节点的 模拟阶段 和 验证阶段 进行并行化处理
2. 改进fabric的peer/orderer的源码，加快交易的处理速度
3. 在调度层面做好负载均衡

## 2 可扩展性

物理资源编排：是否具有大规模场景下的应用部署和管理的能力，是否可以跨区域的对云上硬件资源进行扩展。

## 3 资源调度

负载均衡：对于非均匀的资源请求可以实现智能调度，合理的分配系统的资源。

## 4 安全性

是否可以避免外部攻击和内部攻击，以及是否可以保障系统的容错性。

1. 怎么在区块链网络中 保证密钥的安全存储和管理？目前的解决方案是什么？
   1. 阿里：使用TEE（如Intel SGX）技术，保护区块链的私钥安全；
   2. 腾讯：密钥保险箱使用用户的信息对密钥加密，并将加密后的数据分割存储在多个不同的节点上。正常的业务中并不会访问用户的密钥保险箱，当用户密钥丢失后，可以通过对用户信息认证步骤，完成找回密钥。
2. 支持国密算法（SM234）；
3. 提供隐私数据的加解密和授权设施：
   1. 用户可以调用加密SDK进行自己隐私数据的保护，同时通过解密授权服务，完成对相关人员的授权访问，并在访问结束后可以动态的回收这些授权。
4. 提供用户身份管理设施：
   1. 提供用户身份与公钥、地址之间的映射关系的保存设施，同时可由颁发机构生成数字证书，满足法律效力的要求。
5. 隐私保护：
   1. 怎么保护用户的隐私信息：多条通道多条链之间进行隔离，信息进行加密，智能合约进行请求控制等方式。
6. 监控产品管理运维过程中的高风险操作，引入风控验证&日志审计功能

## 5 可感知性

数据分析：深度感知用户的数据行为，可以评估用户链上账户的应用状态，主动提示用户。

## 6 底层资源普适性

底层可以支持多种混合的计算架构，可以导入多种异构的物理资源。
