# 2 选举Org中的Leader Peer

## 1 入口

gossip\service\gossip_service.go

InitializeChannel()

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

## 2 g.newLeaderElectionComponent

newLeaderElectionComponent

gossip\service\gossip_service.go

```go
func (g *GossipService) newLeaderElectionComponent(channelID string, callback func(bool),
	electionMetrics *gossipmetrics.ElectionMetrics) election.LeaderElectionService {
	
	return election.NewLeaderElectionService(adapter, string(PKIid), callback, config)
}
```

### 1.NewLeaderElectionService

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

### 2.le.start()

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

## 3 le.handleMessages()

判断接收到的是哪些消息

gossip\election\election.go

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

### 1.le.handleMessage(msg Msg)

处理具体的关于election 的消息

gossip\election\election.go

```go
func (le *leaderElectionSvcImpl) handleMessage(msg Msg) {
	msgType := "proposal"
    // 首先检查接收消息的类型，如果该消息是declaration消息，即声明消息发送节点是Leader主节点
	if msg.IsDeclaration() {
		msgType = "declaration"
	}
	le.logger.Debug(le.id, ":", msg.SenderID(), "sent us", msgType)
	le.Lock()
	defer le.Unlock()
	
    // 1.如果是竞选消息
	if msg.IsProposal() {
        // 将该消息发送节点的PKIID添加到proposals消息集合中并等待处理
		le.proposals.Add(string(msg.SenderID()))
	} else if msg.IsDeclaration() { // 测试是否声明当前节点是Leader主节点
	// 2.如果是其他peer宣称已经为leader的消息
        // 2.1将该节点的le状态调整为leader存在状态，表明已经存在Leader主节点
        atomic.StoreInt32(&le.leaderExists, int32(1))
        // 2.2如果election选举模块处于休眠状态中（le.sleeping标志位为true），并且interruptChan通道中不存在中断消息。
		if le.sleeping && len(le.interruptChan) == 0 {
			// 发送空结构struct{}{}到le.interruptChan通道，作为信号唤醒election选举模块，并重新参与竞争Leader主节点
            le.interruptChan <- struct{}{}
		}
        // 2.3如果发送者的id比自己的id小，并且当前节点声明为Leader主节点
		if bytes.Compare(msg.SenderID(), le.id) < 0 && le.IsLeader() {
            // 消息发送者自动成为Leader主节点，当前节点放弃竞争Leader主节点。
			le.stopBeingLeader()
		}
	} else {
		// We shouldn't get here
		le.logger.Error("Got a message that's not a proposal and not a declaration")
	}
}
```

### 2.le.stopBeingLeader

放弃竞选leader

```go
func (le *leaderElectionSvcImpl) stopBeingLeader() {
	le.logger.Info(le.id, "Stopped being a leader")
	atomic.StoreInt32(&le.isLeader, int32(0))
	le.callback(false)
}
```

#### 1.callback的定义

1. 回调函数是在初始化选举模块时定义的(第2部分已经传进来了)，其中回调函数为g.onStatusChangeFactory

   gossip\service\gossip_service.go

```go
g.leaderElection[channelID] = g.newLeaderElectionComponent(channelID, g.onStatusChangeFactory(channelID,
				support.Committer), g.metrics.ElectionMetrics)
```

2. g.onStatusChangeFactory

   gossip\service\gossip_service.go

   停止当前节点请求区块的Deliver服务实例broadcastClient客户端，并删除关联的区块提供者BlocksProvider结构键值对，设置其b.done标志位为1。

   这样，使得区块提供者在DeliverBlocks()方法中跳出消息处理循环，转换为普通节点并从其他节点接收区块。

```go
func (g *GossipService) onStatusChangeFactory(channelID string, committer blocksprovider.LedgerInfo) func(bool) {
	return func(isLeader bool) {
		if isLeader {
			yield := func() {
				g.lock.RLock()
				le := g.leaderElection[channelID]
				g.lock.RUnlock()
				le.Yield()
			}
			logger.Info("Elected as a leader, starting delivery service for channel", channelID)
			if err := g.deliveryService[channelID].StartDeliverForChannel(channelID, committer, yield); err != nil {
				logger.Errorf("Delivery service is not able to start blocks delivery for chain, due to %+v", err)
			}
		} else {
			logger.Info("Renounced leadership, stopping delivery service for channel", channelID)
			if err := g.deliveryService[channelID].StopDeliverForChannel(channelID); err != nil {
				logger.Errorf("Delivery service is not able to stop blocks delivery for chain, due to %+v", err)
			}
		}
	}
}
```

## 4 le.waitForMembershipStabilization

1. 在一段时间内，等待网络成员的关系不发生变化，即网络稳定时

```go
func (le *leaderElectionSvcImpl) waitForMembershipStabilization(timeLimit time.Duration) {
	endTime := time.Now().Add(timeLimit)
    // 1.通过discovery模块获取本地保存的当前节点数量viewSize
	viewSize := len(le.adapter.Peers())
	for !le.shouldStop() {
        // 2.等待一段时间
		time.Sleep(le.config.MembershipSampleInterval)
		// 3.拿到网络中新的成员
        newSize := len(le.adapter.Peers())
		// 4.通过成员的数量判断网络结构是否发生了变化，如果成员结构不再发生变化
        if newSize == viewSize || time.Now().After(endTime) ||le.isLeaderExists() {
			return
		}
		viewSize = newSize
	}
}
```

### 1.le.adapter.Peers

获取网络中peer成员

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

### 2.ai.gossip.PeersOfChannel

拿到channel中的成员

gossip\gossip\gossip_impl.go

```go
// PeersOfChannel returns the NetworkMembers considered alive
// and also subscribed to the channel given
func (g *Node) PeersOfChannel(channel common.ChannelID) []discovery.NetworkMember {
	gc := g.chanState.getGossipChannelByChainID(channel)

	return gc.GetPeers()
}
```

## 5 le.run()

```go
func (le *leaderElectionSvcImpl) run() {
	defer le.stopWG.Done()
	for !le.shouldStop() {
        // 1.判断当前组织内是否已经产生了Leader主节点
		if !le.isLeaderExists() {
			// 如果不存在leader，则开始选举
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
        
        // 3.选举之后-如果成为了leader
		if le.IsLeader() {
			le.leader()
		} else {
        // 4.选举之后-如果没有成为leader
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

最后，le.beLeader()方法调用le.callback(true)回调函数，

- 实际上调用了election模块初始化时传入的onStatusChangeFactory()方法参数（service/gossip_service.go）。
- 参数为true则意味着将当前节点转换为Leader主节点，并启动该节点的Deliver服务实例，代表组织负责从Orderer服务节点请求获取通道账本数据。
- 然后，设置le.leaderExists标志位为1，以标识当前节点为Leader主节点。

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



