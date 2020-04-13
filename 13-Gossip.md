# Gossip 科普：什么是Gossip协议？

---

# Gossip 协议在 Fabric中的作用

Gossip数据传播协议在fabric中主要有三个功能：

1. 通过Gossip持续不断的识别和发送，利用它完成**管理节点**的作用，如发现通道成员，或者也可以检测离线节点。
2. 利用Gossip协议，向通道中的所有的节点**传播账本数据**。所有没有和当前通道的数据同步的节点，能够识别出丢失的区块，并将正确的数据复制过来，使自己能够保持完整的区块信息。
3. 通过点对点的数据传输的方式，能够以**最快**的方式使节点连接到网络中，并同步账本数据。

## Peer节点和Gossip

Peer节点**通过gossip协议，完成（传播账本&通道数据）的作用**。

Gossip协议是**持续的**，通道中的每一个Peer节点都会持续不断的从多个节点接收到当前一致的账本数据。

- 因为网络故障的一些原因（延迟，网络分区等），一些peer节点会丢失一些区块，因此peer节点会持续不断的从其他peer节点上利用gossip协议同步自己丢失的区块，完善自己的账本。
- Peer节点并不会接受来自非目标节点的与其无关的Gossip消息。

**Gossip消息是带有签名**的，因此拜占庭成员发送的伪造消息很容易被识别到。



**Peer 节点基于 gossip 的数据广播操作接收通道中其他的节点的信息**，然后将这些信息随机发送给通道上的一些其他节点，随机发送的节点数量是一个可配置的常量。

Peer 节点可以用“拉”的方式获取信息而不用一直等待。这是一个重复的过程，以使通道中的成员、账本和状态信息同步并保持最新。

在分发新区块的时候，通道中 **主** 节点从排序服务拉取数据然后分发给它所在组织的节点。

---

# Gossip在Fabric中的应用

## 主节点的选举机制

主节点的选举机制：可以使**orderer节点**和**开始分发区块信息的节点**（这个分发区块信息的节点我们称之为主节点）之间**互相连接**。



主节点的选举机制**可以有效的利用排序服务所能提供的带宽**：

- orderer节点出块之后，并不需要把这个块挨个分发给每一个peer节点了，只需要把块发给主节点，然后主节点在把块分发给其他peer节点。
- 同理，利用了gossip协议，主节点也不需要一一的发送给所有的节点---节点之间互相点对点的同步消息即可。



主节点的选举模型：

1. 静态模式：系统管理员静态的配置一个节点为peer组织的主节点。
2. 动态模式：peer组织中，自己选举一个peer节点为主节点。



### 静态主节点的选举：

静态主节点的选举可以配置peer组织中的一个或多个peer为主节点。注：太多的话会浪费orderer排序服务提供的带宽。

关于gossip配置信息的部分我们可以在fabric-samples/config/core.yaml中的peer-gossip部分窥见一斑：

``` yaml
peer:
	...
	gossip:
		...
		# NOTE: orgLeader and useLeaderElection parameters are mutual exclusive.
        # Setting both to true would result in the termination of the peer
        # since this is undefined state. If the peers are configured with
        # useLeaderElection=false, make sure there is at least 1 peer in the
        # organization that its orgLeader is set to true.

        # Defines whenever peer will initialize dynamic algorithm for
        # "leader" selection, where leader is the peer to establish
        # connection with ordering service and use delivery protocol
        # to pull ledger blocks from ordering service. It is recommended to
        # use leader election for large networks of peers.
        useLeaderElection: false # 不参加选举
        # Statically defines peer to be an organization "leader",
        # where this means that current peer will maintain connection
        # with ordering service and disseminate block across peers in
        # its own organization
        orgLeader: true # 是主节点
        ...
		
```

不要将 `CORE_PEER_GOSSIP_USELEADERELECTION` 和 `CORE_PEER_GOSSIP_ORGLEADER` 都设置为 `true`，这将会导致错误。



值得注意的是，上面的两个参数我们**可以在peer容器的环境变量中进行覆盖和修改**，我们从fabric-samples/first-network/base/peer-base.yaml中可以看到peer容器的系统环境变量覆盖了上面的配置信息。

```yaml
services:
  peer-base:
    image: hyperledger/fabric-peer:$IMAGE_TAG
    environment:
      ...
      - CORE_PEER_GOSSIP_USELEADERELECTION=false
      - CORE_PEER_GOSSIP_ORGLEADER=true
      ...
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
```

参数解释：

```yaml
export CORE_PEER_GOSSIP_USELEADERELECTION=false # 不参加选举
export CORE_PEER_GOSSIP_ORGLEADER=true # 直接成为peer组织的主节点
```



注：如果我们不希望某一个peer成为peer组织中的主节点，成为旁观者，可以对这两个环境变量进行修改为false：

```yaml
services:
  peer-base:
    image: hyperledger/fabric-peer:$IMAGE_TAG
    environment:
      ...
      - CORE_PEER_GOSSIP_USELEADERELECTION=false
      - CORE_PEER_GOSSIP_ORGLEADER=false
```

### 动态主节点选举机制：

动态主节点选举使组织中的节点可以 **选举 一个节点来连接排序服务**，并拉取新区块。

- 这个主节点由每个组织单独选举。

动态选举出的主节点通过向其他节点发送 **心跳** 信息来证明自己处于存活状态。

- 如果一个或者更多的节点在一个段时间内没有收到 **心跳** 信息，它们就会选举出一个新的主节点。
  配置控制主节点 **心跳** 信息的发送频率：

  ```yaml
  peer:
      # Gossip related configuration
      gossip:
          election:
              leaderAliveThreshold: 10s
  ```



**在网络比较差有多个网络分区存在的情况下**，组织中会存在多个主节点以保证组织中节点的正常工作。

- 在网络恢复正常之后，其中一个主节点会**放弃领导权**。
- 在一个没有网络分区的稳定状态下，会只有 **唯一** 一个活动的主节点和排序服务相连



关于动态主节点的配置信息与静态主节点的配置正好相反，可以看上面的配置信息的过程。

---

## 锚节点

**gossip 利用锚节点来保证不同组织间的互相通信**，从而完善各个组织间的成员信息。

当Peer 节点提交了一个包含锚节点更新的配置区块时，**Peer 节点会连接到锚节点并获取它所知道的所有节点信息**。

- 因为 gossip 的通信是固定的，而且 Peer 节点总会被告知它们不知道的节点，所以可以建立起一个通道上成员的视图。

由于组织间的通信依赖于 gossip，所以**在通道配置中必须至少有一个锚节点**，锚节点就可以获取通道中所有节点的信息。

- 为了系统的可用性和冗余性，**每个组织都提供自己的一些锚节点**。
- 注意，**锚节点不一定和主节点是同一个节点**。

### 锚节点源码配置：

例如，假设我们在一个通道有三个组织 A、B 和 C，组织 C 定义了锚节点 peer0.orgC。

1. 当 peer1.orgA 连接到 peer0.orgC 时，peer1.orgA 将会告诉 peer0.orgC 有关 peer0.orgA 的信息。
2. 稍后等 peer1.orgB 连接到 peer0.orgC 时，peer0.orgC 会告诉peer1.orgB 关于 peer0.orgA 的信息。
3. 因此，组织 A 和 B 可以不通过 peer0.orgC 从而间接的交换成员信息。



Anchor Peer 的配置信息我们可以从 fabric-samples/first-network/configtx.yaml 中看到：

```yaml
Organizations:
    - &OrdererOrg
        ...
        OrdererEndpoints:
            - orderer.example.com:7050
    - &Org1
        ...
        # leave this flag set to true.
        AnchorPeers:
            # AnchorPeers defines the location of peers which can be used
            # for cross org gossip communication.  Note, this value is only
            # encoded in the genesis block in the Application section context
            - Host: peer0.org1.example.com
              Port: 7051
    - &Org2
        ...
        AnchorPeers:
            - Host: peer0.org2.example.com
              Port: 9051
```



---

### 外部和内部端点（endpoint）

为了让 gossip 高效地工作，**Peer 节点需要包**含其**所在组织**以及**其他组织**的端点信息。

#### 内部端点

当 Peer 节点启动的时候，它会使用 `core.yaml` 文件中的 `peer.gossip.bootstrap` 来宣传自己并交换成员信息，同时建立所属组织中可用节点的视图。

```yaml
peer:
    # Gossip related configuration
    gossip:
        # Bootstrap set to initialize gossip with.
        # This is a list of other peers that this peer reaches out to at startup.
        # Important: The endpoints here have to be endpoints of peers in the same
        # organization, because the peer would refuse connecting to these endpoints
        # unless they are in the same organization as the peer.
        bootstrap: 127.0.0.1:7051
```



`core.yaml` 文件中的 `peer.gossip.bootstrap` 属性用于**在 一个组织内部 启动 gossip**。

如果你要使用 gossip，通常要**为组织中的所有节点配置一组启动节点**（使用空格隔开的节点列表）。

**内部端点通常是由 Peer 节点自动计算**的，或者在 `core.yaml` 中的 `core.peer.address` 指明。

```yaml
peer:
    ...
    # When used as peer config, this represents the endpoint to other peers
    # in the same organization. For peers in other organization, see
    # core.gossip.externalEndpoint for more info.
    # When used as CLI config, this means the peer's endpoint to interact with
    address: 0.0.0.0:7051
```

如果你要覆盖该值，可以为peer容器设置环境变量 `CORE_PEER_GOSSIP_ENDPOINT`。



#### 外部端点

启动信息也同样需要建立 **跨组织** 的通信。**初始的跨组织启动信息通过“锚节点”设置提供。**

如果想**让其他组织知道你所在组织中的其他节点**，你需要设置 `core.yaml` 文件中的 `peer.gossip.externalendpoint`。

- **如果没有设置，节点的端点信息就不会广播到其他组织的 Peer 节点**。

```yaml
peer:
	...
    # Gossip related configuration
    gossip:
    	...
        # Bootstrap set to initialize gossip with.
        # This is a list of other peers that this peer reaches out to at startup.
        # Important: The endpoints here have to be endpoints of peers in the same
        # organization, because the peer would refuse connecting to these endpoints
        # unless they are in the same organization as the peer.
        bootstrap: 127.0.0.1:7051
        # Overrides the endpoint that the peer publishes to peers
        # in its organization. For peers in foreign organizations
        # see 'externalEndpoint'
        endpoint:
        # This is an endpoint that is published to peers outside of the organization.
        # If this isn't set, the peer will not be known to other organizations.
        externalEndpoint:
```



这些属性的设置在peer容器中的配置如下：first-network/base/docker-compose-base.yaml

```yaml
services:
  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org1.example.com
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:7051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org1.example.com:7052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7052
      # a list of peer endpoints within the peer's org
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:8051
      # the peer endpoint, as known outside the org
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    ports:
      - 7051:7051
```

## Gossip 消息

**Gossip消息是带有签名**的，因此拜占庭成员发送的**伪造消息很容易被识别到**。

在线的节点通过持续**广播“存活”消息来表明其处于可用状态****，每一条消息都包含了“公钥基础设施（PKI）” ID 和发送者的签名。

- 节点通过收集这些存活的消息来维护通道成员。
- 如果没有节点收到某个节点的存活信息，这个“死亡”的节点会被从通道成员关系中剔除。
- 因为“存活”的消息是经过签名的，恶意节点无法假冒其他节点，因为他们没有根 CA 签发的密钥。

除了自动转发接收到的消息之外，状态协调进程还会**在每个通道上的 Peer 节点之间同步世界状态**。

- 每个 Peer 节点都持续从通道中的其他节点拉取区块，来修复他们缺失的状态。
- 因为基于 gossip 的数据分发不需要固定的连接，所以该过程可以可靠地提供共享账本的一致性和完整性，包括对节点崩溃的容忍。

因为**通道是隔离的**，所以一个**通道中的节点无法和其他通道通信或者共享信息**。

- 尽管节点可以加入多个通道，但是分区消息传递通过基于 Peer 节点所在通道的应用消息的路由策略，来防止区块被分发到其他通道的 Peer 节点。

### Gossip数据传输过程中的一些安全新因素：

1. **通过 Peer 节点 TLS 层来处理点对点消息的安全性，不需要使用签名**。
   Peer 节点通过 CA 签发的证书来授权。尽管没有使用 TLS 证书，但在 gossip 层使用了经过授权的 Peer 节点证书。账本区块经过排序服务签名，然后被分发到通道上的主节点。
2. **通过 Peer 节点的成员服务提供者来管理授权**。
   当 Peer 节点第一次连接到通道时，TLS 会话将与成员身份绑定。这就利用网络和通道中成员的身份来验证了与 Peer 节点相连的节点的身份。