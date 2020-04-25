# 3 更新节点信息

Gossip Service服务器中的discover模块会周期性的发送/接收gossip msg，从而完成对网络中存活和死亡的peer节点的管理。

**这里只分析发送端，接受端接收到存活信息更新部分在成员管理的Join那部分已经分析过了**

## 1.更新节点存活信息

### 1.入口-更新存活节点信息

gossip\discovery\discovery_impl.go

在初始化gossip服务中的discovery模块时，discovery模块周期性的向维护的节点列表中的其他peer节点发送存活信息

```go
// NewDiscoveryService returns a new discovery service with the comm module passed and the crypto service passed
func NewDiscoveryService(self NetworkMember, comm CommService, crypt CryptoService, disPol DisclosurePolicy,
	...

	go d.periodicalSendAlive()
	go d.periodicalCheckAlive()
	go d.handleMessages()
	go d.periodicalReconnectToDead()
	go d.handleEvents()

	return d
}
```

### 2.periodicalSendAlive

```go
func (d *gossipDiscoveryImpl) periodicalSendAlive() {
	for !d.toDie() {
		time.Sleep(d.aliveTimeInterval)
        // 1.构造本地节点的AliveMsg类型节点存活消息，封装本地节点PKI-ID、网络端点、元数据、节点启动时间、消息序号等。
		msg, err := d.createSignedAliveMessage(true)
		d.lock.Lock()
		d.selfAliveMessage = msg
		d.lock.Unlock()
        // 2.将AliveMsg消息封装为emittedGossipMessage类型消息，添加到emitter模块的消息缓冲区中，等待发送给通道组织中的其他Peer节点
		d.comm.Gossip(msg)
	}
}
```

#### 1.createSignedAliveMessage

```go
func (d *gossipDiscoveryImpl) createSignedAliveMessage(includeInternalEndpoint bool) (*protoext.SignedGossipMessage, error) {
	msg, internalEndpoint := d.aliveMsgAndInternalEndpoint()
    // 对该消息进行签名，并封装为SignedGossipMessage类型消息。
	envp := d.crypt.SignMessage(msg, internalEndpoint)
	
	signedMsg := &protoext.SignedGossipMessage{
		GossipMessage: msg,
		Envelope:      envp,
	}

	if !includeInternalEndpoint {
		signedMsg.Envelope.SecretEnvelope = nil
	}

	return signedMsg, nil
}
```

#### 2.SignMessage

```go
// SignMessage signs an AliveMessage and updates its signature field
func (sa *discoverySecurityAdapter) SignMessage(m *pg.GossipMessage, internalEndpoint string) *pg.Envelope {
	signer := func(msg []byte) ([]byte, error) {
		return sa.mcs.Sign(msg)
	}
    // 如果当前时间运行在discovery-SecurityAdapter模块规定的时间includeIdentityPeriod（默认为10秒，peer.gossip.publish-CertPeriod配置项）内，则需要在AliveMsg消息中添加本地Peer节点的身份证书信息。
	if protoext.IsAliveMsg(m) && time.Now().Before(sa.includeIdentityPeriod) {
		m.GetAliveMsg().Identity = sa.identity
	}
	sMsg := &protoext.SignedGossipMessage{
		GossipMessage: m,
	}
	e, err := sMsg.Sign(signer)

	if internalEndpoint == "" {
		return e
	}
	protoext.SignSecret(e, signer, &pg.Secret{
		Content: &pg.Secret_InternalEndpoint{
			InternalEndpoint: internalEndpoint,
		},
	})
	return e
}
```

### 3.Gossip()

```go
func (da *discoveryAdapter) Gossip(msg *protoext.SignedGossipMessage) {
	if da.toDie() {
		return
	}

	da.gossipFunc(msg)
}
```

gossipFunc在初始化newDiscoveryAdapter时就定义了

gossip\gossip\gossip_impl.go

```go
func (g *Node) newDiscoveryAdapter() *discoveryAdapter {
	return &discoveryAdapter{
		c:        g.comm,
		stopping: int32(0),
		gossipFunc: func(msg *protoext.SignedGossipMessage) {
            // 检查发送节点数量
			if g.conf.PropagateIterations == 0 {
				return
			}
            // 添加到emitter模块并等待发送
			g.emitter.Add(&emittedGossipMessage{
				SignedGossipMessage: msg,
				filter: func(_ common.PKIidType) bool {
					return true
				},
			})
		},
		...
	}
}
```



## 2.更新节点成员关系信息

### 1.入口-Gossip服务start()

1. 初始化Gossip服务时，进行节点启动start()

   gossip\gossip\gossip_impl.go

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

2. start()

   gossip\gossip\gossip_impl.go

```go
func (g *Node) start() {
    // 创建消息处理循环
	go g.syncDiscovery()
	go g.handlePresumedDead()

	incMsgs := g.comm.Accept(msgSelector)
	go g.acceptMessages(incMsgs)
}
```

### 2.syncDiscovery

```go
func (g *Node) syncDiscovery() {
	
	for !g.toDie() {
        // 随机向发送g.conf.PullPeerNum个成员发送MemReq消息，以获取其他节点的成员关系消息，并更新本地的相关信息列表
		g.disc.InitiateSync(g.conf.PullPeerNum)
        // 间隔一段时间
		time.Sleep(g.conf.PullInterval)
	}
}
```

### 3.InitiateSync

```go
func (d *gossipDiscoveryImpl) InitiateSync(peerNum int) {
	if d.toDie() {
		return
	}
	var peers2SendTo []*NetworkMember
    // 1.创建节点成员关系请求消息
	m, err := d.createMembershipRequest(true)
	memReq, err := protoext.NoopSign(m)
	
	d.lock.RLock()

    // k=peer.gossip.pullPeerNum配置文件中的配置项，随机选择k个节点，这里确定k的大小
	n := d.aliveMembership.Size()
	k := peerNum
	if k > n {
		k = n
	}

    // 转换为数组
	aliveMembersAsSlice := d.aliveMembership.ToSlice()
    // 随机选择k个节点
	for _, i := range util.GetRandomIndices(k, n-1) {
        // 获取AliveMsg消息中的节点成员信息
		pulledPeer := aliveMembersAsSlice[i].GetAliveMsg().Membership
		var internalEndpoint string
		if aliveMembersAsSlice[i].Envelope.SecretEnvelope != nil {
			internalEndpoint = protoext.InternalEndpoint(aliveMembersAsSlice[i].Envelope.SecretEnvelope)
		}
        // 将节点的端点、元数据、PKI-ID、内部端点（若存在）等封装为NetworkMember网络成员对象
		netMember := &NetworkMember{
			Endpoint:         pulledPeer.Endpoint,
			Metadata:         pulledPeer.Metadata,
			PKIid:            pulledPeer.PkiId,
			InternalEndpoint: internalEndpoint,
		}
        // 添加到待发送节点列表peers2SendTo中
		peers2SendTo = append(peers2SendTo, netMember)
	}

	d.lock.RUnlock()

    // 遍历待发送Peer节点列表，对MemReq消息进行过滤与签名之后，基于comm通信模块发送到指定节点。
	for _, netMember := range peers2SendTo {
		d.comm.SendToPeer(netMember, memReq)
	}
}
```

### 4.SendToPeer

gossip\gossip\gossip_impl.go

```go
func (da *discoveryAdapter) SendToPeer(peer *discovery.NetworkMember, msg *protoext.SignedGossipMessage) {
	if da.toDie() {
		return
	}
	// Check membership requests for peers that we know of their PKI-ID.
	// The only peers we don't know about their PKI-IDs are bootstrap peers.
	if memReq := msg.GetMemReq(); memReq != nil && len(peer.PKIid) != 0 {
		selfMsg, err := protoext.EnvelopeToGossipMessage(memReq.SelfInformation)
		
		// Apply the EnvelopeFilter of the disclosure policy
		// on the alive message of the selfInfo field of the membership request
		_, omitConcealedFields := da.disclosurePolicy(peer)
		selfMsg.Envelope = omitConcealedFields(selfMsg)
		// Backup old known field
		oldKnown := memReq.Known
		// Override new SelfInfo message with updated envelope
		memReq = &pg.MembershipRequest{
			SelfInformation: selfMsg.Envelope,
			Known:           oldKnown,
		}
		msgCopy := proto.Clone(msg.GossipMessage).(*pg.GossipMessage)

		// Update original message
		msgCopy.Content = &pg.GossipMessage_MemReq{
			MemReq: memReq,
		}
		// Update the envelope of the outer message, no need to sign (point2point)
		msg, err = protoext.NoopSign(msgCopy)
		
		da.c.Send(msg, &comm.RemotePeer{PKIID: peer.PKIid, Endpoint: peer.PreferredEndpoint()})
		return
	}
	da.c.Send(msg, &comm.RemotePeer{PKIID: peer.PKIid, Endpoint: peer.PreferredEndpoint()})
}
```



## 3.更新节点身份消息

certStore模块通过Gossip服务实例的certPuller模块周期性地发送Pull类节点身份消息，请求拉取其他节点的PeerIdentity类型节点身份消息，更新本地idMapper模块的节点身份信息字典pkiID2Cert。

### 1.入口-gossip 服务初始化

1. 初始化Gossip服务

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {
	...
    
	g.certPuller = g.createCertStorePuller()
	g.certStore = newCertStore(g.certPuller, g.idMapper, selfIdentity, mcs)

	return g
}
```

### 2.createCertStorePuller

```go
func (g *Node) createCertStorePuller() pull.Mediator {
	...
	return pull.NewPullMediator(conf, adapter)
}
```

### 3.NewPullMediator

```go
// NewPullMediator returns a new Mediator
func NewPullMediator(config Config, adapter *PullAdapter) Mediator {
	...
	p.engine = algo.NewPullEngineWithFilter(p, config.PullInterval, egressDigFilter.byContext(), config.PullEngineConfig)

	return p
}
```

### 4.NewPullEngineWithFilter

```go
// NewPullEngineWithFilter creates an instance of a PullEngine with a certain sleep time
// between pull initiations, and uses the given filters when sending digests and responses
func NewPullEngineWithFilter(participant PullAdapter, sleepTime time.Duration, df DigestFilter,
	...
	go func() {
		for !engine.toDie() {
			time.Sleep(sleepTime)
			if engine.toDie() {
				return
			}
            // 选择特定节点发送Pull类Hello消息，请求PeerIdentity类型的节点身份信息，等待回复消息并更新到本地。
			engine.initiatePull()
		}
	}()
	return engine
}
```

### 5.initiatePull

gossip\gossip\algo\pull.go

选择特定节点发送Pull类Hello消息，请求PeerIdentity类型的节点身份信息，等待回复消息并更新到本地。

```go
func (engine *PullEngine) initiatePull() {
	engine.lock.Lock()
	defer engine.lock.Unlock()

    // 1.设置可以接收Digest消息
	engine.acceptDigests()
    // 2.遍历选择的Peer节点列表发送数据
	for _, peer := range engine.SelectPeers() {
        // 2.1创建随机数Nonce
		nonce := engine.newNONCE()
		// 2.2添加Nonce随机数
        engine.outgoingNONCES.Add(nonce)
        // 2.3设置与Nonce对应的peer节点
		engine.nonces2peers[nonce] = peer
        // 2.4设置与peer节点对应的Nonce
		engine.peers2nonces[peer] = nonce
        // 2.5向该peer节点发送带有Nonce的Hello消息
		engine.Hello(peer, nonce)
	}
	// 3.定时器启动处理Digest摘要消息
	time.AfterFunc(engine.digestWaitTime, func() {
		engine.processIncomingDigests()
	})
}
```

### 6.engine.Hello

```go
// Hello sends a hello message to initiate the protocol
// and returns an NONCE that is expected to be returned
// in the digest message.
func (p *pullMediatorImpl) Hello(dest string, nonce uint64) {
	...

	p.logger.Debug("Sending", p.config.MsgType, "hello to", dest)
	sMsg, err := protoext.NoopSign(helloMsg)
	p.Sndr.Send(sMsg, p.peersWithEndpoints(dest)...)
}
```



### 7.接受端接受到拉取身份请求

**这部分没看懂啊啊啊啊啊啊！**

#### 1.入口-Gossip服务启动start()

gossip\gossip\gossip_impl.go

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {
	
	go g.start()
	go g.connect2BootstrapPeers()

	return g
}

```



```go
func (g *Node) start() {
	go g.syncDiscovery()
	go g.handlePresumedDead()
	
    // 接收信息
	incMsgs := g.comm.Accept(msgSelector)
	go g.acceptMessages(incMsgs)

}
```

#### 2.acceptMessages

```go
func (g *Node) acceptMessages(incMsgs <-chan protoext.ReceivedMessage) {
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

#### 3.g.handleMessage

```go
func (g *Node) handleMessage(m protoext.ReceivedMessage) {
	...	

	if protoext.IsPullMsg(msg.GossipMessage) && protoext.GetPullMsgType(msg.GossipMessage) == pg.PullMsgType_IDENTITY_MSG {
		g.certStore.handleMessage(m)
	}
}
```

#### 4.g.certStore.handleMessage

```go
func (cs *certStore) handleMessage(msg protoext.ReceivedMessage) {
	if update := msg.GetGossipMessage().GetDataUpdate(); update != nil {
		for _, env := range update.Data {
			m, err := protoext.EnvelopeToGossipMessage(env)
			
			if !protoext.IsIdentityMsg(m.GossipMessage) {
				cs.logger.Warning("Got a non-identity message:", m, "aborting")
				return
			}
			if err := cs.validateIdentityMsg(m); err != nil {
				cs.logger.Warningf("Failed validating identity message: %+v", errors.WithStack(err))
				return
			}
		}
	}
	cs.pull.HandleMessage(msg)
}
```

#### 5.cs.pull.HandleMessage(msg)

```go
func (p *pullMediatorImpl) HandleMessage(m protoext.ReceivedMessage) {
	if m.GetGossipMessage() == nil || !protoext.IsPullMsg(m.GetGossipMessage().GossipMessage) {
		return
	}
	msg := m.GetGossipMessage()
	msgType := protoext.GetPullMsgType(msg.GossipMessage)
	if msgType != p.config.MsgType {
		return
	}

	p.logger.Debug(msg)

	itemIDs := []string{}
	items := []*protoext.SignedGossipMessage{}
	var pullMsgType MsgType

	if helloMsg := msg.GetHello(); helloMsg != nil {
		pullMsgType = HelloMsgType
		p.engine.OnHello(helloMsg.Nonce, m)
	} else if digest := msg.GetDataDig(); digest != nil {
		d := p.PullAdapter.IngressDigFilter(digest)
		itemIDs = util.BytesToStrings(d.Digests)
		pullMsgType = DigestMsgType
		p.engine.OnDigest(itemIDs, d.Nonce, m)
	} else if req := msg.GetDataReq(); req != nil {
		itemIDs = util.BytesToStrings(req.Digests)
		pullMsgType = RequestMsgType
		p.engine.OnReq(itemIDs, req.Nonce, m)
	} else if res := msg.GetDataUpdate(); res != nil {
		itemIDs = make([]string, len(res.Data))
		items = make([]*protoext.SignedGossipMessage, len(res.Data))
		pullMsgType = ResponseMsgType
		for i, pulledMsg := range res.Data {
			msg, err := protoext.EnvelopeToGossipMessage(pulledMsg)
			if err != nil {
				p.logger.Warningf("Data update contains an invalid message: %+v", errors.WithStack(err))
				return
			}
			p.MsgCons(msg)
			itemIDs[i] = p.IdExtractor(msg)
			items[i] = msg
			p.Lock()
			p.itemID2Msg[itemIDs[i]] = msg
			p.logger.Debugf("Added %s to the in memory item map, total items: %d", itemIDs[i], len(p.itemID2Msg))
			p.Unlock()
		}
		p.engine.OnRes(itemIDs, res.Nonce)
	}

	// Invoke hooks for relevant message type
	for _, h := range p.hooksByMsgType(pullMsgType) {
		h(itemIDs, items, m)
	}
}
```



## 4.清理节点身份信息

### 1.入口-gossip 服务初始化-g.idMapper 

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {
	...
    
	g.idMapper = identity.NewIdentityMapper(mcs, selfIdentity, func(pkiID common.PKIidType, identity api.PeerIdentityType) {
		g.comm.CloseConn(&comm.RemotePeer{PKIID: pkiID})
		g.certPuller.Remove(string(pkiID))
	}, sa)

	go g.start()
	go g.connect2BootstrapPeers()

	return g
}	
```

### 2.NewIdentityMapper

```go
// NewIdentityMapper method, all we need is a reference to a MessageCryptoService
func NewIdentityMapper(mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType, onPurge purgeTrigger, sa api.SecurityAdvisor) Mapper {
	...
	go idMapper.periodicalPurgeUnusedIdentities()
	return idMapper
}
```

### 3.idMapper.periodicalPurgeUnusedIdentities

```go
func (is *identityMapperImpl) periodicalPurgeUnusedIdentities() {
	usageTh := GetIdentityUsageThreshold()
	for {
		select {
		case <-is.stopChan:
			return
		case <-time.After(usageTh / 10):
			is.SuspectPeers(func(_ api.PeerIdentityType) bool {
				return false
			})
		}
	}
}

```

### 4.SuspectPeers

```go
// SuspectPeers re-validates all peers that match the given predicate
func (is *identityMapperImpl) SuspectPeers(isSuspected api.PeerSuspector) {
	for _, identity := range is.validateIdentities(isSuspected) {
        // 停止身份过期定时器
		identity.cancelExpirationTimer()
        // 从idMapper模块的pkiID2Cert字典中删除已过期、被撤销及身份证书无效的Peer节点身份信息
		is.delete(identity.pkiID, identity.peerIdentity)
	}
}
```

### 5.validateIdentities

```go
// validateIdentities returns a list of identities that have been revoked, expired or haven't been used for a long time
func (is *identityMapperImpl) validateIdentities(isSuspected api.PeerSuspector) []*storedIdentity {
	now := time.Now()
	usageTh := GetIdentityUsageThreshold()
	is.RLock()
	defer is.RUnlock()
    
	var revokedIdentities []*storedIdentity
    // 循环遍历本地idMapper模块的pkiID2Cert字典
	for pkiID, storedIdentity := range is.pkiID2Cert {
        // 检查该节点身份信息的最后访问时间lastAccessTime
		if pkiID != is.selfPKIID && storedIdentity.fetchLastAccessTime().Add(usageTh).Before(now) {
            // 如果lastAccessTime与当前时间的时间差超过1个小时，则撤销该节点的身份信息并添加到revokedIdentities列表（[]*storedIdentity）中
			revokedIdentities = append(revokedIdentities, storedIdentity)
			continue
		}
        // isSuspected默认返回false
		if !isSuspected(storedIdentity.peerIdentity) {
			continue
		}
        // 检查身份信息的有效性
		if err := is.mcs.ValidateIdentity(storedIdentity.fetchIdentity()); err != nil {
			revokedIdentities = append(revokedIdentities, storedIdentity)
		}
	}
	return revokedIdentities
}
```

### 6.delete

```go
func (is *identityMapperImpl) delete(pkiID common.PKIidType, identity api.PeerIdentityType) {
	is.Lock()
	defer is.Unlock()
	is.onPurge(pkiID, identity)
    // 从idMapper模块的pkiID2Cert字典中删除指定PKI-ID关联的Peer节点身份信息键值对。
	delete(is.pkiID2Cert, string(pkiID))
}
```

### 7.is.onPurge

onPurge这个函数在初始化identityMapper就制定了

```go
// 创建idMapper模块，定义了onPurge()回调函数
g.idMapper = identity.NewIdentityMapper(mcs, selfIdentity, func(pkiID common.PKIidType, identity api.PeerIdentityType) {
    	// 关闭与identity关联节点（pkiID）的连接，从comm通信模块的pki2Conn列表中删除对应的节点连接对象键值对。
		g.comm.CloseConn(&comm.RemotePeer{PKIID: pkiID})
    	// 从certStore模块的itemID2Msg消息列表与engine对象中删除identity关联节点的节点身份消息与摘要信息（节点PKI-ID）
		g.certPuller.Remove(string(pkiID))
	}, sa)
```

```go
func NewIdentityMapper(mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType, onPurge purgeTrigger, sa api.SecurityAdvisor) Mapper {
	
}
```

