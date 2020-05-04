# 7 更新通道状态信息

StateInfo消息包含了节点账本的区块链信息（包括最新的账本高度等），其他节点的StateInfo消息都被缓存到本地节点上指定通道的GossipChannel通道对象中，并保存在stateInfoMsgStore消息存储对象中。

StateInfo消息用于提供给本地节点：

1. 获取通道上其他节点的最新账本高度
2. 计算账本区块链的高度差以判断其与其他节点的账本数据差异
3. 标识出本地账本的缺失数据（包括区块数据与隐私数据）范围
4. 通过反熵算法的数据同步机制从其他节点拉取数据
5. 更新到本地账本。

## 0.入口

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
    // 1.读取账本中保存的通道配置chanConf，即与channelConfigKey对应的值
	chanConf, err := retrievePersistedChannelConfig(l)
	// 2.创建新的通道配置实体对象Bundle结构
	bundle, err := channelconfig.NewBundle(cid, chanConf, p.CryptoProvider)
	// 3.检查通道配置对象是否支持指定的功能特性
	capabilitiesSupportedOrPanic(bundle)
	// 4.检查通道配置对象上策略管理器的规范性
	channelconfig.LogSanityChecks(bundle)
	// 5.
	gossipEventer := p.GossipService.NewConfigEventer()
	// 6.回调函数gossipCallbackWrapper()：根据当前新链的通道配置创建与更新对应的GossipChannel通道对象，用于过滤消息并控制其在通道内的传播。
	gossipCallbackWrapper := func(bundle *channelconfig.Bundle) {
		// Application应用通道配置
        ac, ok := bundle.ApplicationConfig()
        // 更新当前通道链结构的通道配置。
		gossipEventer.ProcessConfigUpdate(&gossipSupport{
			Validator:   bundle.ConfigtxValidator(), 	// 交易验证器
			Application: ac, 							// Application应用通道配置
			Channel:     bundle.ChannelConfig(), 		// 通道配置
		})
		p.GossipService.SuspectPeers(func(identity api.PeerIdentityType) bool {
			return true
		})
	}
	// 6.回调函数trustedRootsCallbackWrapper()：更新当前节点上默认的gRPC服务器中配置的信任根CA证书列表。
	trustedRootsCallbackWrapper := func(bundle *channelconfig.Bundle) {
		p.updateTrustedRoots(bundle)
	}
	// 6.回调函数mspCallback：设置全局变量mspMap字典中当前新链上的MSP组件管理器对象，将其设置为当前新链通道配置中的MSP组件管理器，以用于管理该通道配置上所有的MSP组件集合。
	mspCallback := func(bundle *channelconfig.Bundle) {
		mspmgmt.XXXSetMSPManager(cid, bundle.MSPManager())
	}

	osLogger := flogging.MustGetLogger("peer.orderers")
	namedOSLogger := osLogger.With("channel", cid)
    ordererSource := orderers.NewConnectionSource(namedOSLogger, p.OrdererEndpointOverrides)
	// 6.回调函数
	ordererSourceCallback := func(bundle *channelconfig.Bundle) {
		globalAddresses := bundle.ChannelConfig().OrdererAddresses()
		orgAddresses := map[string]orderers.OrdererOrg{}
		if ordererConfig, ok := bundle.OrdererConfig(); ok {
			for orgName, org := range ordererConfig.Organizations() {
				certs := [][]byte{}
				for _, root := range org.MSP().GetTLSRootCerts() {
					certs = append(certs, root)
				}

				for _, intermediate := range org.MSP().GetTLSIntermediateCerts() {
					certs = append(certs, intermediate)
				}

				orgAddresses[orgName] = orderers.OrdererOrg{
					Addresses: org.Endpoints(),
					RootCerts: certs,
				}
			}
		}
		ordererSource.Update(globalAddresses, orgAddresses)
	}
	
    // 7.创建channel
	channel := &Channel{
		ledger:         l,
		resources:      bundle,
		cryptoProvider: p.CryptoProvider,
	}
	
    // 8.更新channel配置：构造并设置新通道资源配置的BundleSource对象
	channel.bundleSource = channelconfig.NewBundleSource(
		bundle,							// 通道资源配置实体Bundle对象
		ordererSourceCallback,			// 回调函数
		gossipCallbackWrapper,			// 回调函数
		trustedRootsCallbackWrapper,	// 回调函数
		mspCallback,					// 回调函数
		channel.bundleUpdate,			// 
	)
	// 9.创建提交器
	committer := committer.NewLedgerCommitter(l)
    // 10.创建Committer模块的交易验证器，用于执行VSCC以验证交易背书策略的有效性
	validator := &txvalidator.ValidationRouter{
		...
	}

	// 11.创建指定通道上的transient隐私数据存储对象），用于临时缓存Endorser背书节点通过Gossip消息协议分发的隐私数据
	store, err := p.openStore(bundle.ConfigtxValidator().ChannelID())
	channel.store = store
	
    // 12.创建隐私数据集合存储对象，用于从通道账本中获取指定链码的隐私数据集合配置信息，封装了隐私数据的访问权限策略。
	simpleCollectionStore := privdata.NewSimpleCollectionStore(l, deployedCCInfoProvider)
    
    // 13.初始化指定通道上的Gossip服务模块
	// 若是Leader主节点，则从Orderer节点获取通道账本区块，否则，从组织内其他节点接收数据
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
	
    // 1.
	gc.memFilter = &membershipFilter{adapter: gc.Adapter, gossipChannel: gc}
    // 2.
	comparator := protoext.NewGossipMessageComparator(adapter.GetConf().MaxBlockCountToStore)
	// 3.完成pull block的工作
	gc.blocksPuller = gc.createBlockPuller()
	// 4.定期删除block msg store
	gc.blockMsgStore = msgstore.NewMessageStoreExpirable(comparator, func(m interface{}) {
		gc.logger.Debugf("Removing %s from the message store", seqNumFromMsg(m))
		gc.blocksPuller.Remove(seqNumFromMsg(m))
	}, gc.GetConf().BlockExpirationInterval, nil, nil, func(m interface{}) {
		gc.logger.Debugf("Removing %s from the message store", seqNumFromMsg(m))
		gc.blocksPuller.Remove(seqNumFromMsg(m))
	})
	// 5.
	hashPeerExpiredInMembership := func(o interface{}) bool {
		pkiID := o.(*protoext.SignedGossipMessage).GetStateInfo().PkiId
		return gc.Lookup(pkiID) == nil
	}
    // 6.
	verifyStateInfoMsg := func(msg *protoext.SignedGossipMessage, orgs ...api.OrgIdentityType) bool {
		si := msg.GetStateInfo()
		// No point in verifying ourselves
		if bytes.Equal(gc.pkiID, si.PkiId) {
			return true
		}
		peerIdentity := adapter.GetIdentityByPKIID(si.PkiId)
		if len(peerIdentity) == 0 {
			gc.logger.Warning("Identity for peer", si.PkiId, "doesn't exist")
			return false
		}
		isOrgInChan := func(org api.OrgIdentityType) bool {
			if len(orgs) == 0 {
				if !gc.IsOrgInChannel(org) {
					return false
				}
			} else {
				found := false
				for _, chanMember := range orgs {
					if bytes.Equal(chanMember, org) {
						found = true
						break
					}
				}
				if !found {
					return false
				}
			}
			return true
		}

		org := gc.GetOrgOfPeer(si.PkiId)
		if !isOrgInChan(org) {
			gc.logger.Warning("peer", peerIdentity, "'s organization(", string(org), ") isn't in the channel", string(channelID))
			return false
		}
		if err := gc.mcs.VerifyByChannel(channelID, peerIdentity, msg.Signature, msg.Payload); err != nil {
			gc.logger.Warningf("Peer %v isn't eligible for channel %s : %+v", peerIdentity, string(channelID), err)
			return false
		}
		return true
	}
    // 7.
	gc.stateInfoMsgStore = newStateInfoCache(gc.GetConf().StateInfoCacheSweepInterval, hashPeerExpiredInMembership, verifyStateInfoMsg)
	// 8.
	ttl := adapter.GetConf().MsgExpirationTimeout
	pol := protoext.NewGossipMessageComparator(0)
	gc.leaderMsgStore = msgstore.NewMessageStoreExpirable(pol, msgstore.Noop, ttl, nil, nil, nil)
	// 9.
	gc.ConfigureChannel(joinMsg)

    
	// 10.Periodically publish state info
	go gc.periodicalInvocation(gc.publishStateInfo, gc.stateInfoPublishScheduler.C)
	// 11.Periodically request state info
	go gc.periodicalInvocation(gc.requestStateInfo, gc.stateInfoRequestScheduler.C)

    // 12.
	ticker := time.NewTicker(gc.GetConf().TimeForMembershipTracker)
	gc.membershipTracker = &membershipTracker{
		getPeersToTrack: gc.GetPeers,
		report:          gc.reportMembershipChanges,
		stopChan:        make(chan struct{}, 1),
		tickerChannel:   ticker.C,
		metrics:         metrics,
		chainID:         channelID,
	}
    // 13.
	go gc.membershipTracker.trackMembershipChanges()
	
    // 14.
    return gc
}
```

### 9.periodicalInvocation

gossip\gossip\channel\channel.go

```go
func (gc *gossipChannel) periodicalInvocation(fn func(), c <-chan time.Time) {
	for {
		select {
        // 1.时间到了就invoke一次
		case <-c:
			fn()
		case <-gc.stopChan:
			return
		}
	}
}
```



## 1.发送端：定期发布通道状态信息

### 1.gossipChannel.publishStateInfo

gossip\gossip\channel\channel.go

```go
func (gc *gossipChannel) publishStateInfo() {
    // 1.检查Gossip状态信息消息更新标志位
	if atomic.LoadInt32(&gc.shouldGossipStateInfo) == int32(0) {
		return
	}
    // 2.获取通道状态信息消息
	gc.RLock()
	stateInfoMsg := gc.stateInfoMsg
	gc.RUnlock()
    // 3.发送状态信息消息
	gc.Gossip(stateInfoMsg)
	// 4.检查是否存在组织内的存活成员节点
    if len(gc.GetMembership()) > 0 {
		atomic.StoreInt32(&gc.shouldGossipStateInfo, int32(0))
	}
}

```



## 2.发送端：定期请求通道状态信息

### 1.gossipChannel.requestStateInfo

gossip\gossip\channel\channel.go

```go
func (gc *gossipChannel) requestStateInfo() {
    // 1.创建状态信息请求消息，构造一个SignedGossipMessage，类型为state info request
	req, err := gc.createStateInfoRequest()
	// 2.获取发送节点列表
	endpoints := filter.SelectPeers(gc.GetConf().PullPeerNum, gc.GetMembership(), gc.IsMemberInChan)
    // 3.向endpoint发送state info request，请求获取其他节点上的通道StateInfo消息
	gc.Send(req, endpoints...)
}
```

## 3.接收端：接收通道状态信息&&请求

### 0.入口

#### 1.New gossip node

gossip\gossip\gossip_impl.go

在gossip node启动时会接收消息

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

#### 2.Node.start

```go
func (g *Node) start() {
	go g.syncDiscovery()
	go g.handlePresumedDead()

	...
	incMsgs := g.comm.Accept(msgSelector)

	go g.acceptMessages(incMsgs)

	g.logger.Info("Gossip instance", g.conf.ID, "started")
}
```

#### 3.Node.acceptMessages

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

#### 4.Node.handleMessage

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
			// If we're not in the channel, we should still forward to peers of our org
			// in case it's a StateInfo message
			if g.IsInMyOrg(discovery.NetworkMember{PKIid: m.GetConnectionInfo().ID}) && protoext.IsStateInfoMsg(msg.GossipMessage) {
				if g.stateInfoMsgStore.Add(msg) {
					g.emitter.Add(&emittedGossipMessage{
						SignedGossipMessage: msg,
						filter:              m.GetConnectionInfo().ID.IsNotSameFilter,
					})
				}
			}
			...
		} else {
			if protoext.IsLeadershipMsg(m.GetGossipMessage().GossipMessage) {
				...
			}
			gc.HandleMessage(m)
		}
		return
	}

	if selectOnlyDiscoveryMessages(m) {
		...
		g.forwardDiscoveryMsg(m)
	}

	if protoext.IsPullMsg(msg.GossipMessage) && protoext.GetPullMsgType(msg.GossipMessage) == pg.PullMsgType_IDENTITY_MSG {
		g.certStore.handleMessage(m)
	}
}
```

#### 5.gossipChannel.HandleMessage

```go
// HandleMessage processes a message sent by a remote peer
func (gc *gossipChannel) HandleMessage(msg protoext.ReceivedMessage) {
	
	...
    // 1.接受到了state request
	if protoext.IsStateInfoPullRequestMsg(m.GossipMessage) {
        // 1.创建StateInfoSnapshot类型的状态信息快照消息
        // 2.调用msg.Respond()方法对消息签名并回复给请求节点。
		msg.Respond(gc.createStateInfoSnapshot(orgID))
		return
	}
	
    // 2.接收到了对state request的回复
	if protoext.IsStateInfoSnapshot(m.GossipMessage) {
		// 1.遍历并验证该消息包含的StateInfo消息。
        // 2.如果通过了验证，则添加到本地GossipChannel通道对象上的消息存储对象stateInfoMsg-Store中。
        gc.handleStateInfSnapshot(m.GossipMessage, msg.GetConnectionInfo().ID)
		return
	}

	if protoext.IsDataMsg(m.GossipMessage) || protoext.IsStateInfoMsg(m.GossipMessage) {
		added := false

		if protoext.IsDataMsg(m.GossipMessage) {
			...
		} else { // StateInfoMsg verification should be handled in a layer above
			//  since we don't have access to the id mapper here
			// 3.接收到了stateInfo Msg，就添加到stateInfoMsgStore中
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

	if protoext.IsPullMsg(m.GossipMessage) && protoext.GetPullMsgType(m.GossipMessage) == proto.PullMsgType_BLOCK_MSG {
		...
	}

	if protoext.IsLeadershipMsg(m.GossipMessage) {
		...
	}
}
```

### 1.gossipChannel.createStateInfoSnapshot

接收请求端：接收到了发送请求端的state request

```go
func (gc *gossipChannel) createStateInfoSnapshot(requestersOrg api.OrgIdentityType) *proto.GossipMessage {
	
    // 获取GossipChannel通道对象上stateInfoMsgStore对象所保存的StateInfo消息列表
	rawElements := gc.stateInfoMsgStore.Get()
	elements := []*proto.Envelope{}
    // 遍历该列表中的每个消息，筛选出符合要求的StateInfo消息
	for _, rawEl := range rawElements {
		msg := rawEl.(*protoext.SignedGossipMessage)
		orgOfCurrentMsg := gc.GetOrgOfPeer(msg.GetStateInfo().PkiId)
		// If we're in the same org as the requester, or the message belongs to a foreign org
		// don't do any filtering
		if sameOrg || !bytes.Equal(orgOfCurrentMsg, gc.selfOrg) {
			elements = append(elements, msg.Envelope)
			continue
		}
		// Else, the requester is in a different org, so disclose only StateInfo messages that their
		// corresponding AliveMessages have external endpoints
		if netMember := gc.Lookup(msg.GetStateInfo().PkiId); netMember == nil || netMember.Endpoint == "" {
			continue
		}
        // 将其Envelope字段对象添加到elements消息列表（[]*proto.Envelope）
		elements = append(elements, msg.Envelope)
	}

    // 封装为StateInfoSnapshot消息
	return &proto.GossipMessage{
		Channel: gc.chainID,
		Tag:     proto.GossipMessage_CHAN_OR_ORG,
		Nonce:   0,
		Content: &proto.GossipMessage_StateSnapshot{
			StateSnapshot: &proto.StateInfoSnapshot{
				Elements: elements,
			},
		},
	}
}
```

### 2.gossipChannel.handleStateInfSnapshot

发送请求端：收到了接收端的发过来的state snapshot

```go
func (gc *gossipChannel) handleStateInfSnapshot(m *proto.GossipMessage, sender common.PKIidType) {
	chanName := string(gc.chainID)
	for _, envelope := range m.GetStateSnapshot().Elements {
        // 1.拿到state Info
		stateInf, err := protoext.EnvelopeToGossipMessage(envelope)
		// 2.判断有效性
		if !protoext.IsStateInfoMsg(stateInf.GossipMessage) {
			return
		}
        // 3.拿到state info的数据字段
		si := stateInf.GetStateInfo()
      	// 4.拿到org id
		orgID := gc.GetOrgOfPeer(si.PkiId)
		if !gc.IsOrgInChannel(orgID) {
			return
		}
		// 5.
		expectedMAC := GenerateMAC(si.PkiId, gc.chainID)
		if !bytes.Equal(si.Channel_MAC, expectedMAC) {
			gc.logger.Warning("Channel", chanName, ": StateInfo message", stateInf,
				", has an invalid MAC. Expected", expectedMAC, ", got", si.Channel_MAC, ", sent from", sender)
			return
        }
        // 6.判断有效性
		err = gc.ValidateStateInfoMessage(stateInf)
		// 7.
		if gc.Lookup(si.PkiId) == nil {
			// Skip StateInfo messages that belong to peers
			// that have been expired
			continue
		}
		// 8.添加到state info message store中
		gc.stateInfoMsgStore.Add(stateInf)
	}
}
```

