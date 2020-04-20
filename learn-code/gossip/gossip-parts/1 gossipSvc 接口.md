# 1 gossipSvc 接口

gossip\service\gossip_service.go

```go
// gossipSvc is the interface of the gossip component.
type gossipSvc interface {
	// SelfMembershipInfo returns the peer's membership information
	SelfMembershipInfo() discovery.NetworkMember

	// SelfChannelInfo returns the peer's latest StateInfo message of a given channel
	SelfChannelInfo(common.ChannelID) *protoext.SignedGossipMessage

	// Send sends a message to remote peers
	Send(msg *gproto.GossipMessage, peers ...*comm.RemotePeer)

	// SendByCriteria sends a given message to all peers that match the given SendCriteria
	SendByCriteria(*protoext.SignedGossipMessage, gossip.SendCriteria) error

	// GetPeers returns the NetworkMembers considered alive
	Peers() []discovery.NetworkMember

	// PeersOfChannel returns the NetworkMembers considered alive
	// and also subscribed to the channel given
	PeersOfChannel(common.ChannelID) []discovery.NetworkMember

	// UpdateMetadata updates the self metadata of the discovery layer
	// the peer publishes to other peers
	UpdateMetadata(metadata []byte)

	// UpdateLedgerHeight updates the ledger height the peer
	// publishes to other peers in the channel
	UpdateLedgerHeight(height uint64, channelID common.ChannelID)

	// UpdateChaincodes updates the chaincodes the peer publishes
	// to other peers in the channel
	UpdateChaincodes(chaincode []*gproto.Chaincode, channelID common.ChannelID)

	// Gossip sends a message to other peers to the network
	Gossip(msg *gproto.GossipMessage)

	// PeerFilter receives a SubChannelSelectionCriteria and returns a RoutingFilter that selects
	// only peer identities that match the given criteria, and that they published their channel participation
	PeerFilter(channel common.ChannelID, messagePredicate api.SubChannelSelectionCriteria) (filter.RoutingFilter, error)

	// Accept returns a dedicated read-only channel for messages sent by other nodes that match a certain predicate.
	// If passThrough is false, the messages are processed by the gossip layer beforehand.
	// If passThrough is true, the gossip layer doesn't intervene and the messages
	// can be used to send a reply back to the sender
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *gproto.GossipMessage, <-chan protoext.ReceivedMessage)

	// JoinChan makes the Gossip instance join a channel
	JoinChan(joinMsg api.JoinChannelMessage, channelID common.ChannelID)

	// LeaveChan makes the Gossip instance leave a channel.
	// It still disseminates stateInfo message, but doesn't participate
	// in block pulling anymore, and can't return anymore a list of peers
	// in the channel.
	LeaveChan(channelID common.ChannelID)

	// SuspectPeers makes the gossip instance validate identities of suspected peers, and close
	// any connections to peers with identities that are found invalid
	SuspectPeers(s api.PeerSuspector)

	// IdentityInfo returns information known peer identities
	IdentityInfo() api.PeerIdentitySet

	// IsInMyOrg checks whether a network member is in this peer's org
	IsInMyOrg(member discovery.NetworkMember) bool

	// Stop stops the gossip component
	Stop()
}
```

## 生成Gossip服务模块实例

```go
// New creates a gossip instance attached to a gRPC server
func New(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) *Node {

    // 创建一个gossip node instance
	g := &Node{
		selfOrg:               sa.OrgByPeerIdentity(selfIdentity),
		secAdvisor:            sa,
		selfIdentity:          selfIdentity,
		presumedDead:          make(chan common.PKIidType, presumedDeadChanSize),
		disc:                  nil,
		mcs:                   mcs,
		conf:                  conf,
		ChannelDeMultiplexer:  comm.NewChannelDemultiplexer(),
		logger:                lgr,
		toDieChan:             make(chan struct{}),
		stopFlag:              int32(0),
		stopSignal:            &sync.WaitGroup{},
		includeIdentityPeriod: time.Now().Add(conf.PublishCertPeriod),
		gossipMetrics:         gossipMetrics,
	}
    
    // 创建message store：可以将一个信息存到节点内的缓冲区里，
    // 具有判断消息是否有效的作用，也可以把无效message从缓冲区移出来
	g.stateInfoMsgStore = g.newStateInfoMsgStore()

    // idMapper里面保存的是pkiID和cert的映射集合
    /* 
    ·idMapper：节点身份管理器（identityMapperImpl类型），负责维护pkiID2Cert字典，保存节点的PKI-ID及其节点身份信息（storedIdentity类型，封装了PKI-ID、身份证书以及过期时间等参数）。
    其中，节点的PKI-ID是调用mcs.GetPK-IidOfCert()方法根据所属组织的MSP ID与节点身份证书信息计算的哈希值（SHA-256哈希算法）。
    同时，执行goidMapper.periodicalPurgeUnusedIdentities()，负责周期性地从pkiID2Cert字典中删除过期、被撤销以及无效的节点身份信息。
	*/
	g.idMapper = identity.NewIdentityMapper(mcs, selfIdentity, func(pkiID common.PKIidType, identity api.PeerIdentityType) {
		g.comm.CloseConn(&comm.RemotePeer{PKIID: pkiID})
		g.certPuller.Remove(string(pkiID))
	}, sa)

	commConfig := comm.CommConfig{
		DialTimeout:  conf.DialTimeout,
		ConnTimeout:  conf.ConnTimeout,
		RecvBuffSize: conf.RecvBuffSize,
		SendBuffSize: conf.SendBuffSize,
	}
    // 通过g.comm可以与一个给定的grpc server连接起来。Gossip服务实例通信模块，向其他模块提供通信接口以实现Peer节点之间的通信。
	g.comm, err = comm.NewCommInstance(s, conf.TLSCerts, g.idMapper, selfIdentity, secureDialOpts, sa,
		gossipMetrics.CommMetrics, commConfig)

    // 用于处理通道消息的消息处理模块。管理了一个节点所连接的所有通道，可以完成消息处理。
    /* 
    其中，GossipChannel通道对象可分别提供blockMsgStore、stateInfoMsgStore与leaderMsgStore等消息存储对象，用于处理通道内传播的消息，包括DataMsg类型数据消息、状态类消息、LeadershipMsg类型主节点选举消息等。
	*/
	g.chanState = newChannelState(g)
    
    // emitter主要用在push/fowarding阶段，可以批量的发送信息。
    /*
    消息会添加到batchingEmitter中，然后定期的分多批次被发送出去，如果消息过多到达了存储上限，也会触发消息的发送机制。
    */
	g.emitter = newBatchingEmitter(conf.PropagateIterations,
		conf.MaxPropagationBurstSize, conf.MaxPropagationBurstLatency,
		g.sendGossipBatch)

    // discoveryAdapter类型的适配器模块，用于为discovery模块接收、过滤与转发消息。
    /*
    该适配器模块封装了comm通信模块用于底层消息的通信，并定义了discovery模块的gossipFunc()分发数据函数和forwardFunc()转发数据函数，它们都将消息添加到emitter模块中请求发送。
    其中，forwardFunc()函数还定义了消息过滤器用于过滤节点。同时，discAdapter适配器模块还提供了incChan通道，用于接收AliveMsg类型、MemReq类型与MemRes类型消息。
	*/
	g.discAdapter = g.newDiscoveryAdapter()
    
	// discoverySecurityAdapter类型的安全适配器模块，封装了Security-Advisor安全辅助组件、idMapper模块、MessageCryptoService消息加密服务组件、通信模块、日志模块、Peer节点身份证书信息等，可提供身份管理、消息加密、签名与验签等安全服务。
    g.disSecAdap = g.newDiscoverySecurityAdapter()

	discoveryConfig := discovery.DiscoveryConfig{
		AliveTimeInterval:            conf.AliveTimeInterval,
		AliveExpirationTimeout:       conf.AliveExpirationTimeout,
		AliveExpirationCheckInterval: conf.AliveExpirationCheckInterval,
		ReconnectInterval:            conf.ReconnectInterval,
		BootstrapPeers:               conf.BootstrapPeers,
	}
    // 负责维护通道中其他Peer节点的状态与成员关系，
	g.disc = discovery.NewDiscoveryService(g.selfNetworkMember(), g.discAdapter, g.disSecAdap, g.disclosurePolicy,
		discoveryConfig)
	g.logger.Infof("Creating gossip service with self membership of %s", g.selfNetworkMember())

    // 周期性地发送Pull类节点身份消息，请求拉取PeerIdentity类型节点身份消息，并通过自身的engine模块管理存储消息摘要
	g.certPuller = g.createCertStorePuller()
    // 通过certPuller模块管理与存储PeerIdentity类型节点身份消息，预处理Pull 类节点身份消息，并交由certPuller模块具体处理。
	g.certStore = newCertStore(g.certPuller, g.idMapper, selfIdentity, mcs)

	if g.conf.ExternalEndpoint == "" {
		g.logger.Warning("External endpoint is empty, peer will not be accessible outside of its organization")
	}
	// Adding delta for handlePresumedDead and
	// acceptMessages goRoutines to block on Wait
	g.stopSignal.Add(2)
	go g.start()
	go g.connect2BootstrapPeers()

	return g
}
```

