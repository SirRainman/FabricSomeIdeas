# 3 Peer节点接收Block消息

## 1 入口-Gossip服务启动

gossip\gossip\gossip_impl.go

1. 初始化Gossip服务

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {
	...
    
	go g.start()
	go g.connect2BootstrapPeers()

	return g
}
```

2. 启动gossip服务

```go
func (g *Node) start() {
	go g.syncDiscovery()
	go g.handlePresumedDead()
	
    // 接收到消息
	incMsgs := g.comm.Accept(msgSelector)
	go g.acceptMessages(incMsgs)
}
```

## 2 acceptMessages

```go
func (g *Node) acceptMessages(incMsgs <-chan protoext.ReceivedMessage) {
	defer g.logger.Debug("Exiting")
	defer g.stopSignal.Done()
	for {
		select {
		case <-g.toDieChan:
			return
		case msg := <-incMsgs:
			g.handleMessage(msg)
		}
	}
}
```

## 3 g.handleMessage

```go
func (g *Node) handleMessage(m protoext.ReceivedMessage) {
	...

	msg := m.GetGossipMessage()

	if !g.validateMsg(m) {
		g.logger.Warning("Message", msg, "isn't valid")
		return
	}

	if protoext.IsChannelRestricted(msg.GossipMessage) {
		if gc := g.chanState.lookupChannelForMsg(m); gc == nil {
			...
		} else {
			...
			gc.HandleMessage(m)
		}
		return
	}
    
	...
}
```

## 4 gc.HandleMessage

gossip\gossip\channel\channel.go

```go
// HandleMessage processes a message sent by a remote peer
func (gc *gossipChannel) HandleMessage(msg protoext.ReceivedMessage) {
	//前面是异常处理部分，在此省略
    ...
    
	if protoext.IsStateInfoPullRequestMsg(m.GossipMessage) {
		msg.Respond(gc.createStateInfoSnapshot(orgID))
		return
	}

	if protoext.IsStateInfoSnapshot(m.GossipMessage) {
		gc.handleStateInfSnapshot(m.GossipMessage, msg.GetConnectionInfo().ID)
		return
	}

    // 如果接收到的是data或者是stateinfo
	if protoext.IsDataMsg(m.GossipMessage) || protoext.IsStateInfoMsg(m.GossipMessage) {
		added := false
		if protoext.IsDataMsg(m.GossipMessage) {
			if m.GetDataMsg().Payload == nil {
				return
			}
			// Would this block go into the message store if it was verified?
			if !gc.blockMsgStore.CheckValid(msg.GetGossipMessage()) {
				return
			}
            // 通过MCS消息加密服务模块验证该区块结构的合法性，以及区块元数据中的签名是否满足指定的区块验证策略BlockValidation。
			if !gc.verifyBlock(m.GossipMessage, msg.GetConnectionInfo().ID) {
				return
			}
            // 将DataMsg消息添加到blockMsgStore与blocksPuller消息存储对象上
			gc.Lock()
			added = gc.blockMsgStore.Add(msg.GetGossipMessage())
			if added {
                // blocksPuller对象周期性地发送Pull类数据消息，以拉取并更新本地节点账本上缺失的DataMsg消息
				gc.blocksPuller.Add(msg.GetGossipMessage())
			}
			gc.Unlock()
		} else { // StateInfoMsg verification should be handled in a layer above
			//  since we don't have access to the id mapper here
			added = gc.stateInfoMsgStore.Add(msg.GetGossipMessage())
		}

		if added {
            // 将该DataMsg消息封装为emittedGossip-Message类型消息，添加到emitter模块缓冲区中，交由emitter模块打包并发送到组织内的其他Peer节点上。
			// Forward the message
			gc.Forward(msg)
			// DeMultiplex to local subscribers
			gc.DeMultiplex(m)
		}
		return
	}

    // 如果接收到的是pull
	...
}
```

## 5 gc.blockMsgStore.Add

```go
// add adds a message to the store
func (s *messageStoreImpl) Add(message interface{}) bool {
	s.lock.Lock()
	defer s.lock.Unlock()

	n := len(s.messages)
	for i := 0; i < n; i++ {
		m := s.messages[i]
		switch s.pol(message, m.data) {
		case common.MessageInvalidated:
			return false
		case common.MessageInvalidates:
			s.invTrigger(m.data)
			s.messages = append(s.messages[:i], s.messages[i+1:]...)
			n--
			i--
		}
	}

	s.messages = append(s.messages, &msg{data: message, created: time.Now()})
	return true
}
```

## 6 gc.blocksPuller.Add

blocksPuller对象周期性地发送Pull类数据消息，以拉取并更新本地节点账本上缺失的DataMsg消息

```go
// Add adds a GossipMessage to the store
func (p *pullMediatorImpl) Add(msg *protoext.SignedGossipMessage) {
	p.Lock()
	defer p.Unlock()
    // 提取区块号作为摘要信息
	itemID := p.IdExtractor(msg)
    // 添加到itemID2Msg摘要消息列表
	p.itemID2Msg[itemID] = msg
    // 添加到engine.state摘要对象集合中
	p.engine.Add(itemID)
	p.logger.Debugf("Added %s, total items: %d", itemID, len(p.itemID2Msg))
}
```

## 7 gc.Forward

将该DataMsg消息封装为emittedGossip-Message类型消息，添加到emitter模块缓冲区中，交由emitter模块打包并发送到组织内的其他Peer节点上。

```go
// Forward sends message to the next hops
func (ga *gossipAdapterImpl) Forward(msg protoext.ReceivedMessage) {
	ga.Node.emitter.Add(&emittedGossipMessage{
		SignedGossipMessage: msg.GetGossipMessage(),
		filter:              msg.GetConnectionInfo().ID.IsNotSameFilter,
	})
}
```

## 8 gc.DeMultiplex

```go
// DeMultiplex broadcasts the message to all channels that were returned
// by AddChannel calls and that hold the respected predicates.
//
// Blocks if any one channel that would receive msg has a full buffer.
func (m *ChannelDeMultiplexer) DeMultiplex(msg interface{}) {
	m.lock.Lock()
	if m.closed {
		m.lock.Unlock()
		return
	}
	channels := m.channels
	m.deMuxInProgress.Add(1)
	m.lock.Unlock()

	for _, ch := range channels {
		if ch.pred(msg) {
			select {
			case <-m.stopCh:
				m.deMuxInProgress.Done()
				return // stopping
			case ch.ch <- msg:
			}
		}
	}
	m.deMuxInProgress.Done()
}
```

