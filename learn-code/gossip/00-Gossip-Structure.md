# Gossip源码结构

1. fabric/gossip

   gossip/
   |-- api 	消息加密服务接口：MessageCryptoSevice SecurityAdvisor接口，peer调用
   |-- comm 	p2p通信模块
   |-- common 	公用函数，定义，结构体
   |-- discovery	节点发现模块，用于发现网络中的存活结点，供gossip调用
   |-- election	   选举模块，用于选举领导peer，供service调用
   |-- filter		转发消息过滤模块，用于判断一个信息是否应该发送给一个网络成员
   |-- gossip	定义了gossip接口，实现goosip服务
   |   |-- algo	算法，PullEngine对象，供pull调用
   |   |-- channel	GossipChannel对象，供channelState实例管理
   |   |-- msgstore	MessageStore对象，供channel调用
   |    '-- pull	Mediator对象，供channel调用
   |-- identity	节点身份管理模块，使用身份映射：PKI-ID与identity之间的映射，供service调用
   |-- metrics
   |-- privdata 	隐私数据的管理模块
   |-- protoext
   |-- service		gossip服务器，封装了gossip服务，状态，分发模块等，与核心代码衔接
   |-- state			状态消息处理模块，供service调用
   `-- util			   公用工具文件夹，提供工具函数

2. internal/peer/gossip/

   internal/peer/gossip/

   |-- msc.go		MeassageCryptoService 消息加密服务实现模块

   `-- sa.go 		  SecurityAdvisor接口实现模块

3. vendor\github.com\hyperledger\fabric-protos-go\gossip\message.pb.go

   消息结构定义模块

---

# Gossip 消息模块启动流程

1. 创建与初始化Gossip服务器实例---initGossipService()

   管理具体的Gossip服务模块与组件以提供Gossip服务功能

2. 初始化通道的Gossip服务模块

   1. Peer节点加入应用通道时请求Endorser节点调用CSCC系统链码，创建本地节点上关联通道的链结构对象，用于管理通道配置、账本等资源。
   2. 初始化该通道上的Gossip服务模块，包括Gossip-Channel通道对象、隐私数据处理句柄privateHandler、state模块、deliverService服务模块等。
   3. 如果配置了动态选举Leader主节点的模式，则创建election选举模块参与竞争Leader主节点。
   4. 如果启用了静态配置模式且当前节点是Leader主节点，则启动该通道上的deliver-Service服务模块，从Orderer节点请求获取该通道上的账本区块，并转发给组织内的其他Peer节点。

---



---





