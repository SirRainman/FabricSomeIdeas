# 4 写在前面-将block存到账本中

问题：peer节点接收到DataMsg之后，是怎么把blockMsgStore存到缓冲区中的？

## 0.入口

什么时候开始存储区块的入口主要有两点，这里只涉及peer节点启动时的操作

1. peer节点启动
2. peer节点加入一个channel



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
	
    // 1.初始化peer channel上的 GossipSevice上的各个服务
    ...
    
    // 2.初始化GossipSevice的state provider服务
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

## 1.state.NewGossipStateProvider

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
	
    // 0.准备工作---拿到gossipChan
    gossipChan, _ := services.Accept(func(message interface{}) bool {
		// Get only data messages
		return protoext.IsDataMsg(message.(*proto.GossipMessage)) &&
			bytes.Equal(message.(*proto.GossipMessage).Channel, []byte(chainID))
	}, false)

	remoteStateMsgFilter := func(message interface{}) bool {
		receivedMsg := message.(protoext.ReceivedMessage)
		msg := receivedMsg.GetGossipMessage()
		if !(protoext.IsRemoteStateMessage(msg.GossipMessage) || msg.GetPrivateData() != nil) {
			return false
		}
		// Ensure we deal only with messages that belong to this channel
		if !bytes.Equal(msg.Channel, []byte(chainID)) {
			return false
		}
		connInfo := receivedMsg.GetConnectionInfo()
		authErr := services.VerifyByChannel(msg.Channel, connInfo.Identity, connInfo.Auth.Signature, connInfo.Auth.SignedData)
		if authErr != nil {
			logger.Warning("Got unauthorized request from", string(connInfo.Identity))
			return false
		}
		return true
	}

    // 0.准备工作---拿到commChan
	// Filter message which are only relevant for nodeMetastate transfer
	_, commChan := services.Accept(remoteStateMsgFilter, true)

	height, err := ledger.LedgerHeight()
	
	s := &GossipStateProviderImpl{
		logger: logger,
		// MessageCryptoService
		mediator: services,
		// Chain ID
		chainID: chainID,
		// Create a queue for payloads, wrapped in a metrics buffer
		payloads: &metricsBuffer{},
		ledger:              ledger,
		stateResponseCh:     make(chan protoext.ReceivedMessage, config.StateChannelSize),
		stateRequestCh:      make(chan protoext.ReceivedMessage, config.StateChannelSize),
		stopCh:              make(chan struct{}),
		stateTransferActive: 0,
		once:                sync.Once{},
		stateMetrics:        stateMetrics,
		requestValidator:    &stateRequestValidator{},
		blockingMode:        blockingMode,
		config:              config,
	}
    
    ...
    
    // 1.更新区块高度
	services.UpdateLedgerHeight(height, common2.ChannelID(s.chainID))

	// 2.Listen for incoming communication
	go s.receiveAndQueueGossipMessages(gossipChan)
	// 3.接收和转发消息
    go s.receiveAndDispatchDirectMessages(commChan)
	
    // 4.从缓冲区中读取数据并存到区块中
    // Deliver in order messages into the incoming channel
	go s.deliverPayloads()
	
    // 5.反熵函数--判断是否pull的机制
    if s.config.StateEnabled {
		// Execute anti entropy to fill missing gaps
		go s.antiEntropy()
	}
	
    // 6.Taking care of state request messages
	go s.processStateRequests()

	return s
}
```



# 5 将Block放到缓冲区

## 1.拿到消息Chan-gossipChan

gossip\state\state.go

```go
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
	gossipChan, _ := services.Accept(func(message interface{}) bool {
		// Get only data messages
		return protoext.IsDataMsg(message.(*proto.GossipMessage)) &&
			bytes.Equal(message.(*proto.GossipMessage).Channel, []byte(chainID))
	}, false)
    
    ...
}
```

### 1.Node.Accept

gossip\gossip\gossip_impl.go

```go
// Accept returns a dedicated read-only channel for messages sent by other nodes that match a certain predicate.
// If passThrough is false, the messages are processed by the gossip layer beforehand.
// If passThrough is true, the gossip layer doesn't intervene and the messages
// can be used to send a reply back to the sender
func (g *Node) Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *pg.GossipMessage, <-chan protoext.ReceivedMessage) {
	// 1如果是拿commChan则通过另外一种方法拿取
    if passThrough {
		return nil, g.comm.Accept(acceptor)
	}
    // 2设置回调函数，判单消息的类型
	acceptByType := func(o interface{}) bool {
		if o, isGossipMsg := o.(*pg.GossipMessage); isGossipMsg {
			return acceptor(o)
		}
		if o, isSignedMsg := o.(*protoext.SignedGossipMessage); isSignedMsg {
			sMsg := o
			return acceptor(sMsg.GossipMessage)
		}
		g.logger.Warning("Message type:", reflect.TypeOf(o), "cannot be evaluated")
		return false
	}
    // 3通过gossip node 拿到一个inCh
	inCh := g.AddChannel(acceptByType)
	outCh := make(chan *pg.GossipMessage, acceptChanSize)
	go func() {
		defer close(outCh)
		for {
			select {
			case <-g.toDieChan:
				return
            // 1.如果inCh中来了新的数据
			case m, channelOpen := <-inCh:
				if !channelOpen {
					return
				}
				select {
				case <-g.toDieChan:
					return
                // 2.如果可以向outCh里面放数据
				case outCh <- m.(*protoext.SignedGossipMessage).GossipMessage:
				}
			}
		}
	}()
	return outCh, nil
}
```

### 2.g.AddChannel

gossip\comm\demux.go

```go
// AddChannel registers a channel with a certain predicate. AddChannel
// returns a read-only channel that will produce values that are
// matched by the predicate function.
//
// If the DeMultiplexer is closed, the channel returned will be closed
// to prevent users of the channel from waiting on the channel.
func (m *ChannelDeMultiplexer) AddChannel(predicate common.MessageAcceptor) <-chan interface{} {
	m.lock.Lock()
	if m.closed { // closed once, can't put anything more in.
		m.lock.Unlock()
		ch := make(chan interface{})
		close(ch)
		return ch
	}
	bidirectionalCh := make(chan interface{}, 10)
	// Assignment to channel converts bidirectionalCh to send-only.
	// Return converts bidirectionalCh to a receive-only.
    // 添加一个channel
	ch := &channel{ch: bidirectionalCh, pred: predicate} 
	m.channels = append(m.channels, ch)
	m.lock.Unlock()
	return bidirectionalCh
}

```

### 3.ChannelDeMultiplexer

```go
// ChannelDeMultiplexer is a struct that can receive channel registrations (AddChannel)
// and publications (DeMultiplex) and it broadcasts the publications to registrations
// according to their predicate. Can only be closed once and never open after a close.
type ChannelDeMultiplexer struct {
	// lock protects everything below it.
	lock   sync.Mutex
	closed bool // one way boolean from false -> true
	stopCh chan struct{}
	// deMuxInProgress keeps track of any calls to DeMultiplex
	// that are still being handled. This is used to determine
	// when it is safe to close all of the tracked channels
	deMuxInProgress sync.WaitGroup
	channels        []*channel
}

```



## 2.go s.receiveAndQueueGossipMessages(gossipChan)

将gossipChan中的数据放到缓冲区里

gossip\state\state.go

```go
func (s *GossipStateProviderImpl) receiveAndQueueGossipMessages(ch <-chan *proto.GossipMessage) {
	for msg := range ch {
		s.logger.Debug("Received new message via gossip channel")
		go func(msg *proto.GossipMessage) {
			if !bytes.Equal(msg.Channel, []byte(s.chainID)) {
				s.logger.Warning("Received enqueue for channel",
					string(msg.Channel), "while expecting channel", s.chainID, "ignoring enqueue")
				return
			}

			dataMsg := msg.GetDataMsg()
			if dataMsg != nil {
				if err := s.addPayload(dataMsg.GetPayload(), nonBlocking); err != nil {
					s.logger.Warningf("Block [%d] received from gossip wasn't added to payload buffer: %v", dataMsg.Payload.SeqNum, err)
					return
				}

			} else {
				s.logger.Debug("Gossip message received is not of data message type, usually this should not happen.")
			}
		}(msg)
	}
}
```

### 1.GossipStateProviderImpl.addPayload

gossip\state\state.go

```go
// addPayload adds new payload into state. It may (or may not) block according to the
// given parameter. If it gets a block while in blocking mode - it would wait until
// the block is sent into the payloads buffer.
// Else - it may drop the block, if the payload buffer is too full.
func (s *GossipStateProviderImpl) addPayload(payload *proto.Payload, blockingMode bool) error {
	if payload == nil {
		return errors.New("Given payload is nil")
	}
	s.logger.Debugf("[%s] Adding payload to local buffer, blockNum = [%d]", s.chainID, payload.SeqNum)
	height, err := s.ledger.LedgerHeight()
	if err != nil {
		return errors.Wrap(err, "Failed obtaining ledger height")
	}

	if !blockingMode && payload.SeqNum-height >= uint64(s.config.StateBlockBufferSize) {
		return errors.Errorf("Ledger height is at %d, cannot enqueue block with sequence of %d", height, payload.SeqNum)
	}

	for blockingMode && s.payloads.Size() > s.config.StateBlockBufferSize*2 {
		time.Sleep(enqueueRetryInterval)
	}

    // 添加到缓冲区中
	s.payloads.Push(payload)
	s.logger.Debugf("Blocks payloads buffer size for channel [%s] is %d blocks", s.chainID, s.payloads.Size())
	return nil
}
```



# 6 Peer节点处理数据消息并提交账本

## 1.go s.deliverPayloads()

gossip\state\state.go

```go
func (s *GossipStateProviderImpl) deliverPayloads() {
	for {
		select {
        // 如果缓冲区接收到了block
		// Wait for notification that next seq has arrived
		case <-s.payloads.Ready():
			
			// 1.Collect all subsequent payloads
			for payload := s.payloads.Pop(); payload != nil; payload = s.payloads.Pop() {
				rawBlock := &common.Block{}
                // 2.拿到block
				if err := pb.Unmarshal(payload.Data, rawBlock); err != nil {
					s.logger.Errorf("Error getting block with seqNum = %d due to (%+v)...dropping block", payload.SeqNum, errors.WithStack(err))
					continue
				}
								
				// 3.拿到私有数据
                // Read all private data into slice
				var p util.PvtDataCollections
				if payload.PrivateData != nil {
					err := p.Unmarshal(payload.PrivateData)
					
				}
                // 4.将私有数据和block一同提交到账本中
				if err := s.commitBlock(rawBlock, p); err != nil {
					if executionErr, isExecutionErr := err.(*vsccErrors.VSCCExecutionFailureError); isExecutionErr {
						s.logger.Errorf("Failed executing VSCC due to %v. Aborting chain processing", executionErr)
						return
					}
					
				}
			}
		case <-s.stopCh:
			s.logger.Debug("State provider has been stopped, finishing to push new blocks.")
			return
		}
	}
}
```

## 2.state.commitBlock

gossip\state\state.go

```go
func (s *GossipStateProviderImpl) commitBlock(block *common.Block, pvtData util.PvtDataCollections) error {

	t1 := time.Now()

	// 1.Commit block with available private transactions
	if err := s.ledger.StoreBlock(block, pvtData); err != nil {
		s.logger.Errorf("Got error while committing(%+v)", errors.WithStack(err))
		return err
	}

    // 2.交流的时间
	sinceT1 := time.Since(t1)
	s.stateMetrics.CommitDuration.With("channel", s.chainID).Observe(sinceT1.Seconds())

	// 3.Update ledger height
	s.mediator.UpdateLedgerHeight(block.Header.Number+1, common2.ChannelID(s.chainID))
	s.stateMetrics.Height.With("channel", s.chainID).Set(float64(block.Header.Number + 1))

	return nil
}
```

### 1.s.ledger.StoreBlock

gossip\privdata\coordinator.go

```go
// StoreBlock stores block with private data into the ledger
func (c *coordinator) StoreBlock(block *common.Block, privateDataSets util.PvtDataCollections) error {
	
	validationStart := time.Now()
    // 1.验证block的有效性
	err := c.Validator.Validate(block)
	c.reportValidationDuration(time.Since(validationStart))

    // 2.初始化block和private data
	blockAndPvtData := &ledger.BlockAndPvtData{
		Block:          block,
		PvtData:        make(ledger.TxPvtDataMap),
		MissingPvtData: make(ledger.TxMissingPvtDataMap),
	}

    // 3.判断账本中是否达到了该block到达的高度
	exist, err := c.DoesPvtDataInfoExistInLedger(block.Header.Number)
	if exist {
		commitOpts := &ledger.CommitOptions{FetchPvtDataFromLedger: true}
		return c.CommitLegacy(blockAndPvtData, commitOpts)
	}

	listMissingPrivateDataDurationHistogram := c.metrics.ListMissingPrivateDataDuration.With("channel", c.ChainID)
	fetchDurationHistogram := c.metrics.FetchDuration.With("channel", c.ChainID)
	purgeDurationHistogram := c.metrics.PurgeDuration.With("channel", c.ChainID)
	pdp := &PvtdataProvider{
		mspID:                                   c.mspID,
		selfSignedData:                          c.selfSignedData,
		logger:                                  logger.With("channel", c.ChainID),
		listMissingPrivateDataDurationHistogram: listMissingPrivateDataDurationHistogram,
		fetchDurationHistogram:                  fetchDurationHistogram,
		purgeDurationHistogram:                  purgeDurationHistogram,
		transientStore:                          c.store,
		pullRetryThreshold:                      c.pullRetryThreshold,
		prefetchedPvtdata:                       privateDataSets,
		transientBlockRetention:                 c.transientBlockRetention,
		channelID:                               c.ChainID,
		blockNum:                                block.Header.Number,
		storePvtdataOfInvalidTx:                 c.Support.CapabilityProvider.Capabilities().StorePvtDataOfInvalidTx(),
		skipPullingInvalidTransactions:          c.skipPullingInvalidTransactions,
		fetcher:                                 c.Fetcher,
		idDeserializerFactory:                   c.idDeserializerFactory,
	}
    
    // 5.
	pvtdataToRetrieve, err := c.getTxPvtdataInfoFromBlock(block)
	
	// 6.Retrieve the private data.
	// RetrievePvtdata checks this peer's eligibility and then retreives from cache, transient store, or from a remote peer.
	retrievedPvtdata, err := pdp.RetrievePvtdata(pvtdataToRetrieve)
	blockAndPvtData.PvtData = retrievedPvtdata.blockPvtdata.PvtData
	blockAndPvtData.MissingPvtData = retrievedPvtdata.blockPvtdata.MissingPvtData

	// 7.commit block and private data
	commitStart := time.Now()
	err = c.CommitLegacy(blockAndPvtData, &ledger.CommitOptions{})
	c.reportCommitDuration(time.Since(commitStart))
	
	// 8.Purge transactions
	retrievedPvtdata.Purge()

	return nil
}
```

### 2.coordinator.CommitLegacy

core\committer\committer_impl.go

```go
// CommitLegacy commits blocks atomically with private data
func (lc *LedgerCommitter) CommitLegacy(blockAndPvtData *ledger.BlockAndPvtData, commitOpts *ledger.CommitOptions) error {
	// 1.Committing new block
	if err := lc.PeerLedgerSupport.CommitLegacy(blockAndPvtData, commitOpts); err != nil {
		return err
	}

	return nil
}
```



2. lc.PeerLedgerSupport.CommitLegacy

   core\ledger\kvledger\kv_ledger.go

```go
// CommitLegacy commits the block and the corresponding pvt data in an atomic operation
func (l *kvLedger) CommitLegacy(pvtdataAndBlock *ledger.BlockAndPvtData, commitOpts *ledger.CommitOptions) error {
	var err error
	block := pvtdataAndBlock.Block
	blockNo := pvtdataAndBlock.Block.Header.Number

	startBlockProcessing := time.Now()
    // 1.convertTxPvtDataArrayToMap
	if commitOpts.FetchPvtDataFromLedger {
		// when we reach here, it means that the pvtdata store has the
		// pvtdata associated with this block but the stateDB might not
		// have it. During the commit of this block, no update would
		// happen in the pvtdata store as it already has the required data.

		// if there is any missing pvtData, reconciler will fetch them
		// and update both the pvtdataStore and stateDB. Hence, we can
		// fetch what is available in the pvtDataStore. If any or
		// all of the pvtdata associated with the block got expired
		// and no longer available in pvtdataStore, eventually these
		// pvtdata would get expired in the stateDB as well (though it
		// would miss the pvtData until then)
		txPvtData, err := l.blockStore.GetPvtDataByNum(blockNo, nil)
		
		pvtdataAndBlock.PvtData = convertTxPvtDataArrayToMap(txPvtData)
	}

    // 2.ValidateAndPrepare
	logger.Debugf("[%s] Validating state for block [%d]", l.ledgerID, blockNo)
	txstatsInfo, updateBatchBytes, err := l.txtmgmt.ValidateAndPrepare(pvtdataAndBlock, true)
	
	elapsedBlockProcessing := time.Since(startBlockProcessing)

	startBlockstorageAndPvtdataCommit := time.Now()
    
    // 3.addBlockCommitHash
	logger.Debugf("[%s] Adding CommitHash to the block [%d]", l.ledgerID, blockNo)
	// we need to ensure that only after a genesis block, commitHash is computed
	// and added to the block. In other words, only after joining a new channel
	// or peer reset, the commitHash would be added to the block
	if block.Header.Number == 1 || l.commitHash != nil {
		l.addBlockCommitHash(pvtdataAndBlock.Block, updateBatchBytes)
	}

	logger.Debugf("[%s] Committing block [%d] to storage", l.ledgerID, blockNo)
	l.blockAPIsRWLock.Lock()
	defer l.blockAPIsRWLock.Unlock()
    
    // 4.CommitWithPvtData
	if err = l.blockStore.CommitWithPvtData(pvtdataAndBlock); err != nil {
		return err
	}
    
	elapsedBlockstorageAndPvtdataCommit := time.Since(startBlockstorageAndPvtdataCommit)

	startCommitState := time.Now()
	logger.Debugf("[%s] Committing block [%d] transactions to state database", l.ledgerID, blockNo)
	if err = l.txtmgmt.Commit(); err != nil {
		panic(errors.WithMessage(err, "error during commit to txmgr"))
	}
	elapsedCommitState := time.Since(startCommitState)

    // 5.History database
	// History database could be written in parallel with state and/or async as a future optimization,
	// although it has not been a bottleneck...no need to clutter the log with elapsed duration.
	if l.historyDB != nil {
		logger.Debugf("[%s] Committing block [%d] transactions to history database", l.ledgerID, blockNo)
		if err := l.historyDB.Commit(block); err != nil {
			panic(errors.WithMessage(err, "Error during commit to history db"))
		}
	}

	// 6.updateBlockStats
	l.updateBlockStats(
		elapsedBlockProcessing,
		elapsedBlockstorageAndPvtdataCommit,
		elapsedCommitState,
		txstatsInfo,
	)
	return nil
}
```

### 3.l.blockStore.CommitWithPvtData(pvtdataAndBlock)

core\ledger\ledgerstorage\store.go

```go
// CommitWithPvtData commits the block and the corresponding pvt data in an atomic operation
func (s *Store) CommitWithPvtData(blockAndPvtdata *ledger.BlockAndPvtData) error {
	blockNum := blockAndPvtdata.Block.Header.Number
	s.rwlock.Lock()
	defer s.rwlock.Unlock()

    // 1.
	pvtBlkStoreHt, err := s.pvtdataStore.LastCommittedBlockHeight()
	if pvtBlkStoreHt < blockNum+1 { // The pvt data store sanity check does not allow rewriting the pvt data.
		// when re-processing blocks (rejoin the channel or re-fetching last few block),
		// skip the pvt data commit to the pvtdata blockstore
		logger.Debugf("Writing block [%d] to pvt block store", blockNum)
		// If a state fork occurs during a regular block commit,
		// we have a mechanism to drop all blocks followed by refetching of blocks
		// and re-processing them. In the current way of doing this, we only drop
		// the block files (and related artifacts) but we do not drop/overwrite the
		// pvtdatastorage as it might leads to data loss.
		// During block reprocessing, as there is a possibility of an invalid pvtdata
		// transaction to become valid, we store the pvtdata of invalid transactions
		// too in the pvtdataStore as we do for the publicdata in the case of blockStore.
		pvtData, missingPvtData := constructPvtDataAndMissingData(blockAndPvtdata)
		if err := s.pvtdataStore.Commit(blockAndPvtdata.Block.Header.Number, pvtData, missingPvtData); err != nil {
			return err
		}
	} else {
		logger.Debugf("Skipping writing block [%d] to pvt block store as the store height is [%d]", blockNum, pvtBlkStoreHt)
	}

	if err := s.AddBlock(blockAndPvtdata.Block); err != nil {
		return err
	}

	if pvtBlkStoreHt == blockNum+1 {
		// we reach here only when the pvtdataStore was ahead
		// of blockStore during the store opening time (would
		// occur after a peer rollback/reset).
		s.isPvtstoreAheadOfBlockstore.Store(false)
	}

	return nil
}
```

### 4.fsBlockStore.AddBlock

```go
// AddBlock adds a new block
func (store *fsBlockStore) AddBlock(block *common.Block) error {
	// track elapsed time to collect block commit time
	startBlockCommit := time.Now()
    
    // 1.store.fileMgr.addBlock
	result := store.fileMgr.addBlock(block)
	
    elapsedBlockCommit := time.Since(startBlockCommit)

    // 2.store.updateBlockStats
	store.updateBlockStats(block.Header.Number, elapsedBlockCommit)

	return result
}
```

### 5.fsBlockStore.blockfileMgr.addBlock

common\ledger\blkstorage\fsblkstorage\blockfile_mgr.go

```go
func (mgr *blockfileMgr) addBlock(block *common.Block) error {
	bcInfo := mgr.getBlockchainInfo()
	if block.Header.Number != bcInfo.Height {
		return errors.Errorf(
		)
	}

	// Add the previous hash check - Though, not essential but may not be a bad idea to
	// verify the field `block.Header.PreviousHash` present in the block.
	// This check is a simple bytes comparison and hence does not cause any observable performance penalty
	// and may help in detecting a rare scenario if there is any bug in the ordering service.
	if !bytes.Equal(block.Header.PreviousHash, bcInfo.CurrentBlockHash) {
		return errors.Errorf(
		)
	}
    
    // 1.serializeBlock
	blockBytes, info, err := serializeBlock(block)
	
	blockHash := protoutil.BlockHeaderHash(block.Header)
    
	//Get the location / offset where each transaction starts in the block and where the block ends
	txOffsets := info.txOffsets
	currentOffset := mgr.cpInfo.latestFileChunksize

	blockBytesLen := len(blockBytes)
	blockBytesEncodedLen := proto.EncodeVarint(uint64(blockBytesLen))
	totalBytesToAppend := blockBytesLen + len(blockBytesEncodedLen)

	//Determine if we need to start a new file since the size of this block
	//exceeds the amount of space left in the current file
	if currentOffset+totalBytesToAppend > mgr.conf.maxBlockfileSize {
		mgr.moveToNextFile()
		currentOffset = 0
	}
    
	// 2.append blockBytesEncodedLen to the file
	err = mgr.currentFileWriter.append(blockBytesEncodedLen, false)
	if err == nil {
		//append the actual block bytes to the file
		err = mgr.currentFileWriter.append(blockBytes, true)
	}

	//Update the checkpoint info with the results of adding the new block
	currentCPInfo := mgr.cpInfo
	newCPInfo := &checkpointInfo{
		latestFileChunkSuffixNum: currentCPInfo.latestFileChunkSuffixNum,
		latestFileChunksize:      currentCPInfo.latestFileChunksize + totalBytesToAppend,
		isChainEmpty:             false,
		lastBlockNumber:          block.Header.Number}
    
	// 3.save the checkpoint information in the database
	if err = mgr.saveCurrentInfo(newCPInfo, false); err != nil {
		truncateErr := mgr.currentFileWriter.truncateFile(currentCPInfo.latestFileChunksize)
		if truncateErr != nil {
			panic(fmt.Sprintf("Error in truncating current file to known size after an error in saving checkpoint info: %s", err))
		}
		return errors.WithMessage(err, "error saving current file info to db")
	}

	//Index block file location pointer updated with file suffex and offset for the new block
	blockFLP := &fileLocPointer{fileSuffixNum: newCPInfo.latestFileChunkSuffixNum}
	blockFLP.offset = currentOffset
	// shift the txoffset because we prepend length of bytes before block bytes
	for _, txOffset := range txOffsets {
		txOffset.loc.offset += len(blockBytesEncodedLen)
	}
	//save the index in the database
	if err = mgr.index.indexBlock(&blockIdxInfo{
		blockNum: block.Header.Number, blockHash: blockHash,
		flp: blockFLP, txOffsets: txOffsets, metadata: block.Metadata}); err != nil {
		return err
	}

	//update the checkpoint info (for storage) and the blockchain info (for APIs) in the manager
	mgr.updateCheckpoint(newCPInfo)
	mgr.updateBlockchainInfo(blockHash, block)
    
	return nil
}
```

### 6.mgr.currentFileWriter.append

common\ledger\blkstorage\fsblkstorage\blockfile_rw.go

```go
func (w *blockfileWriter) append(b []byte, sync bool) error {
	_, err := w.file.Write(b)
	if err != nil {
		return err
	}
	if sync {
		return w.file.Sync()
	}
	return nil
}
```

