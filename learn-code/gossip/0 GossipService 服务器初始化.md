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

