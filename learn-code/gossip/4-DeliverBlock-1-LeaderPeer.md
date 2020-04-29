# 1 Leader向Orderer节点请求区块

## 1.入口

入口主要有两个，这里讨论的是peer节点启动时的过程

1. peer节点加入一个channel时
2. peer节点启动时初始化channel



1. serve()

   internal\peer\node\start.go

   peer节点初始化所加入的所有的channel

```go
func serve() {
    // 1.this brings up all the channels
	peerInstance.Initialize(
		func(cid string) {
		
		},
        ...
	)
}
```

2. Initialize

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
	
	// 1.拿到peer节点上所有的账本
	ledgerIds, err := p.LedgerMgr.GetLedgerIDs()
	for _, cid := range ledgerIds {
		ledger, err := p.LedgerMgr.OpenLedger(cid)
		
        // 2.根据拿到的每个账本初始化channel
		// Create a chain if we get a valid ledger with config block
		err = p.createChannel(cid, ledger, deployedCCInfoProvider, legacyLifecycleValidation, newLifecycleValidation)

		p.initChannel(cid)
	}
}
```

3. createChannel

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
    
    // 1.创建channel
	p.GossipService.InitializeChannel(bundle.ConfigtxValidator().ChannelID(), ordererSource, store, gossipservice.Support{
		
	})

	p.mutex.Lock()
	defer p.mutex.Unlock()
	
    // 2.如果channel创建成功，则将该channel添加到peer维护的channels中 
    if p.channels == nil {
		p.channels = map[string]*Channel{}
	}
	p.channels[cid] = channel

	return nil
}
```

4. InitializeChannel

   gossip\service\gossip_service.go

```go
// InitializeChannel allocates the state provider and should be invoked once per channel per execution
func (g *GossipService) InitializeChannel(channelID string, ordererSource *orderers.ConnectionSource, store *transientstore.Store, support Support) {
	
    ...
    
    // 1.这里监听消息端口，如果有消息就存到区块里
	g.chains[channelID] = state.NewGossipStateProvider(
		
	}

    // 2.周期性的向orderer节点请求区块
	// Delivery service might be nil only if it was not able to get connected
	// to the ordering service
	if g.deliveryService[channelID] != nil {
		// Parameters:
		//              - peer.gossip.useLeaderElection
		//              - peer.gossip.orgLeader
		//
		// are mutual exclusive, setting both to true is not defined, hence
		// peer will panic and terminate
		leaderElection := g.serviceConfig.UseLeaderElection
		isStaticOrgLeader := g.serviceConfig.OrgLeader

		if leaderElection && isStaticOrgLeader {
			logger.Panic("Setting both orgLeader and useLeaderElection to true isn't supported, aborting execution")
		}

		if leaderElection {
        	// 1.如果是动态选举，则进入动态选举过程。由选举出来的leader节点向orderer节点请求区块，并完成扩散数据块的操作
			g.leaderElection[channelID] = g.newLeaderElectionComponent(channelID, g.onStatusChangeFactory(channelID,
				support.Committer), g.metrics.ElectionMetrics)
		} else if isStaticOrgLeader {
			// 2.如果是静态选举，则由静态主节点直接向orderer请求区块，并完成转发的步骤
			g.deliveryService[channelID].StartDeliverForChannel(channelID, support.Committer, func() {})
		} else {
			// 3.error
		}
	} else {
		logger.Warning("Delivery client is down won't be able to pull blocks for chain", channelID)
	}

}
```

## 2.deliverServiceImpl.StartDeliverForChannel

core\deliverservice\deliveryclient.go

```go
// StartDeliverForChannel starts blocks delivery for channel
// initializes the grpc stream for given chainID, creates blocks provider instance
// that spawns in go routine to read new blocks starting from the position provided by ledger
// info instance.
func (d *deliverServiceImpl) StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error {
	d.lock.Lock()
	defer d.lock.Unlock()
	
	
	logger.Info("This peer will retrieve blocks from ordering service and disseminate to other peers in the organization for channel", chainID)

    // 1.初始化一个blocksprovider.Deliverer
	dc := &blocksprovider.Deliverer{
		ChannelID:     chainID,
		Gossip:        d.conf.Gossip,
		Ledger:        ledgerInfo,
		BlockVerifier: d.conf.CryptoSvc,
		Dialer: DialerAdapter{
			Client: d.conf.DeliverGRPCClient,
		},
		Orderers:          d.conf.OrdererSource,
		DoneC:             make(chan struct{}),
		Signer:            d.conf.Signer,
		DeliverStreamer:   DeliverAdapter{},
		Logger:            flogging.MustGetLogger("peer.blocksprovider").With("channel", chainID),
		MaxRetryDelay:     d.conf.DeliverServiceConfig.ReConnectBackoffThreshold,
		MaxRetryDuration:  d.conf.DeliverServiceConfig.ReconnectTotalTimeThreshold,
		InitialRetryDelay: 100 * time.Millisecond,
		YieldLeadership:   !d.conf.IsStaticLeader,
	}

    // 2.计算该deliver client的tls 的哈希值，用于安全的连接操作
	if d.conf.DeliverGRPCClient.MutualTLSRequired() {
		dc.TLSCertHash = util.ComputeSHA256(d.conf.DeliverGRPCClient.Certificate().Certificate[0])
	}

    // 3.在block provider中注册该deliver client
	d.blockProviders[chainID] = dc
    
    // 4.开始DeliverBlocks
	go func() {
		dc.DeliverBlocks()
		finalizer()
	}()
	return nil
}
```

## 3.DeliverBlocks.DeliverBlocks

internal\pkg\peer\blocksprovider\blocksprovider.go

```go
// DeliverBlocks used to pull out blocks from the ordering service to
// distributed them across peers
func (d *Deliverer) DeliverBlocks() {
	failureCounter := 0
	totalDuration := time.Duration(0)

	// InitialRetryDelay * backoffExponentBase^n > MaxRetryDelay
	// backoffExponentBase^n > MaxRetryDelay / InitialRetryDelay
	// n * log(backoffExponentBase) > log(MaxRetryDelay / InitialRetryDelay)
	// n > log(MaxRetryDelay / InitialRetryDelay) / log(backoffExponentBase)
	maxFailures := int(math.Log(float64(d.MaxRetryDelay)/float64(d.InitialRetryDelay)) / math.Log(backoffExponentBase))
	for {
		select {
		case <-d.DoneC:
			return
		default:
		}

        // 1.异常处理
		if failureCounter > 0 {
			...
		}

        // 2.拿到现在的高度
		ledgerHeight, err := d.Ledger.LedgerHeight()
		
		// 3.构造查询ledgerHeight高度的区块的Envelope
		seekInfoEnv, err := d.createSeekInfo(ledgerHeight)
		
		// 4.尝试连接orderer
		deliverClient, endpoint, cancel, err := d.connect(seekInfoEnv)
		
        // 5.创建一个chan，用于存储orderer.DeliverResponse
		recv := make(chan *orderer.DeliverResponse)
		go func() {
			for {
				// 接收到orderer.DeliverResponse
                resp, err := deliverClient.Recv()
				
				select {
                    // 1.存到recv中
				case recv <- resp:
				case <-d.DoneC:
                    // 2.关闭
					close(recv)
					return
				}
			}
		}()

	RecvLoop: // Loop until the endpoint is refreshed, or there is an error on the connection
		for {
			select {
			case <-endpoint.Refreshed:
				connLogger.Infof("Ordering endpoints have been refreshed, disconnecting from deliver to reconnect using updated endpoints")
				break RecvLoop
			case response, ok := <-recv:
				if !ok {
					connLogger.Warningf("Orderer hung up without sending status")
					failureCounter++
					break RecvLoop
				}
                // 处理orderer.DeliverResponse
				err = d.processMsg(response)
				if err != nil {
					connLogger.Warningf("Got error while attempting to receive blocks: %v", err)
					failureCounter++
					break RecvLoop
				}
				failureCounter = 0
			case <-d.DoneC:
				break RecvLoop
			}
		}

		// cancel and wait for our spawned go routine to exit
		cancel()
		<-recv
	}
}
```

### 1.Deliverer.connect

```go
func (d *Deliverer) connect(seekInfoEnv *common.Envelope) (orderer.AtomicBroadcast_DeliverClient, *orderers.Endpoint, func(), error) {
    // 1.拿到orderer的地址信息
	endpoint, err := d.Orderers.RandomEndpoint()
	// 2.尝试连接
	conn, err := d.Dialer.Dial(endpoint.Address, endpoint.CertPool)
	// 3.拿到context
	ctx, ctxCancel := context.WithCancel(context.Background())
	// 4.创建一个deliver client
	deliverClient, err := d.DeliverStreamer.Deliver(ctx, conn)
	// 5.发送查询信息
	err = deliverClient.Send(seekInfoEnv)
	// 6.返回一个关闭连接的函数
	return deliverClient, endpoint, func() {
		deliverClient.CloseSend()
		ctxCancel()
		conn.Close()
	}, nil
}
```



### 2.DeliverClient.Recv

internal\pkg\peer\blocksprovider\fake\ab_deliver_client.go

```go
func (fake *DeliverClient) Recv() (*orderer.DeliverResponse, error) {
	fake.recvMutex.Lock()
	ret, specificReturn := fake.recvReturnsOnCall[len(fake.recvArgsForCall)]
	fake.recvArgsForCall = append(fake.recvArgsForCall, struct {
	}{})
	fake.recordInvocation("Recv", []interface{}{})
	fake.recvMutex.Unlock()
	if fake.RecvStub != nil {
		return fake.RecvStub()
	}
	if specificReturn {
		return ret.result1, ret.result2
	}
	fakeReturns := fake.recvReturns
	return fakeReturns.result1, fakeReturns.result2
}
```

### 3.Deliverer.processMsg

internal\pkg\peer\blocksprovider\blocksprovider.go

```go
func (d *Deliverer) processMsg(msg *orderer.DeliverResponse) error {
    
	switch t := msg.Type.(type) {
    // 1.异常
	case *orderer.DeliverResponse_Status:
		if t.Status == common.Status_SUCCESS {
			return errors.Errorf("received success for a seek that should never complete")
		}
		return errors.Errorf("received bad status %v from orderer", t.Status)
    
    // 2.从orderer.DeliverResponse中，拿到区块
	case *orderer.DeliverResponse_Block:
		blockNum := t.Block.Header.Number
        // 1.验证block
		if err := d.BlockVerifier.VerifyBlock(gossipcommon.ChannelID(d.ChannelID), blockNum, t.Block); err != nil {
			return errors.WithMessage(err, "block from orderer could not be verified")
		}
		marshaledBlock, err := proto.Marshal(t.Block)

		// 2.Create payload with a block received
		payload := &gossip.Payload{
			Data:   marshaledBlock,
			SeqNum: blockNum,
		}

		// 3.Use payload to create gossip message
		gossipMsg := &gossip.GossipMessage{
			Nonce:   0,
			Tag:     gossip.GossipMessage_CHAN_AND_ORG,
			Channel: []byte(d.ChannelID),
			Content: &gossip.GossipMessage_DataMsg{
				DataMsg: &gossip.DataMessage{
					Payload: payload,
				},
			},
		}

		d.Logger.Debugf("Adding payload to local buffer, blockNum = [%d]", blockNum)
		// 4.Add payload to local state payloads buffer
		if err := d.Gossip.AddPayload(d.ChannelID, payload); err != nil {
			d.Logger.Warningf("Block [%d] received from ordering service wasn't added to payload buffer: %v", blockNum, err)
			return errors.WithMessage(err, "could not add block as payload")
		}

		// 5.Gossip messages with other nodes
		d.Logger.Debugf("Gossiping block [%d]", blockNum)
		d.Gossip.Gossip(gossipMsg)
		return nil
	default:
		d.Logger.Warningf("Received unknown: %v", t)
		return errors.Errorf("unknown message type '%T'", msg.Type)
	}
}
```

#### 1.GossipServiceAdapter.AddPayload

Leader主节点会直接block添加到缓冲区中，不像其他的peer那样，存到message store中

internal\pkg\peer\blocksprovider\fake\gossip_service_adapter.go

```go
func (fake *GossipServiceAdapter) AddPayload(arg1 string, arg2 *gossip.Payload) error {
	fake.addPayloadMutex.Lock()
	ret, specificReturn := fake.addPayloadReturnsOnCall[len(fake.addPayloadArgsForCall)]
	fake.addPayloadArgsForCall = append(fake.addPayloadArgsForCall, struct {
		arg1 string
		arg2 *gossip.Payload
	}{arg1, arg2})
	fake.recordInvocation("AddPayload", []interface{}{arg1, arg2})
	fake.addPayloadMutex.Unlock()
    
	if fake.AddPayloadStub != nil {
		return fake.AddPayloadStub(arg1, arg2)
	}
	if specificReturn {
		return ret.result1
	}
	fakeReturns := fake.addPayloadReturns
	return fakeReturns.result1
}
```

#### 2.Deliverer.Gossip.Gossip(gossipMsg)

internal\pkg\peer\blocksprovider\fake\gossip_service_adapter.go

```go
func (fake *GossipServiceAdapter) Gossip(arg1 *gossip.GossipMessage) {
	fake.gossipMutex.Lock()
	fake.gossipArgsForCall = append(fake.gossipArgsForCall, struct {
		arg1 *gossip.GossipMessage
	}{arg1})
	fake.recordInvocation("Gossip", []interface{}{arg1})
	fake.gossipMutex.Unlock()
	if fake.GossipStub != nil {
		fake.GossipStub(arg1)
	}
}
```



# 2 Orderer节点收到请求&发送区块

