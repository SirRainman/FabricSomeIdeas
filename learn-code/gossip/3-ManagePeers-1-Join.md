# 1 管理新加入的peer节点

在初始化GossipService服务器时，peer节点就已经完成了连接其他peer节点的操作了

## 1 入口-初始化GossipSevice服务器

详细的初始化GossipService服务器已经分析过了，这里不再赘述

1. Peer节点启动：internal\peer\node\start.go-serve() 

```go
// FIXME: Creating the gossip service has the side effect of starting a bunch
// of go routines and registration with the grpc server.
gossipService, err := initGossipService(
    policyMgr,
    metricsProvider,
    peerServer,
    signingIdentity,
    cs,
    coreConfig.PeerAddress,
    deliverGRPCClient,
    deliverServiceConfig,
    privdataConfig,
)
defer gossipService.Stop()
```

2. 返回服务器：gossip\service\gossip_service.go

```go
// New creates the gossip service.
func New(
   ...
) (*GossipService, error) {
   serializedIdentity, err := peerIdentity.Serialize()
   

   logger.Infof("Initialize gossip with endpoint %s", endpoint)

   gossipComponent := gossip.New(
      ...
   )

   return &GossipService{
      ...
   }, nil
}
```

3. 返回Gossip服务：gossip\gossip\gossip_impl.go

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
   mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
   secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {
    
   ...
   
   go g.start()
    
   // 重点连接其他节点
   go g.connect2BootstrapPeers()

   return g
}
```

## 2 Peer连接其他Peer节点

### 1 connect2BootstrapPeers

gossip\gossip\gossip_impl.go

```go
func (g *Node) connect2BootstrapPeers() {
    // 1.连接Bootstrap节点列表（peer.gossip.bootstrap配置项）中的节点
    // 2.发送MemReq消息请求获取成员关系信息。
	for _, endpoint := range g.conf.BootstrapPeers {
		endpoint := endpoint
        // identifier()方法用于获取每个远程Bootstrap中peer节点的PeerIdentification
        // 其中，PeerIdentification封装了节点的PKI-ID与SelfOrg标志位（标识是否与当前节点属于同一个组织）
		identifier := func() (*discovery.PeerIdentification, error) {
            // 1.与远程Peer节点建立握手连接，拿到对方的PeerIdentity
			remotePeerIdentity, err := g.comm.Handshake(&comm.RemotePeer{Endpoint: endpoint})
			// 2.检查远程Bootstrap节点与当前节点是否属于同一个MSP组织。
			sameOrg := bytes.Equal(g.selfOrg, g.secAdvisor.OrgByPeerIdentity(remotePeerIdentity))
			// 3.检查通过后基于身份证书信息重新解析获取远程Peer节点的PKI-ID
			pkiID := g.mcs.GetPKIidOfCert(remotePeerIdentity)
			// 4.返回构造远程Bootstrap节点的PeerIdentification
			return &discovery.PeerIdentification{ID: pkiID, SelfOrg: sameOrg}, nil
		}
        
        // 连接其他节点
		g.disc.Connect(discovery.NetworkMember{
			InternalEndpoint: endpoint,
			Endpoint:         endpoint,
		}, identifier)
	}

}
```

### 2 Handshake

gossip\comm\comm_impl.go

与远程peer节点建立连接

```go
func (c *commImpl) Handshake(remotePeer *RemotePeer) (api.PeerIdentityType, error) {
	...
	
	cc, err := grpc.DialContext(ctx, remotePeer.Endpoint, dialOpts...)
	defer cc.Close()

    // 1.为本地Peer节点创建gossipClient客户端
	cl := proto.NewGossipClient(cc)
    
    // 2.与远程Bootstrap节点建立服务连接，获取客户端通信流stream
	stream, err := cl.GossipStream(ctx)

    // 3.认证远程Peer节点的合法性，并创建节点连接信息connInfo
	connInfo, err := c.authenticateRemotePeer(stream, true, true)
    // 4.验证远程Peer节点的PKI-ID与connInfo.ID是否一致。
	if len(remotePeer.PKIID) > 0 && !bytes.Equal(connInfo.ID, remotePeer.PKIID) {
		return nil, fmt.Errorf("PKI-ID of remote peer doesn't match expected PKI-ID")
	}
    // 5.如果通过了检查，则与远程节点正式建立了握手连接，并返回该节点的身份证书信息
	return connInfo.Identity, nil
}
```

### 3 Connect

gossip\discovery\discovery_impl.go

连接除自身节点以外的peer节点

```go
func (d *gossipDiscoveryImpl) Connect(member NetworkMember, id identifier) {
	...
	go func() {
		for i := 0; i < maxConnectionAttempts && !d.toDie(); i++ {
            // 1.获取远程Bootstrap节点的PeerIdentification
			id, err := id()
			// 2.构造对应的网络成员对象
			peer := &NetworkMember{
				InternalEndpoint: member.InternalEndpoint,
				Endpoint:         member.Endpoint,
				PKIid:            id.ID,
			}
            // 3.创建MemReq消息
            // MemReq消息封装了包含自身信息的AliveMsg消息，封装了PKI-ID、网络端点、元数据、节点启动时间、消息序号等信息
			m, err := d.createMembershipRequest(id.SelfOrg)
			
            // 4.添加消息随机数Nonce并签名
			req, err := protoext.NoopSign(m)
			req.Nonce = util.RandomUInt64()
			req, err = protoext.NoopSign(req.GossipMessage)
            
			// 5.将MemReq消息发送到远程的Bootstrap节点上
			go d.sendUntilAcked(peer, req)
			return
		}

	}()
}
```

### 4 sendUntilAcked

尝试发送指定的MemReq消息

```go
func (d *gossipDiscoveryImpl) sendUntilAcked(peer *NetworkMember, message *protoext.SignedGossipMessage) {
	nonce := message.Nonce
	for i := 0; i < maxConnectionAttempts && !d.toDie(); i++ {
        // 1.创建以消息随机数Nonce为主题的消息订阅者sub
		sub := d.pubsub.Subscribe(fmt.Sprintf("%d", nonce), time.Second*5)
		// 2.将请求消息发送到远程的Peer节点上，建立节点连接对象与本地消息处理循环
        d.comm.SendToPeer(peer, message)
        // 3.监听通道并等待主题应答消息
		if _, timeoutErr := sub.Listen(); timeoutErr == nil {
			return
		}
        // 休眠等待d.reconnectInterval，再次循环尝试发送请求消息
		time.Sleep(d.reconnectInterval)
	}
}
```

### 5 Listen

```go
// Listen blocks until a publish was made
// to the subscription, or an error if the
// subscription's TTL passed
func (s *subscription) Listen() (interface{}, error) {
	select {
    // 定时器每5秒触发一次并解除程序阻塞，表示发送消息已经超时，并且没有收到任何应答消息
	case <-time.After(s.ttl):
		return nil, errors.New("timed out")
	// 消息订阅者s.c通道：如果接收到应答消息，则表示远程节点已经接收了该消息
    case item := <-s.c:
		return item, nil
	}
}
```

## 3 远端peer&本地peer接收消息后的处理

远程Bootstrap节点接收到**MemReq**消息之后：

1. 经过comm通信模块上ChannelDe-Multiplexer模块的过滤，
2. 最终交由discovery模块的handleMsgFromComm()方法处理，
   1. 对远程节点上保存的成员关系消息列表进行过滤后，
   2. 获取aliveSnapshot存活节点与deadPeers离线节点的成员关系消息列表，
   3. 并被重新封装为MemRes类型的成员关系响应消息，再回复给本地Peer节点。



本地comm模块通过conn.readFromStream()方法接收到**MemRes**消息：

1. 检查通过后交由conn.handler()方法处理，
2. 实际上是通过comm模块上的msgPublisher对象进行过滤并分发到incMsgs通道，交由discovery模块的handleMsgFromComm()方法处理。
   1. 该方法先处理以Nonce消息随机数为主题的消息订阅请求，将该主题消息放入订阅通道中，通知discovery模块解除sendUntilAcked()→sub.Listen()程序的阻塞等待。
   2. 接着，解析并获取接收消息中的Alive存活节点与Dead离线节点的成员消息列表，更新本地的节点信息相关列表，包括aliveLastTS、deadLastTS、aliveMembership、deadMembership等以及id2Member列表。

### 1 入口-初始化discovery

gossip\gossip\gossip_impl.go

1. 在初始化gossip node中的discover模块时，创建了一个go routine用于接受发送过来的MemReq

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {
    
	...
	g.disc = discovery.NewDiscoveryService(g.selfNetworkMember(), g.discAdapter, g.disSecAdap, g.disclosurePolicy,
		discoveryConfig)
    ...
    
	go g.start()
	go g.connect2BootstrapPeers()

	return g
}
```



2. NewDiscoveryService

   gossip\discovery\discovery_impl.go

   可以从执行的函数看出：peer节点的gossip服务持续不断的发送消息，和收到消息。

```go
// NewDiscoveryService returns a new discovery service with the comm module passed and the crypto service passed
func NewDiscoveryService(self NetworkMember, comm CommService, crypt CryptoService, disPol DisclosurePolicy,
	config DiscoveryConfig) Discovery {
	d := &gossipDiscoveryImpl{
	}

	go d.periodicalSendAlive()
	go d.periodicalCheckAlive()
	go d.handleMessages()
	go d.periodicalReconnectToDead()
	go d.handleEvents()

	return d
}
```

### 2 handleMessages

接收到消息

gossip\discovery\discovery_impl.go

```go
func (d *gossipDiscoveryImpl) handleMessages() {
	// 拿到接收消息的缓冲区
	in := d.comm.Accept()
	for {
		select {
        // 1.如果是收到正常消息
		case m := <-in:
			d.handleMsgFromComm(m)
		// 2.如果gossip服务停止
        case <-d.toDieChan:
			return
		}
	}
}
```

### 3 handleMsgFromComm

gossip\discovery\discovery_impl.go

```go
func (d *gossipDiscoveryImpl) handleMsgFromComm(msg protoext.ReceivedMessage) {
	// 拿到GossipMessage
	m := msg.GetGossipMessage()

    // 1.如果收到的是member request
	if memReq := m.GetMemReq(); memReq != nil {
		// 1.先将请求info部分转化成gossip message
        selfInfoGossipMsg, err := protoext.EnvelopeToGossipMessage(memReq.SelfInformation)

        // 2.解密判断有效性
		if !d.crypt.ValidateAliveMsg(selfInfoGossipMsg) {
			return
		}
        
        // 3.处理消息
		if d.msgStore.CheckValid(selfInfoGossipMsg) {
			d.handleAliveMessage(selfInfoGossipMsg)
		}

		var internalEndpoint string
		if m.Envelope.SecretEnvelope != nil {
			internalEndpoint = protoext.InternalEndpoint(m.Envelope.SecretEnvelope)
		}

		// Sending a membership response to a peer may block this routine
		// in case the sending is deliberately slow (i.e attack).
		// will keep this async until I'll write a timeout detector in the comm layer
        // 4.返回 member response
		go d.sendMemResponse(selfInfoGossipMsg.GetAliveMsg().Membership, internalEndpoint, m.Nonce)
		return
	}
    
	// 2.如果收到的是is Alive
	if protoext.IsAliveMsg(m.GossipMessage) {
		if !d.msgStore.CheckValid(m) || !d.crypt.ValidateAliveMsg(m) {
			return
		}
		// If the message was sent by me, ignore it and don't forward it further
		if d.isSentByMe(m) {
			return
		}

		d.msgStore.Add(m)
		d.handleAliveMessage(m)
		d.comm.Forward(msg)
		return
	}

    // 3.如果收到的是member response
	if memResp := m.GetMemRes(); memResp != nil {
        // 1.处理以Nonce消息随机数为主题的消息订阅请求，将该主题消息放入订阅通道中，通知discovery模块解除sendUntilAcked()→sub.Listen()程序的阻塞等待。
		d.pubsub.Publish(fmt.Sprintf("%d", m.Nonce), m.Nonce)
        
        // 2.获取接收消息中的Alive存活节点的成员消息列表，更新本地的节点信息相关列表，包括aliveLastTS、aliveMembership等以及id2Member列表。
		for _, env := range memResp.Alive {
			am, err := protoext.EnvelopeToGossipMessage(env)
			
			if !protoext.IsAliveMsg(am.GossipMessage) {
				d.logger.Warning("Expected alive message, got", am, "instead")
				return
			}

			if d.msgStore.CheckValid(am) && d.crypt.ValidateAliveMsg(am) {
				d.handleAliveMessage(am)
			}
		}
        
        // 3.获取接收消息中的Dead离线节点的成员消息列表，更新本地的节点信息相关列表，deadLastTS、deadMembership等以及id2Member列表。
		for _, env := range memResp.Dead {
			dm, err := protoext.EnvelopeToGossipMessage(env)

			// Newer alive message exists or the message isn't authentic
			if !d.msgStore.CheckValid(dm) || !d.crypt.ValidateAliveMsg(dm) {
				continue
			}

			newDeadMembers := []*protoext.SignedGossipMessage{}
			d.lock.RLock()
			if _, known := d.id2Member[string(dm.GetAliveMsg().Membership.PkiId)]; !known {
				newDeadMembers = append(newDeadMembers, dm)
			}
			d.lock.RUnlock()
			d.learnNewMembers([]*protoext.SignedGossipMessage{}, newDeadMembers)
		}
	}
}
```

### 4 handleAliveMessage

```go
func (d *gossipDiscoveryImpl) handleAliveMessage(m *protoext.SignedGossipMessage) {

	if d.isSentByMe(m) {
		return
	}

	pkiID := m.GetAliveMsg().Membership.PkiId

	ts := m.GetAliveMsg().Timestamp

	d.lock.RLock()
	_, known := d.id2Member[string(pkiID)]
	d.lock.RUnlock()

	if !known {
		d.learnNewMembers([]*protoext.SignedGossipMessage{m}, []*protoext.SignedGossipMessage{})
		return
	}

	d.lock.RLock()
	_, isAlive := d.aliveLastTS[string(pkiID)]
	lastDeadTS, isDead := d.deadLastTS[string(pkiID)]
	d.lock.RUnlock()

	if !isAlive && !isDead {
		d.logger.Panicf("Member %s is known but not found neither in alive nor in dead lastTS maps, isAlive=%v, isDead=%v", m.GetAliveMsg().Membership.Endpoint, isAlive, isDead)
		return
	}

	if isAlive && isDead {
		d.logger.Panicf("Member %s is both alive and dead at the same time", m.GetAliveMsg().Membership)
		return
	}

	if isDead {
		if before(lastDeadTS, ts) {
			// resurrect peer
			d.resurrectMember(m, *ts)
		} else if !same(lastDeadTS, ts) {
			d.logger.Debug("got old alive message about dead peer ", protoext.MemberToString(m.GetAliveMsg().Membership), "lastDeadTS:", lastDeadTS, "but got ts:", ts)
		}
		return
	}

	d.lock.RLock()
	lastAliveTS, isAlive := d.aliveLastTS[string(pkiID)]
	d.lock.RUnlock()

	if isAlive {
		if before(lastAliveTS, ts) {
			d.learnExistingMembers([]*protoext.SignedGossipMessage{m})
		} else if !same(lastAliveTS, ts) {
			d.logger.Debug("got old alive message about alive peer ", protoext.MemberToString(m.GetAliveMsg().Membership), "lastAliveTS:", lastAliveTS, "but got ts:", ts)
		}

	}
	// else, ignore the message because it is too old
}
```

### 5 Publish

处理以Nonce消息随机数为主题的消息订阅请求，将该主题消息放入订阅通道中，

通知discovery模块解除sendUntilAcked()→sub.Listen()程序的阻塞等待。

```go
// Publish publishes an item to all subscribers on the topic
func (ps *PubSub) Publish(topic string, item interface{}) error {
	ps.RLock()
	defer ps.RUnlock()
    
	s, subscribed := ps.subscriptions[topic]
	if !subscribed {
		return errors.New("no subscribers")
	}
	
    for _, sub := range s.ToArray() {
		c := sub.(*subscription).c
		select {
		case c <- item:
		default: // Not enough room in buffer, continue in order to not block publisher
		}
	}
	return nil
}
```

