# 2 Peer节点接收Block消息

## 1.入口-Gossip服务启动

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

   gossip\gossip\gossip_impl.go

```go
func (g *Node) start() {
	go g.syncDiscovery()
	go g.handlePresumedDead()
	
    // 接收到消息
	incMsgs := g.comm.Accept(msgSelector)
	go g.acceptMessages(incMsgs)
}
```

## 2.Node.acceptMessages

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

## 3.Node.handleMessage

```go
func (g *Node) handleMessage(m protoext.ReceivedMessage) {
	...

	msg := m.GetGossipMessage()

	if !g.validateMsg(m) {
		g.logger.Warning("Message", msg, "isn't valid")
		return
	}

    // 1.IsChannelRestricted returns whether this GossipMessage should be routed only in its channel
	if protoext.IsChannelRestricted(msg.GossipMessage) {
        //1如果不是该通道中的消息，考虑到这条消息可能是state，仍然应该传给同org中的其他的peer
		if gc := g.chanState.lookupChannelForMsg(m); gc == nil {
			if g.IsInMyOrg(discovery.NetworkMember{PKIid: m.GetConnectionInfo().ID}) && protoext.IsStateInfoMsg(msg.GossipMessage) {
				if g.stateInfoMsgStore.Add(msg) {
					...
				}
			}
            
            ...
		} else {
        //2如果是该通道中的消息
			if protoext.IsLeadershipMsg(m.GetGossipMessage().GossipMessage) {
				if err := g.validateLeadershipMessage(m.GetGossipMessage()); err != nil {
					return
				}
			}
			gc.HandleMessage(m)
		}
		return
	}
    
	...
}
```



## 4.Node.stateInfoMsgStore.Add

gossip\gossip\msgstore\msgs.go

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



## 5.gossipChannel.HandleMessage

gossip\gossip\channel\channel.go

```go
// HandleMessage processes a message sent by a remote peer
func (gc *gossipChannel) HandleMessage(msg protoext.ReceivedMessage) {
	//前面是异常处理部分，在此省略
    ...
    
    // 1.
	if protoext.IsStateInfoPullRequestMsg(m.GossipMessage) {
		msg.Respond(gc.createStateInfoSnapshot(orgID))
		return
	}

    // 2.
	if protoext.IsStateInfoSnapshot(m.GossipMessage) {
		gc.handleStateInfSnapshot(m.GossipMessage, msg.GetConnectionInfo().ID)
		return
	}

    // 3.如果接收到的是data或者是stateinfo
	if protoext.IsDataMsg(m.GossipMessage) || protoext.IsStateInfoMsg(m.GossipMessage) {
		added := false
        // 1.如果接收到的是data
		if protoext.IsDataMsg(m.GossipMessage) {
			// 1.检查是否有data要存到缓冲区中
            if m.GetDataMsg().Payload == nil {
				return
			}
			// 2.Would this block go into the message store if it was verified?
			if !gc.blockMsgStore.CheckValid(msg.GetGossipMessage()) {
				return
			}
            // 3.通过MCS消息加密服务模块验证该区块结构的合法性，以及区块元数据中的签名是否满足指定的区块验证策略BlockValidation。
			if !gc.verifyBlock(m.GossipMessage, msg.GetConnectionInfo().ID) {
				return
			}
            // 4.将DataMsg消息添加到blockMsgStore与blocksPuller消息存储对象上
			gc.Lock()
			added = gc.blockMsgStore.Add(msg.GetGossipMessage())
			if added {
                // blocksPuller对象周期性地发送Pull类数据消息，以拉取并更新本地节点账本上缺失的DataMsg消息
				gc.blocksPuller.Add(msg.GetGossipMessage())
			}
			gc.Unlock()
		} else { 
        // 2.如果接收到的是stateinfo
            // StateInfoMsg verification should be handled in a layer above
			//  since we don't have access to the id mapper here
			added = gc.stateInfoMsgStore.Add(msg.GetGossipMessage())
		}

		if added {
            // 将该DataMsg消息封装为emittedGossip-Message类型消息，添加到emitter模块缓冲区中，交由emitter模块打包并发送到组织内的其他Peer节点上。
			// Forward the message
			gc.Forward(msg)
			// DeMultiplex to local subscribers
			gossipChannel.DeMultiplex(m)
		}
		return
	}

    // 4.如果接收到的是pull
	if protoext.IsPullMsg(m.GossipMessage) && protoext.GetPullMsgType(m.GossipMessage) == proto.PullMsgType_BLOCK_MSG {
		...
	}
    
    // 5.如果接收到的是选举信息
    ...
}
```

## 6.messageStoreImpl.Add

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

## 7.pullMediatorImpl.Add

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

## 8.gossipAdapterImpl.Forward

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

## 9.ChannelDeMultiplexer.DeMultiplex

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

# 3 Peer定期请求更新数据

## 1.入口

### 1.serve

peer容器执行peer node start指令后

internal\peer\node\start.go

```go
func serve(args []string) error {	
	...
    
    // this brings up all the channels
	peerInstance.Initialize(
		
	)
	...
}
```

### 2.Peer.Initialize

core\peer\peer.go

```go
// Initialize sets up any channels that the peer has from the persistence. This
// function should be called at the start up when the ledger and gossip
// ready
func (p *Peer) Initialize(
	init func(string),
	server *comm.GRPCServer,
	pm plugin.Mapper,
	deployedCCInfoProvider ledger.DeployedChaincodeInfoProvider,
	legacyLifecycleValidation plugindispatcher.LifecycleResources,
	newLifecycleValidation plugindispatcher.CollectionAndLifecycleResources,
	nWorkers int,
) {
	...
	
    // 1.拿到所有的账本
	ledgerIds, err := p.LedgerMgr.GetLedgerIDs()

	for _, cid := range ledgerIds {
		peerLogger.Infof("Loading chain %s", cid)
		// 2.加载每一个账本
        ledger, err := p.LedgerMgr.OpenLedger(cid)
		
		// 3.Create a chain if we get a valid ledger with config block
		err = p.createChannel(cid, ledger, deployedCCInfoProvider, legacyLifecycleValidation, newLifecycleValidation)

        // 4.
		p.initChannel(cid)
	}
}
```

### 3.Peer.createChannel

core\peer\peer.go

```go
// createChannel creates a new channel object and insert it into the channels slice.
func (p *Peer) createChannel(
	cid string,
	l ledger.PeerLedger,
	deployedCCInfoProvider ledger.DeployedChaincodeInfoProvider,
	legacyLifecycleValidation plugindispatcher.LifecycleResources,
	newLifecycleValidation plugindispatcher.CollectionAndLifecycleResources,
) error {
    
    ...
    
	p.GossipService.InitializeChannel(bundle.ConfigtxValidator().ChannelID(), ordererSource, store, gossipservice.Support{
		Validator:       validator,			// 交易验证器
		Committer:       committer,			// 账本提交器
		CollectionStore: simpleCollectionStore,		// 隐私数据集合存储对象
		IdDeserializeFactory: gossipprivdata.IdentityDeserializerFactoryFunc(func(chainID string) msp.IdentityDeserializer {
			return mspmgmt.GetManagerForChain(chainID)
		}),
		CapabilityProvider: channel,
	})

	p.mutex.Lock()
	defer p.mutex.Unlock()
    // 14.构造新链结构并插入Peer节点上的链结构字典
	if p.channels == nil {
		p.channels = map[string]*Channel{}
	}
	p.channels[cid] = channel

	return nil
}
```

### 4.configEventer.ProcessConfigUpdate

gossip\service\eventer.go

```go
// ProcessConfigUpdate should be invoked whenever a channel's configuration is initialized or updated
// it invokes the associated method in configEventReceiver when configuration is updated
// but only if the configuration value actually changed
// Note, that a changing sequence number is ignored as changing configuration
func (ce *configEventer) ProcessConfigUpdate(config Config) {
	logger.Debugf("Processing new config for channel %s", config.ChannelID())
	orgMap := cloneOrgConfig(config.Organizations())
	if ce.lastConfig != nil && reflect.DeepEqual(ce.lastConfig.orgMap, orgMap) {
		logger.Debugf("Ignoring new config for channel %s because it contained no anchor peer updates", config.ChannelID())
	} else {
		// 1.拿到所有的anchor peer
		var newAnchorPeers []*peer.AnchorPeer
		for _, group := range config.Organizations() {
			newAnchorPeers = append(newAnchorPeers, group.AnchorPeers()...)
		}
		// 2.更新配置
		newConfig := &configStore{
			orgMap:      orgMap,
			anchorPeers: newAnchorPeers,
		}
		ce.lastConfig = newConfig
		// 3.更新anchor peer
		logger.Debugf("Calling out because config was updated for channel %s", config.ChannelID())
		ce.receiver.updateAnchors(config)
	}
}
```

### 5.GossipService.updateAnchors

gossip\service\gossip_service.go

```go
// updateAnchors constructs a joinChannelMessage and sends it to the gossipSvc
func (g *GossipService) updateAnchors(config Config) {
	// 1.判断该peer所在的org是不是在channel中
    myOrg := string(g.secAdv.OrgByPeerIdentity(api.PeerIdentityType(g.peerIdentity)))
	if !g.amIinChannel(myOrg, config) {
		logger.Error("Tried joining channel", config.ChannelID(), "but our org(", myOrg, "), isn't "+
			"among the orgs of the channel:", orgListFromConfig(config), ", aborting.")
		return
	}
    // 2.构造加入channel的消息
	jcm := &joinChannelMessage{seqNum: config.Sequence(), members2AnchorPeers: map[string][]api.AnchorPeer{}}
	for _, appOrg := range config.Organizations() {
		logger.Debug(appOrg.MSPID(), "anchor peers:", appOrg.AnchorPeers())
		jcm.members2AnchorPeers[appOrg.MSPID()] = []api.AnchorPeer{}
		for _, ap := range appOrg.AnchorPeers() {
			anchorPeer := api.AnchorPeer{
				Host: ap.Host,
				Port: int(ap.Port),
			}
			jcm.members2AnchorPeers[appOrg.MSPID()] = append(jcm.members2AnchorPeers[appOrg.MSPID()], anchorPeer)
		}
	}

	// Initialize new state provider for given committer
	logger.Debug("Creating state provider for channelID", config.ChannelID())
	// 3.gossip node 加入到通道中
    g.JoinChan(jcm, gossipcommon.ChannelID(config.ChannelID()))
}
```

### 6.Node.JoinChan

gossip\gossip\gossip_impl.go

```go
// JoinChan makes gossip participate in the given channel, or update it.
func (g *Node) JoinChan(joinMsg api.JoinChannelMessage, channelID common.ChannelID) {
	// joinMsg is supposed to have been already verified
    
    // 1.gossip node的chanstate模块执行加入通道命令
	g.chanState.joinChannel(joinMsg, channelID, g.gossipMetrics.MembershipMetrics)
	
    // 2.更新member信息，并连接
	g.logger.Info("Joining gossip network of channel", string(channelID), "with", len(joinMsg.Members()), "organizations")
	for _, org := range joinMsg.Members() {
		g.learnAnchorPeers(string(channelID), org, joinMsg.AnchorPeersOf(org))
	}
}
```

### 7.channelState.joinChan

gossip\gossip\chanstate.go

```go
func (cs *channelState) joinChannel(joinMsg api.JoinChannelMessage, channelID common.ChannelID,
	metrics *metrics.MembershipMetrics) {
	if cs.isStopping() {
		return
	}
	cs.Lock()
	defer cs.Unlock()
	if gc, exists := cs.channels[string(channelID)]; !exists {
		pkiID := cs.g.comm.GetPKIid()
		ga := &gossipAdapterImpl{Node: cs.g, Discovery: cs.g.disc}
		gc := channel.NewGossipChannel(pkiID, cs.g.selfOrg, cs.g.mcs, channelID, ga, joinMsg, metrics, nil)
        // 1.如果channelState模块维护的channels中没有该channel则重新构造一个
		cs.channels[string(channelID)] = gc
	} else {
        // 2.如果已存在则更新加入信息
		gc.ConfigureChannel(joinMsg)
	}
}
```

### 8.channel.NewGossipChannel

gossip\gossip\channel\channel.go

```go
// NewGossipChannel creates a new GossipChannel
func NewGossipChannel(pkiID common.PKIidType, org api.OrgIdentityType, mcs api.MessageCryptoService,
	channelID common.ChannelID, adapter Adapter, joinMsg api.JoinChannelMessage,
	metrics *metrics.MembershipMetrics, logger util.Logger) GossipChannel {
    
    ...
    
	gc := &gossipChannel{
		...
	}
	
	// 3.完成pull block的工作
	gc.blocksPuller = gc.createBlockPuller()
	
    // 14.
    return gc
}
```

## 2.gossipChannel.createBlockPuller

```go
func (gc *gossipChannel) createBlockPuller() pull.Mediator {
	conf := pull.Config{
		MsgType:           proto.PullMsgType_BLOCK_MSG,
		Channel:           []byte(gc.chainID),
		ID:                gc.GetConf().ID,
		PeerCountToSelect: gc.GetConf().PullPeerNum,
		PullInterval:      gc.GetConf().PullInterval,
		Tag:               proto.GossipMessage_CHAN_AND_ORG,
		PullEngineConfig: algo.PullEngineConfig{
			DigestWaitTime:   gc.GetConf().DigestWaitTime,
			RequestWaitTime:  gc.GetConf().RequestWaitTime,
			ResponseWaitTime: gc.GetConf().ResponseWaitTime,
		},
	}
	seqNumFromMsg := func(msg *protoext.SignedGossipMessage) string {
		dataMsg := msg.GetDataMsg()
		if dataMsg == nil || dataMsg.Payload == nil {
			gc.logger.Warning("Non-data block or with no payload")
			return ""
		}
		return fmt.Sprintf("%d", dataMsg.Payload.SeqNum)
	}
	adapter := &pull.PullAdapter{
		Sndr:        gc,
		MemSvc:      gc.memFilter,
		IdExtractor: seqNumFromMsg,
		MsgCons: func(msg *protoext.SignedGossipMessage) {
			gc.DeMultiplex(msg)
		},
	}

	adapter.IngressDigFilter = func(digestMsg *proto.DataDigest) *proto.DataDigest {
		gc.RLock()
		height := gc.ledgerHeight
		gc.RUnlock()
		digests := digestMsg.Digests
		digestMsg.Digests = nil
		for i := range digests {
			seqNum, err := strconv.ParseUint(string(digests[i]), 10, 64)
			if err != nil {
				gc.logger.Warningf("Can't parse digest %s : %+v", digests[i], err)
				continue
			}
			if seqNum >= height {
				digestMsg.Digests = append(digestMsg.Digests, digests[i])
			}

		}
		return digestMsg
	}

	return pull.NewPullMediator(conf, adapter)
}
```

## 3.NewPullMediator.adapter

```go
// NewPullMediator returns a new Mediator
func NewPullMediator(config Config, adapter *PullAdapter) Mediator {
	egressDigFilter := adapter.EgressDigFilter

	acceptAllFilter := func(_ protoext.ReceivedMessage) func(string) bool {
		return func(_ string) bool {
			return true
		}
	}

	if egressDigFilter == nil {
		egressDigFilter = acceptAllFilter
	}

	p := &pullMediatorImpl{
		PullAdapter:  adapter,
		msgType2Hook: make(map[MsgType][]MessageHook),
		config:       config,
		logger:       util.GetLogger(util.PullLogger, config.ID),
		itemID2Msg:   make(map[string]*protoext.SignedGossipMessage),
	}

	p.engine = algo.NewPullEngineWithFilter(p, config.PullInterval, egressDigFilter.byContext(), config.PullEngineConfig)

	if adapter.IngressDigFilter == nil {
		// Create accept all filter
		adapter.IngressDigFilter = func(digestMsg *proto.DataDigest) *proto.DataDigest {
			return digestMsg
		}
	}
	return p

}

```

## 4.NewPullEngineWithFilter

```go
// NewPullEngineWithFilter creates an instance of a PullEngine with a certain sleep time
// between pull initiations, and uses the given filters when sending digests and responses
func NewPullEngineWithFilter(participant PullAdapter, sleepTime time.Duration, df DigestFilter,
	config PullEngineConfig) *PullEngine {
	engine := &PullEngine{
		PullAdapter:        participant,
		stopFlag:           int32(0),
		state:              util.NewSet(),
		item2owners:        make(map[string][]string),
		peers2nonces:       make(map[string]uint64),
		nonces2peers:       make(map[uint64]string),
		acceptingDigests:   int32(0),
		acceptingResponses: int32(0),
		incomingNONCES:     util.NewSet(),
		outgoingNONCES:     util.NewSet(),
		digFilter:          df,
		digestWaitTime:     config.DigestWaitTime,
		requestWaitTime:    config.RequestWaitTime,
		responseWaitTime:   config.ResponseWaitTime,
	}

	go func() {
		for !engine.toDie() {
			time.Sleep(sleepTime)
			if engine.toDie() {
				return
			}
			engine.initiatePull()
		}
	}()

	return engine
}
```

## 5.PullEngine.initiatePull

```go
func (engine *PullEngine) initiatePull() {
	engine.lock.Lock()
	defer engine.lock.Unlock()

	engine.acceptDigests()
	// 选择特定节点发送Hello消息以请求获取DataMsg消息。
    for _, peer := range engine.SelectPeers() {
		nonce := engine.newNONCE()
		engine.outgoingNONCES.Add(nonce)
		engine.nonces2peers[nonce] = peer
		engine.peers2nonces[peer] = nonce
		engine.Hello(peer, nonce)
	}

	time.AfterFunc(engine.digestWaitTime, func() {
		engine.processIncomingDigests()
	})
}
```

## 6.接受端接收到pull信息

入口如2.5中所示

### 1.gossipChannel.HandleMessage

gossip\gossip\channel\channel.go

```go
// HandleMessage processes a message sent by a remote peer
func (gc *gossipChannel) HandleMessage(msg protoext.ReceivedMessage) {
	
    ...
    
    // 4.如果接收到的是pull
	if protoext.IsPullMsg(m.GossipMessage) && protoext.GetPullMsgType(m.GossipMessage) == proto.PullMsgType_BLOCK_MSG {
		// 异常处理部分
        ...
        
		// 1.如果是更新数据信息相关
		if protoext.IsDataUpdate(m.GossipMessage) {
			// Iterate over the envelopes, and filter out blocks
			// that we already have in the blockMsgStore, or blocks that
			// are too far in the past.
			var msgs []*protoext.SignedGossipMessage
			var items []*proto.Envelope
			filteredEnvelopes := []*proto.Envelope{}
			for _, item := range m.GetDataUpdate().Data {
				gMsg, err := protoext.EnvelopeToGossipMessage(item)
  				...
				// 异常处理部分 && 判断数据的有效性
				...
                
                msgs = append(msgs, gMsg)
				items = append(items, item)
			}

			gc.Lock()
			defer gc.Unlock()

			for i, gMsg := range msgs {
				item := items[i]
				added := gc.blockMsgStore.Add(gMsg)
				if !added {
					// If this block doesn't need to be added, it means it either already exists in memory, or that it is too far in the past
					continue
				}
				filteredEnvelopes = append(filteredEnvelopes, item)
			}

			// Replace the update message with just the blocks that should be processed
			m.GetDataUpdate().Data = filteredEnvelopes
		}
		gc.blocksPuller.HandleMessage(msg)
	}
    
    // 5.如果接收到的是选举信息
    ...
}
```

### 2.pullMediatorImpl.HandleMessage

gossip\gossip\pull\pullstore.go

```go
func (p *pullMediatorImpl) HandleMessage(m protoext.ReceivedMessage) {
	
    ...

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

### 3.PullEngine.OnHello

gossip\gossip\pull\pullstore.go

```go
// OnHello notifies the engine a hello has arrived
func (engine *PullEngine) OnHello(nonce uint64, context interface{}) {
	engine.incomingNONCES.Add(nonce)

	time.AfterFunc(engine.requestWaitTime, func() {
		engine.incomingNONCES.Remove(nonce)
	})

	a := engine.state.ToArray()
	var digest []string
	filter := engine.digFilter(context)
	for _, item := range a {
		dig := item.(string)
		if !filter(dig) {
			continue
		}
		digest = append(digest, dig)
	}
	if len(digest) == 0 {
		return
	}
	engine.SendDigest(digest, nonce, context)
}
```

### 4.pullMediatorImpl.SendDigest

gossip\gossip\pull\pullstore.go

```go
// SendDigest sends a digest to a remote PullEngine.
// The context parameter specifies the remote engine to send to.
func (p *pullMediatorImpl) SendDigest(digest []string, nonce uint64, context interface{}) {
	digMsg := &proto.GossipMessage{
		Channel: p.config.Channel,
		Tag:     p.config.Tag,
		Nonce:   0,
		Content: &proto.GossipMessage_DataDig{
			DataDig: &proto.DataDigest{
				MsgType: p.config.MsgType,
				Nonce:   nonce,
				Digests: util.StringsToBytes(digest),
			},
		},
	}
	remotePeer := context.(protoext.ReceivedMessage).GetConnectionInfo()
	if p.logger.IsEnabledFor(zapcore.DebugLevel) {
		p.logger.Debug("Sending", p.config.MsgType, "digest:", formattedDigests(digMsg.GetDataDig()), "to", remotePeer)
	}

	context.(protoext.ReceivedMessage).Respond(digMsg)
}
```

