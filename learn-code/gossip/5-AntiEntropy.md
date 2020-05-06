# 0 Gossip反熵算法

Gossip消息模块引入反熵算法，用于解决网络延迟等原因造成节点间账本状态（账本高度）的差异问题，以降低整个通道内所有节点间账本状态的不一致程度，即“反熵”。

反熵算法antiEntropy()从其他节点同步本地账本的缺失数据，包括区块数据与隐私数据，并通过区块号范围来标识缺失的数据集合。

## 1.入口-gossip 通道初始化

1. GossipService.InitializeChannel

   gossip\service\gossip_service.go

```go
// InitializeChannel allocates the state provider and should be invoked once per channel per execution
func (g *GossipService) InitializeChannel(channelID string, ordererSource *orderers.ConnectionSource, store *transientstore.Store, support Support) {
	
	g.chains[channelID] = state.NewGossipStateProvider(
		flogging.MustGetLogger(util.StateLogger),
		channelID,
		servicesAdapter,
		coordinator,
		g.metrics.StateMetrics,
		blockingMode,
		stateConfig)
}
```

2. NewGossipStateProvider

   gossip\state\state.go

```go
// NewGossipStateProvider creates state provider with coordinator instance
// to orchestrate arrival of private rwsets and blocks before committing them into the ledger.
func NewGossipStateProvider(
	logger util.Logger,
	chainID string,
	services *ServicesMediator,
	ledger ledgerResources,
	stateMetrics *metrics.StateMetrics,
	blockingMode bool,
	config *StateConfig,
) GossipStateProvider {
	
    ...
    
	// Listen for incoming communication
	go s.receiveAndQueueGossipMessages(gossipChan)
	go s.receiveAndDispatchDirectMessages(commChan)
	// Deliver in order messages into the incoming channel
	go s.deliverPayloads()
	if s.config.StateEnabled {
		// Execute anti entropy to fill missing gaps
		go s.antiEntropy()
	}
	// Taking care of state request messages
	go s.processStateRequests()

	return s
}
```

Gossip通道初始化后，会执行go s.antiEntropy()开启反熵线程

## 2.反熵算法

gossip\state\state.go

```go
func (s *GossipStateProviderImpl) antiEntropy() {
	defer s.logger.Debug("State Provider stopped, stopping anti entropy procedure.")

	for {
		select {
		case <-s.stopCh:
			return
        // 定时器消息
		case <-time.After(s.config.StateCheckInterval):
            // 获取账本高度
			ourHeight, err := s.ledger.LedgerHeight()
			if err != nil {
				// Unable to read from ledger continue to the next round
				s.logger.Errorf("Cannot obtain ledger height, due to %+v", errors.WithStack(err))
				continue
			}
			if ourHeight == 0 {
				s.logger.Error("Ledger reported block height of 0 but this should be impossible")
				continue
			}
            // 获取最大账本高度
			maxHeight := s.maxAvailableLedgerHeight()
            // 检查本地的账本高度current是否小于最大账本高度max，即当前账本是否缺少了其他节点的账本数据
			if ourHeight >= maxHeight {
				continue
			}
			// 向其他节点请求指定范围内缺失的数据集合
			s.requestBlocksInRange(uint64(ourHeight), uint64(maxHeight)-1)
		}
	}
}
```



# 1 获取当前最大的账本高度

## 1.s.maxAvailableLedgerHeight

gossip\state\state.go

```go
// maxAvailableLedgerHeight iterates over all available peers and checks advertised meta state to
// find maximum available ledger height across peers
func (s *GossipStateProviderImpl) maxAvailableLedgerHeight() uint64 {
	max := uint64(0)
    // 遍历过滤后的members节点列表
	for _, p := range s.mediator.PeersOfChannel(common2.ChannelID(s.chainID)) {
		if p.Properties == nil {
			s.logger.Debug("Peer", p.PreferredEndpoint(), "doesn't have properties, skipping it")
			continue
		}
        // 从节点的Properties字段中解析出账本高度peerHeight
		peerHeight := p.Properties.LedgerHeight
        // 如果当前记录的最大账本高度max比peerHeight小，则将max更新为最新的账本高度peerHeight。
		if max < peerHeight {
			max = peerHeight
		}
	}
	return max
}
```

### 1.s.mediator.PeersOfChannel

```go
// PeersOfChannel returns the NetworkMembers considered alive
// and also subscribed to the channel given
func (g *Node) PeersOfChannel(channel common.ChannelID) []discovery.NetworkMember {
    // 获取与指定通道（channel）对应的GossipChannel通道对象gc
	gc := g.chanState.getGossipChannelByChainID(channel)
	if gc == nil {
		g.logger.Debug("No such channel", channel)
		return nil
	}
	// 获取通道上符合条件的成员节点列表
	return gc.GetPeers()
}
```

### 2.gc.GetPeers

```go
// GetPeers returns a list of peers with metadata as published by them
func (gc *gossipChannel) GetPeers() []discovery.NetworkMember {
	var members []discovery.NetworkMember
	if gc.hasLeftChannel() {
		return members
	}
	// 遍历当前节点discovery模块中aliveMembership对象的存活节点消息列表
	for _, member := range gc.GetMembership() {
        // idMapper模块的pkiID2Cert字典和stateInfoMsgStore消息存储对象中都应该保存指定节点的身份证书信息和StateInfo消息，检查是否存在；
		if !gc.EligibleForChannel(member) {
			continue
		}
        // 从stateInfoMsgStore对象中获取与指定节点（member.PKIid）对应的StateInfo消息
		stateInf := gc.stateInfoMsgStore.MsgByID(member.PKIid)
		if stateInf == nil {
			continue
		}
		props := stateInf.GetStateInfo().Properties
		if props != nil && props.LeftChannel {
			continue
		}
        // 检查合法后将该消息的Envelop字段、Properties字段（封装了账本高度）等更新到member中，再将检查更新过的member添加到节点列表members中。
		member.Properties = stateInf.GetStateInfo().Properties
		member.Envelope = stateInf.Envelope
		members = append(members, member)
	}
	return members
}
```

# 2 分批发送远程状态请求消息

## 1.s.requestBlocksInRange

gossip\state\state.go

```go
// requestBlocksInRange capable to acquire blocks with sequence
// numbers in the range [start...end).
func (s *GossipStateProviderImpl) requestBlocksInRange(start uint64, end uint64) {
    // 1.首先设置stateTransferActive标志位为1，表示state模块正在请求从远程节点获取本地的缺失数据，并在退出时重置该标志位为0。
	atomic.StoreInt32(&s.stateTransferActive, 1)
	defer atomic.StoreInt32(&s.stateTransferActive, 0)

    // 建立消息处理循环，采用分批请求数据的方式，将StateRequest类型的远程状态请求消息发送到其他符合条件的节点上，设置本次循环的数据请求范围是{prev，end}。
    // prev设置为当前本地的账本高度current，end设置为当前的最大账本高度max
	for prev := start; prev <= end; {
        // 1.获取下一个区块号
		next := min(end, prev+s.config.StateBatchSize)
		// 2.创建StateRequest远程状态请求消息
		gossipMsg := s.stateRequestMessage(prev, next)

        // 将responseReceived标志位作为内层循环的判断条件
		responseReceived := false
		tryCounts := 0
		// 没有接收到响应消息,如果成功接收到每次指定范围内的数据集合，则结束内层消息处理循环，并跳出到外层的消息处理循环中继续发送请求消息。
		for !responseReceived {
            // 3.检查是否超过最大重试次数
			if tryCounts > s.config.StateMaxRetries {
				return
			}
			// 4.过滤出符合条件的节点列表,选择Peer节点请求数据
			peer, err := s.selectPeerToRequestFrom(next)
			// 5.通过comm模块将StateRequest消息发送给指定节点peer，并阻塞等待通道消息
			s.mediator.Send(gossipMsg, peer)
			// 6.重试次数增1
            tryCounts++

			// 7.等待直到超时或有响应消息到达
			select {
           	// 1.接收到状态响应消息通道的消息
			case msg, stillOpen := <-s.stateResponseCh:
				if !stillOpen {
					return
				}
                // 2.检查Nonce随机数是否一致，以判断该响应消息是否有效
				if msg.GetGossipMessage().Nonce !=
					gossipMsg.Nonce {
					continue
				}
				// Got corresponding response for state request, can continue
				// 3.处理状态响应消息
                index, err := s.handleStateResponse(msg)
				// 4.更新prev对象
				prev = index + 1
                // 5.收到回复消息
				responseReceived = true
            // 若认为没有接收到响应消息，则跳转到内层的消息处理循环起始处重试；
			case <-time.After(s.config.StateResponseTimeout):
			}
		}
	}
}
```

## 2.selectPeerToRequestFrom

gossip\state\state.go

```go
// selectPeerToRequestFrom selects peer which has required blocks to ask missing blocks from
func (s *GossipStateProviderImpl) selectPeerToRequestFrom(height uint64) (*comm.RemotePeer, error) {
	// Filter peers which posses required range of missing blocks
	peers := s.filterPeers(s.hasRequiredHeight(height))

	n := len(peers)
	
	// Select peer to ask for blocks
	return peers[util.RandomInt(n)], nil
}
```

### 1.s.filterPeers

gossip\state\state.go

```go
// filterPeers returns list of peers which aligns the predicate provided
func (s *GossipStateProviderImpl) filterPeers(predicate func(peer discovery.NetworkMember) bool) []*comm.RemotePeer {
	var peers []*comm.RemotePeer
	// 遍历通道中的Peer节点列表
	for _, member := range s.mediator.PeersOfChannel(common2.ChannelID(s.chainID)) {
        // 判断该节点成员是否符合条件，若符合则添加到Peer节点列表中
		if predicate(member) {
			peers = append(peers, &comm.RemotePeer{Endpoint: member.PreferredEndpoint(), PKIID: member.PKIid})
		}
	}

	return peers
}
```

### 2.predicate

predicate就是传进来判断是否达到高度要求的函数

s.hasRequiredHeight(height):

```go
// hasRequiredHeight returns predicate which is capable to filter peers with ledger height above than indicated by provided input parameter
// 测试该Peer节点账本是否拥有指定请求的区块高度，即是否包含请求的区块
func (s *GossipStateProviderImpl) hasRequiredHeight(height uint64) func(peer discovery.NetworkMember) bool {
	return func(peer discovery.NetworkMember) bool {
		if peer.Properties != nil {
            // 判断账本高度是否达到请求的区块高度
			return peer.Properties.LedgerHeight >= height
		}
		return false
	}
}
```

## 3.s.mediator.Send(gossipMsg, peer)

```go
// Send sends a message to remote peers
func (g *Node) Send(msg *pg.GossipMessage, peers ...*comm.RemotePeer) {
	m, err := protoext.NoopSign(msg)
    
	g.comm.Send(m, peers...)
}

```

### 1.g.comm.Send

```go
func (c *commImpl) Send(msg *protoext.SignedGossipMessage, peers ...*RemotePeer) {
	if c.isStopping() || len(peers) == 0 {
		return
	}
	c.logger.Debug("Entering, sending", msg, "to ", len(peers), "peers")

	for _, peer := range peers {
		go func(peer *RemotePeer, msg *protoext.SignedGossipMessage) {
			c.sendToEndpoint(peer, msg, nonBlockingSend)
		}(peer, msg)
	}
}
```

---

# 3 远程节点处理远程状态请求消息

## 1.远程节点回复远程状态响应消息

### 1.入口-远程节点接收到状态响应消息

1. Gossip通道初始化

   gossip\service\gossip_service.go

```go
// InitializeChannel allocates the state provider and should be invoked once per channel per execution
func (g *GossipService) InitializeChannel(channelID string, ordererSource *orderers.ConnectionSource, store *transientstore.Store, support Support) {
	g.lock.Lock()
	defer g.lock.Unlock()
	...
    blockingMode := !g.serviceConfig.NonBlockingCommitMode
	stateConfig := state.GlobalConfig()
	g.chains[channelID] = state.NewGossipStateProvider(
		flogging.MustGetLogger(util.StateLogger),
		channelID,
		servicesAdapter,
		coordinator,
		g.metrics.StateMetrics,
		blockingMode,
		stateConfig)
	...

}
```

2. state.NewGossipStateProvider

   gossip\state\state.go

```go
// NewGossipStateProvider creates state provider with coordinator instance
// to orchestrate arrival of private rwsets and blocks before committing them into the ledger.
func NewGossipStateProvider(
	logger util.Logger,
	chainID string,
	services *ServicesMediator,
	ledger ledgerResources,
	stateMetrics *metrics.StateMetrics,
	blockingMode bool,
	config *StateConfig,
) GossipStateProvider {
	
	...
    
	// Listen for incoming communication
	go s.receiveAndQueueGossipMessages(gossipChan)
	go s.receiveAndDispatchDirectMessages(commChan)
    
	// Deliver in order messages into the incoming channel
	go s.deliverPayloads()
	if s.config.StateEnabled {
		// Execute anti entropy to fill missing gaps
		go s.antiEntropy()
	}
	// Taking care of state request messages
	go s.processStateRequests()

	return s
}
```

3. s.processStateRequests()

   gossip\state\state.go

```go
func (s *GossipStateProviderImpl) processStateRequests() {
	for {
		msg, stillOpen := <-s.stateRequestCh
		if !stillOpen {
			return
		}
		s.handleStateRequest(msg)
	}
}
```

### 2.s.handleStateRequest(msg)

gossip\state\state.go

```go
// handleStateRequest handles state request message, validate batch size, reads current leader state to
// obtain required blocks, builds response message and send it back
func (s *GossipStateProviderImpl) handleStateRequest(msg protoext.ReceivedMessage) {
	// 1.从接收消息中解析出StateRequest消息request
	request := msg.GetGossipMessage().GetStateRequest()
	// 2.检查消息的合法性
	if err := s.requestValidator.validate(request, s.config.StateBatchSize); err != nil {
		return
	}
	// 3.获取本地账本的高度
	currentHeight, err := s.ledger.LedgerHeight()
	// 4.设置请求的数据范围上限endSeqNum
	endSeqNum := min(currentHeight, request.EndSeqNum)
	// 5.初始化状态响应消息
	response := &proto.RemoteStateResponse{Payloads: make([]*proto.Payload, 0)}
	// 6.开始遍历请求的数据范围
    for seqNum := request.StartSeqNum; seqNum <= endSeqNum; seqNum++ {
		// 1.获取请求节点的连接信息
		connInfo := msg.GetConnectionInfo()
		// 2.构造请求节点的peerAuthorInfo签名数据对象
        peerAuthInfo := protoutil.SignedData{
			Data:      connInfo.Auth.SignedData,
			Signature: connInfo.Auth.Signature,
			Identity:  connInfo.Identity,
		}
        // 3.从本地账本的区块文件中获取指定区块号seqNum的区块block
		block, pvtData, err := s.ledger.GetPvtDataAndBlockByNum(seqNum, peerAuthInfo)
		// 4.得到block的二进制流
		blockBytes, err := pb.Marshal(block)
		// 5.得到隐私数据的二进制流
		var pvtBytes [][]byte
		if pvtData != nil {
			// Marshal private data
			pvtBytes, err = pvtData.Marshal()
		}

		// 6.将blcok和private data添加到响应消息的负载列表中
		response.Payloads = append(response.Payloads, &proto.Payload{
			SeqNum:      seqNum,
			Data:        blockBytes,
			PrivateData: pvtBytes,
		})
	}
	// 7.将相应消息返回给数据请求者
	msg.Respond(&proto.GossipMessage{
		// Copy nonce field from the request, so it will be possible to match response
		Nonce:   msg.GetGossipMessage().Nonce,
		Tag:     proto.GossipMessage_CHAN_OR_ORG,
		Channel: []byte(s.chainID),
		Content: &proto.GossipMessage_StateResponse{StateResponse: response},
	})
}
```

#### 3.s.ledger.GetPvtDataAndBlockByNum

core\committer\committer_impl.go

```go
// GetPvtDataAndBlockByNum gets block by number and also returns all related private data that requesting peer is eligible for.
// The order of private data in slice of PvtDataCollections doesn't imply the order of transactions in the block related to these private data, to get the correct placement need to read TxPvtData.SeqInBlock field
func (c *coordinator) GetPvtDataAndBlockByNum(seqNum uint64, peerAuthInfo protoutil.SignedData) (*common.Block, util.PvtDataCollections, error) {
	// 1.从committer那里拿到数据
    blockAndPvtData, err := c.Committer.GetPvtDataAndBlockByNum(seqNum)
	// 待返回的隐私数据结果集合
	seqs2Namespaces := aggregatedCollections{}
	for seqInBlock := range blockAndPvtData.Block.Data.Data {
		txPvtDataItem, exists := blockAndPvtData.PvtData[uint64(seqInBlock)]

		// Iterate through the private write sets and include them in response if requesting peer is eligible for it
		for _, ns := range txPvtDataItem.WriteSet.NsPvtRwset {
			for _, col := range ns.CollectionPvtRwset {
				cc := privdata.CollectionCriteria{
					Channel:    c.ChainID,
					Namespace:  ns.Namespace,
					Collection: col.CollectionName,
				}
				sp, err := c.CollectionStore.RetrieveCollectionAccessPolicy(cc)
				
				isAuthorized := sp.AccessFilter()
                // 验证peerAuthInfo是否满足账本中保存的隐私数据集合访问权限策略的要求，将通过验证的隐私数据集合转换成隐私数据列表
				if !isAuthorized(peerAuthInfo) {
					logger.Debugf("Skipping collection criteria [%#v] because peer isn't authorized", cc)
					continue
				}
				seqs2Namespaces.addCollection(uint64(seqInBlock), txPvtDataItem.WriteSet.DataModel, ns.Namespace, col)
			}
		}
	}

	return blockAndPvtData.Block, seqs2Namespaces.asPrivateData(), nil
}
```



# 4 本地节点处理远程状态响应消息

在第二部分（分批发送远程状态请求消息）中，发送远程状态响应消息之后，等待远程节点的相应

## 1.入口-处理远端节点的响应

```go
// requestBlocksInRange capable to acquire blocks with sequence
// numbers in the range [start...end).
func (s *GossipStateProviderImpl) requestBlocksInRange(start uint64, end uint64) {
   
    // 建立消息处理循环，采用分批请求数据的方式，将StateRequest类型的远程状态请求消息发送到其他符合条件的节点上，设置本次循环的数据请求范围是{prev，end}。
    // prev设置为当前本地的账本高度current，end设置为当前的最大账本高度max
	for prev := start; prev <= end; {
   		...
		// 没有接收到响应消息,如果成功接收到每次指定范围内的数据集合，则结束内层消息处理循环，并跳出到外层的消息处理循环中继续发送请求消息。
		for !responseReceived {
            ...
			// 5.发送---通过comm模块将StateRequest消息发送给指定节点peer，并阻塞等待通道消息
			s.mediator.Send(gossipMsg, peer)
			...
            
			// 7.等待直到超时或有响应消息到达
			select {
           	// 1.接收到状态响应消息通道的消息
			case msg, stillOpen := <-s.stateResponseCh:
				...
				// Got corresponding response for state request, can continue
				// 3.处理状态响应消息
                index, err := s.handleStateResponse(msg)
				...
                // 5.收到回复消息
				responseReceived = true
            // 若认为没有接收到响应消息，则跳转到内层的消息处理循环起始处重试；
			case <-time.After(s.config.StateResponseTimeout):
			}
		}
	}
}
```

## 2.s.handleStateResponse(msg)

```go
func (s *GossipStateProviderImpl) handleStateResponse(msg protoext.ReceivedMessage) (uint64, error) {
	max := uint64(0)
	// 1.拿到状态请求的响应消息
	response := msg.GetGossipMessage().GetStateResponse()
    // 2.提取消息负载，验证和添加到本地消息负载缓冲区
    if len(response.GetPayloads()) == 0 {
		return uint64(0), errors.New("Received state transfer response without payload")
	}
	// 3.获取并遍历消息负载列表
	for _, payload := range response.GetPayloads() {
		// 1.拿到block
		block, err := protoutil.UnmarshalBlock(payload.Data)
		// 2.验证block的有效性，如区块元数据的签名集合是否满足区块验证策略等。
		if err := s.mediator.VerifyBlock(common2.ChannelID(s.chainID), payload.SeqNum, block); err != nil {
			return uint64(0), err
		}
        // 3.更新最高值
		if max < payload.SeqNum {
			max = payload.SeqNum
		}
		// 4.以阻塞方式将当前消息负载payload添加到本地的消息负载缓冲区中，等待处理并提交到账本。
		err = s.addPayload(payload, blocking)
	}
	return max, nil
}
```

