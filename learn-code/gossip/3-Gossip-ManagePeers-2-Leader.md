# 2 选举Org中的Leader Peer

## 1 入口

gossip\service\gossip_service.go

InitializeChannel()

感觉前面的调用过程很复杂，不知道怎样就调用到了这里，完成该peer的gossip channel 部分的工作

```go
// InitializeChannel allocates the state provider and should be invoked once per channel per execution
func (g *GossipService) InitializeChannel(channelID string, ordererSource *orderers.ConnectionSource, store *transientstore.Store, support Support) {
	
	...

	// Delivery service might be nil only if it was not able to get connected
	// to the ordering service
	if g.deliveryService[channelID] != nil {
		// 1.从配置文件中读取参数，判断参加选举的方式
        // - peer.gossip.useLeaderElection
		// - peer.gossip.orgLeader
        leaderElection := g.serviceConfig.UseLeaderElection
		isStaticOrgLeader := g.serviceConfig.OrgLeader

        // 禁止同时为true
		if leaderElection && isStaticOrgLeader {
			logger.Panic("Setting both orgLeader and useLeaderElection to true isn't supported, aborting execution")
		}

        // 2.如果是动态选举
		if leaderElection {
			// 为该peer在该通道内生成一个LeaderElectionComponent
			g.leaderElection[channelID] = g.newLeaderElectionComponent(channelID, g.onStatusChangeFactory(channelID,
				support.Committer), g.metrics.ElectionMetrics)
		} else if isStaticOrgLeader {
		// 3.如果是静态选举
			g.deliveryService[channelID].StartDeliverForChannel(channelID, support.Committer, func() {})
		} else {
			logger.Debug("This peer is not configured to connect to ordering service for blocks delivery, channel", channelID)
		}
	} else {
		logger.Warning("Delivery client is down won't be able to pull blocks for chain", channelID)
	}

}
```

## 2 newLeaderElectionComponent

1. newLeaderElectionComponent

   gossip\service\gossip_service.go

```go
func (g *GossipService) newLeaderElectionComponent(channelID string, callback func(bool),
	electionMetrics *gossipmetrics.ElectionMetrics) election.LeaderElectionService {
	
	return election.NewLeaderElectionService(adapter, string(PKIid), callback, config)
}
```

2. NewLeaderElectionService

   gossip\election\election.go

```go
// NewLeaderElectionService returns a new LeaderElectionService
func NewLeaderElectionService(adapter LeaderElectionAdapter, id string, callback leadershipCallback, config ElectionConfig) LeaderElectionService {
	
	le := &leaderElectionSvcImpl{
		...
	}

	if callback != nil {
		le.callback = callback
	}
	// 开始参加选举
	go le.start()
	return le
}
```

3. le.start()

   gossip\election\election.go

```go
func (le *leaderElectionSvcImpl) start() {
	le.stopWG.Add(2)
    // 1.处理接收到的信息
	go le.handleMessages()
    // 2.等待网络环境安静下来
	le.waitForMembershipStabilization(le.config.StartupGracePeriod)
	// 3.参加选举
    go le.run()
}
```

## 3 le.start()-le.handleMessages()

1. 判断接收到的是哪些消息

```go
func (le *leaderElectionSvcImpl) handleMessages() {
	
	defer le.stopWG.Done()
	msgChan := le.adapter.Accept()
	for {
		select {
        //1.如果从stopChan中读取到数据，则直接返回
		case <-le.stopChan:
			return
        //2.如果从msgChan中读到了数据
		case msg := <-msgChan:
            // 2.1如果不是检查是否存活的消息
			if !le.isAlive(msg.SenderID()) {
				le.logger.Debug(le.id, ": Got message from", msg.SenderID(), "but it is not in the view")
				break
			}
            // 2.2处理接收到的消息
			le.handleMessage(msg)
		}
	}
}
```

2. 处理具体的关于election 的消息

```go
func (le *leaderElectionSvcImpl) handleMessage(msg Msg) {
	
	le.Lock()
	defer le.Unlock()
	
    // 1.如果是竞选消息
	if msg.IsProposal() {
        // 把发送者的id假如到集合中
		le.proposals.Add(string(msg.SenderID()))
	} else if msg.IsDeclaration() {
	// 2.如果是其他peer宣称已经为leader的消息
        // 2.1将该节点的le状态调整为leader存在状态
        atomic.StoreInt32(&le.leaderExists, int32(1))
        // 2.2如果在sleeping状态则向interruptChan中放一个空的结构体
		if le.sleeping && len(le.interruptChan) == 0 {
			le.interruptChan <- struct{}{}
		}
        // 2.3如果发送者的id比自己的id小，且network中已经存在了leader，则该节点放弃竞选leader
		if bytes.Compare(msg.SenderID(), le.id) < 0 && le.IsLeader() {
			le.stopBeingLeader()
		}
	} else {
		// We shouldn't get here
		le.logger.Error("Got a message that's not a proposal and not a declaration")
	}
}
```

3. 放弃竞选leader

   下面的这些地方没有找到具体的地方在哪...

   调用le.callback(false)→gossipService-Impl.onStatusChangeFactory()回调方法，停止当前节点请求区块的Deliver服务实例broadcastClient客户端，并删除关联的区块提供者BlocksProvider结构键值对，设置其b.done标志位为1。

   这样，使得区块提供者在DeliverBlocks()方法中跳出消息处理循环，转换为普通节点并从其他节点接收区块。

```go
func (le *leaderElectionSvcImpl) stopBeingLeader() {
	le.logger.Info(le.id, "Stopped being a leader")
	atomic.StoreInt32(&le.isLeader, int32(0))
	le.callback(false)
}
```





## 4 le.start()-le.waitForMembershipStabilization

1. 在一段时间内，等待网络成员的关系不发生变化，即网络稳定时

```go
func (le *leaderElectionSvcImpl) waitForMembershipStabilization(timeLimit time.Duration) {
	endTime := time.Now().Add(timeLimit)
    // 1.通过discovery模块获取本地保存的当前节点数量viewSize
	viewSize := len(le.adapter.Peers())
	for !le.shouldStop() {
        // 等待一段时间
		time.Sleep(le.config.MembershipSampleInterval)
		// 拿到网络中新的成员
        newSize := len(le.adapter.Peers())
		// 如果成员结构不再发生变化
        if newSize == viewSize || time.Now().After(endTime) || le.isLeaderExists() {
			return
		}
		viewSize = newSize
	}
}
```

2. 获取网络中peer成员

   gossip\election\adapter.go

```go
func (ai *adapterImpl) Peers() []Peer {
	peers := ai.gossip.PeersOfChannel(ai.channel)

	var res []Peer
	for _, peer := range peers {
		if ai.gossip.IsInMyOrg(peer) {
			res = append(res, &peerImpl{peer})
		}
	}

	return res
}

```

3. 拿到channel中的成员

   gossip\gossip\gossip_impl.go

```go
// PeersOfChannel returns the NetworkMembers considered alive
// and also subscribed to the channel given
func (g *Node) PeersOfChannel(channel common.ChannelID) []discovery.NetworkMember {
	gc := g.chanState.getGossipChannelByChainID(channel)

	return gc.GetPeers()
}
```





## 5 le.start()-le.run()

```go
func (le *leaderElectionSvcImpl) run() {
	defer le.stopWG.Done()
	for !le.shouldStop() {
        // 1.判断当前组织内是否已经产生了Leader主节点
		if !le.isLeaderExists() {
			le.leaderElection()
		}
		// If we are yielding and some leader has been elected,
		// stop yielding
		if le.isLeaderExists() && le.isYielding() {
			le.stopYielding()
		}
		if le.shouldStop() {
			return
		}
		if le.IsLeader() {
			le.leader()
		} else {
			le.follower()
		}
	}
}
```

### 1 开始选举

```go
func (le *leaderElectionSvcImpl) leaderElection() {
	
	// If we're yielding to other peers, do not participate
	// in leader election
	if le.isYielding() {
		return
	}
	// Propose ourselves as a leader
	le.propose()
	// Collect other proposals
	le.waitForInterrupt(le.config.LeaderElectionDuration)
	// If someone declared itself as a leader, give up
	// on trying to become a leader too
	if le.isLeaderExists() {
		le.logger.Info(le.id, ": Some peer is already a leader")
		return
	}

	if le.isYielding() {
		le.logger.Debug(le.id, ": Aborting leader election because yielding")
		return
	}
	// Leader doesn't exist, let's see if there is a better candidate than us
	// for being a leader
    // 循环读取le.proposals集合中缓存的其他节点PKI-ID，按照字母顺序排序，比较当前Peer节点与其他节点的PKI-ID。
	for _, o := range le.proposals.ToArray() {
		id := o.(string)
		if bytes.Compare(peerID(id), le.id) < 0 {
			return
		}
	}
	// If we got here, there is no one that proposed being a leader
	// that's a better candidate than us.
    // 如果当前Peer节点PKI-ID最小，则调用le.beLeader()方法，声明当前Peer节点为Leader主节点，再设置isLeader标志位为1。
    // 注意，如果存在多个节点都声明为Leader主节点，即收到其他节点发送的declaration消息，则主动选择PKI-ID字母序最小的节点成为Leader主节点。
    // 如果发送消息的节点PKI-ID更靠前，则当前节点主动放弃竞争Leader主节点，并标识自身的isLeader标志位为0。
	le.beLeader()
	atomic.StoreInt32(&le.leaderExists, int32(1))
}
```

### 2 成为leader

最后，le.beLeader()方法调用le.callback(true)回调函数，实际上调用了election模块初始化时传入的onStatusChangeFactory()方法参数（service/gossip_service.go）。参数为true则意味着将当前节点转换为Leader主节点，并启动该节点的Deliver服务实例，代表组织负责从Orderer服务节点请求获取通道账本数据。然后，设置le.leaderExists标志位为1，以标识当前节点为Leader主节点。

```go
func (le *leaderElectionSvcImpl) beLeader() {
	le.logger.Info(le.id, ": Becoming a leader")
	atomic.StoreInt32(&le.isLeader, int32(1))
	le.callback(true)
}
```

### 3 选举之后---Leader主节点

```go
func (le *leaderElectionSvcImpl) leader() {
    // 1.构造LeadershipMsg消息，即声明当前节点是Leader主节点的declaration消息
	leaderDeclaration := le.adapter.CreateMessage(true)
    // 2.将该声明消息广播到组织内的其他节点
	le.adapter.Gossip(leaderDeclaration)
	le.adapter.ReportMetrics(true)
    // 3.让当前节点进入休眠状态
	le.waitForInterrupt(le.config.LeaderAliveThreshold / 2)
}
```

### 4 选举之后---Follower节点

```go
func (le *leaderElectionSvcImpl) follower() {
	// 1.清空消息集合，并设置当前节点的leaderExists标志位为0
	le.proposals.Clear()
	atomic.StoreInt32(&le.leaderExists, int32(0))
	le.adapter.ReportMetrics(false)
    // 2.阻塞等待
	select {
	case <-time.After(le.config.LeaderAliveThreshold):
	case <-le.stopChan:
	}
}
```



