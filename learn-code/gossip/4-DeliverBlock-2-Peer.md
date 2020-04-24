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
			if !gc.verifyBlock(m.GossipMessage, msg.GetConnectionInfo().ID) {
				return
			}
			gc.Lock()
			added = gc.blockMsgStore.Add(msg.GetGossipMessage())
			if added {
				gc.blocksPuller.Add(msg.GetGossipMessage())
			}
			gc.Unlock()
		} else { // StateInfoMsg verification should be handled in a layer above
			//  since we don't have access to the id mapper here
			added = gc.stateInfoMsgStore.Add(msg.GetGossipMessage())
		}

		if added {
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

5 messageStoreImpl.Add

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

