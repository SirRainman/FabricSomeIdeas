参考文献：

1. [源码分析](https://blog.csdn.net/idsuf698987/article/details/77898724)
2. [选举算法](https://zhuanlan.zhihu.com/p/27989809)

# Gossip源码结构

1. fabric/gossip

   gossip/
   |-- api 	消息加密服务接口，与peer衔接的接口
   |-- comm 	实现点对点的通信
   |-- common 	公用函数，定义，结构体
   |-- discovery	发现模块，用于发现网络中的有效结点，供gossip调用
   |-- election	   选举模块，用于选举领导peer，供service调用
   |-- filter		过滤模块，用于选择一个信息是否应该发送给一个网络成员
   |-- gossip	定义了gossip接口，实现goosip服务
   |   |-- algo	算法，PullEngine对象，供pull调用
   |   |-- channel	GossipChannel对象，供channelState实例管理
   |   |-- msgstore	MessageStore对象，供channel调用
   |    '-- pull	Mediator对象，供channel调用
   |-- identity	身份映射模块，用于PKI-ID与identity之间的映射，供service调用
   |-- metrics
   |-- privdata
   |-- protoext
   |-- service		gossip服务器，封装了gossip服务，状态，分发模块等，与核心代码衔接
   |-- state			state模块，供service调用
   `-- util			   公用工具文件夹，提供工具函数



还有其他部分没有标出来：[fabric1.0的源码结构](https://blog.csdn.net/idsuf698987/article/details/77898724)

---

# 0 "GossipService 服务器"初始化

fabric gossip主要是为了peer节点之间数据传播使用的，所以初始化是和peer节点一起初始化的。

peer节点初始化入口：fabric\internal\peer\node\start.go---serve()

```go
func serve(args []string) error {
	
    ...

	gossipService, err := initGossipService(
		...
	)
	if err != nil {
		return errors.WithMessage(err, "failed to initialize gossip service")
	}
	defer gossipService.Stop()
    
    ...
    
}
```

其中initGossipService()的具体步骤为：

1. 开启tls
2. 初始化消息加密服务
3. 初始化security advisor
4. 初始化gossip相关的数据结构

```go
// initGossipService will initialize the gossip service by:
// 1. Enable TLS if configured;
// 2. Init the message crypto service;
// 3. Init the security advisor;
// 4. Init gossip related struct.
func initGossipService(...) (*gossipservice.GossipService, error) {
    
    ...
    
	return gossipservice.New(
		...
	)
}
```

GossipService的具体结构为：gossip\service\gossip_service.go

```go
type GossipService struct {
	gossipSvc
	privateHandlers map[string]privateHandler
	chains          map[string]state.GossipStateProvider
	leaderElection  map[string]election.LeaderElectionService
	deliveryService map[string]deliverservice.DeliverService
	deliveryFactory DeliveryServiceFactory
	lock            sync.RWMutex
	mcs             api.MessageCryptoService
	peerIdentity    []byte
	secAdv          api.SecurityAdvisor
	metrics         *gossipmetrics.GossipMetrics
	serviceConfig   *ServiceConfig
	privdataConfig  *gossipprivdata.PrivdataConfig
}
```



---

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

# 2 privateHandler 结构体

gossip\service\gossip_service.go

```go
type privateHandler struct {
	support     Support
	coordinator gossipprivdata.Coordinator
	distributor gossipprivdata.PvtDataDistributor
	reconciler  gossipprivdata.PvtDataReconciler
}
```

# 3 GossipStateProvider 接口

gossip\state\state.go

```go
// GossipStateProvider is the interface to acquire sequences of the ledger blocks
// capable to full fill missing blocks by running state replication and
// sending request to get missing block to other nodes
type GossipStateProvider interface {
	AddPayload(payload *proto.Payload) error

	// Stop terminates state transfer object
	Stop()
}
```

# 4 LeaderElectionService 接口

gossip\election\election.go

```
// LeaderElectionService is the object that runs the leader election algorithm
type LeaderElectionService interface {
   // IsLeader returns whether this peer is a leader or not
   IsLeader() bool

   // Stop stops the LeaderElectionService
   Stop()

   // Yield relinquishes the leadership until a new leader is elected,
   // or a timeout expires
   Yield()
}
```



# 5 DeliveryServiceFactory 接口

```go
// DeliveryServiceFactory factory to create and initialize delivery service instance
type DeliveryServiceFactory interface {
   // Returns an instance of delivery client
   Service(g GossipServiceAdapter, ordererSource *orderers.ConnectionSource, msc api.MessageCryptoService, isStaticLead bool) deliverservice.DeliverService
}
```

# 6 MessageCryptoService 接口

gossip\api\crypto.go

```go
// MessageCryptoService is the contract between the gossip component and the
// peer's cryptographic layer and is used by the gossip component to verify,
// and authenticate remote peers and data they send, as well as to verify
// received blocks from the ordering service.
type MessageCryptoService interface {
   // GetPKIidOfCert returns the PKI-ID of a peer's identity
   // If any error occurs, the method return nil
   // This method does not validate peerIdentity.
   // This validation is supposed to be done appropriately during the execution flow.
   GetPKIidOfCert(peerIdentity PeerIdentityType) common.PKIidType

   // VerifyBlock returns nil if the block is properly signed, and the claimed seqNum is the
   // sequence number that the block's header contains.
   // else returns error
   VerifyBlock(channelID common.ChannelID, seqNum uint64, block *cb.Block) error

   // Sign signs msg with this peer's signing key and outputs
   // the signature if no error occurred.
   Sign(msg []byte) ([]byte, error)

   // Verify checks that signature is a valid signature of message under a peer's verification key.
   // If the verification succeeded, Verify returns nil meaning no error occurred.
   // If peerIdentity is nil, then the verification fails.
   Verify(peerIdentity PeerIdentityType, signature, message []byte) error

   // VerifyByChannel checks that signature is a valid signature of message
   // under a peer's verification key, but also in the context of a specific channel.
   // If the verification succeeded, Verify returns nil meaning no error occurred.
   // If peerIdentity is nil, then the verification fails.
   VerifyByChannel(channelID common.ChannelID, peerIdentity PeerIdentityType, signature, message []byte) error

   // ValidateIdentity validates the identity of a remote peer.
   // If the identity is invalid, revoked, expired it returns an error.
   // Else, returns nil
   ValidateIdentity(peerIdentity PeerIdentityType) error

   // Expiration returns:
   // - The time when the identity expires, nil
   //   In case it can expire
   // - A zero value time.Time, nil
   //   in case it cannot expire
   // - A zero value, error in case it cannot be
   //   determined if the identity can expire or not
   Expiration(peerIdentity PeerIdentityType) (time.Time, error)
}
```

# 7 SecurityAdvisor 接口

gossip\api\channel.go

```go
// SecurityAdvisor defines an external auxiliary object
// that provides security and identity related capabilities
type SecurityAdvisor interface {
   // OrgByPeerIdentity returns the OrgIdentityType
   // of a given peer identity.
   // If any error occurs, nil is returned.
   // This method does not validate peerIdentity.
   // This validation is supposed to be done appropriately during the execution flow.
   OrgByPeerIdentity(PeerIdentityType) OrgIdentityType
}
```

# 8 GossipMetrics 结构体 

gossip\metrics\metrics.go

```go
// GossipMetrics encapsulates all of gossip metrics
type GossipMetrics struct {
   StateMetrics      *StateMetrics
   ElectionMetrics   *ElectionMetrics
   CommMetrics       *CommMetrics
   MembershipMetrics *MembershipMetrics
   PrivdataMetrics   *PrivdataMetrics
}
```

# 9 ServiceConfig 结构体

gossip\service\config.go

```go
// ServiceConfig is the config struct for gossip services
type ServiceConfig struct {
	// PeerTLSEnabled enables/disables Peer TLS.
	PeerTLSEnabled bool
	// Endpoint which overrides the endpoint the peer publishes to peers in its organization.
	Endpoint              string
	NonBlockingCommitMode bool
	// UseLeaderElection defines whenever peer will initialize dynamic algorithm for "leader" selection.
	UseLeaderElection bool
	// OrgLeader statically defines peer to be an organization "leader".
	OrgLeader bool
	// ElectionStartupGracePeriod is the longest time peer waits for stable membership during leader
	// election startup (unit: second).
	ElectionStartupGracePeriod time.Duration
	// ElectionMembershipSampleInterval is the time interval for gossip membership samples to check its stability (unit: second).
	ElectionMembershipSampleInterval time.Duration
	// ElectionLeaderAliveThreshold is the time passes since last declaration message before peer decides to
	// perform leader election (unit: second).
	ElectionLeaderAliveThreshold time.Duration
	// ElectionLeaderElectionDuration is the time passes since last declaration message before peer decides to perform
	// leader election (unit: second).
	ElectionLeaderElectionDuration time.Duration
	// PvtDataPullRetryThreshold determines the maximum duration of time private data corresponding for
	// a given block.
	PvtDataPullRetryThreshold time.Duration
	// PvtDataPushAckTimeout is the maximum time to wait for the acknoledgement from each peer at private
	// data push at endorsement time.
	PvtDataPushAckTimeout time.Duration
	// BtlPullMargin is the block to live pulling margin, used as a buffer to prevent peer from trying to pull private data
	// from peers that is soon to be purged in next N blocks.
	BtlPullMargin uint64
	// TransientstoreMaxBlockRetention defines the maximum difference between the current ledger's height upon commit,
	// and the private data residing inside the transient store that is guaranteed not to be purged.
	TransientstoreMaxBlockRetention uint64
	// SkipPullingInvalidTransactionsDuringCommit is a flag that indicates whether pulling of invalid
	// transaction's private data from other peers need to be skipped during the commit time and pulled
	// only through reconciler.
	SkipPullingInvalidTransactionsDuringCommit bool
}
```

# 10 PrivdataConfig 结构体

gossip\privdata\config.go

```go
// PrivdataConfig is the struct that defines the Gossip Privdata configurations.
type PrivdataConfig struct {
	// ReconcileSleepInterval determines the time reconciler sleeps from end of an interation until the beginning of the next
	// reconciliation iteration.
	ReconcileSleepInterval time.Duration
	// ReconcileBatchSize determines the maximum batch size of missing private data that will be reconciled in a single iteration.
	ReconcileBatchSize int
	// ReconciliationEnabled is a flag that indicates whether private data reconciliation is enabled or not.
	ReconciliationEnabled bool
	// ImplicitCollectionDisseminationPolicy specifies the dissemination  policy for the peer's own implicit collection.
	ImplicitCollDisseminationPolicy ImplicitCollectionDisseminationPolicy
}
```

