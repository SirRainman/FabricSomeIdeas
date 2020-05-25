# 提高Hyperledger fabric应用程序的性能

参考文献：[阿里BaaS](https://help.aliyun.com/document_detail/141448.html)

交易的生命周期：

1. SDK 生成 Proposal，其中包含调用链码的相关参数等信息
2. SDK 将 Proposal 发送到多个不同的 peer 节点
   1. peer 根据 Proposal 的信息，调用用户上传的链码
   2. 链码处理请求，将请求转换为对账本的读集合和写集合
   3. peer 对读集合和写集合进行签名，并将 ProposalResponse 返回给 SDK
   4. （注：如果是查询就到这里戛然而止了）
3. SDK 收到多个 peer 节点的 ProposalResponse，并将读集合与写集合和不同节点的签名拼接在一起，组成 Envelope
4. SDK 将 Envelope 发送给 orderer 节点，并监听 peer 节点的块事件
5. orderer 节点收到足够的 Envelope 后，生成新的区块，并将区块广播给所有 peer 节点
6. 各个 peer 节点对收到区块进行验证，并向 SDK 发送新收到的区块和验证结果
7. SDK 根据事件中的验证结果，判断交易是否成功上链

## SmartContract设计

链码设计的优化目标：

1. 降低链码对每笔交易的处理时间
2. 可以让链码在单位时间内并发的处理多笔交易

### 1 避免Key冲突

在 Fabric 区块链账本中，数据是以 KV 的形式存储的，链码可以通过 `GetState`、`PutState` 等方法对账本数据进行操作。

1. 在committer进行验证交易时，每一个 Key 都有一个版本号，如果有两笔不同的交易对同一个版本的 Key 做不同的修改，其中一笔交易会因为 Key 冲突而失败。

   fabric-sample/high-throughput：以增量的形式对同一个Key进行写入的多笔不同交易来写入数据，减少不同交易对同一个 Key 进行写入的频率。避免间隔过短，即避免对Key进行频繁写入。

2. 在 orderer 产生区块后，交易的顺序也确定了，由于此时第一笔交易已经让 Key 的版本发生了改变，当第二笔交易再次对 Key 进行修改时，就会失败了。

   Sigmod2019-Fabric++：在oderer调整块内的transaction的顺序



验证交易失败缺点：

1. Fabric 中的区块会包含非法的交易，如果业务产生了大量因为 Key 冲突而失败的交易，这些交易也会被记入各个节点的账本，占用节点的存储空间。
2. 同时由于冲突的原因，并行的交易很多会失败，不但会导致 SDK 的成功 TPS 大幅下降，失败的交易还会占用网络的吞吐量。

### 2 减少Stub读取和写入账本的次数

Fabric 中的链码与 peer 节点之间的通信与 SDK 和区块链节点的通信类似，也是通过 GRPC 来进行的。

1. 当在链码中调用查询、写入账本的接口时（例如 `GetState`、`PutState` 等），链码容器发送 GRPC 请求给 peer 节点。
2. Peer查询本地的账本，然后把结果返回给链码容器。
3. 链码容器收到结果后，再返回到链码的逻辑中。

当链码在一次 `Query/Invoke` 中调用了多次账本的查询或写入接口时，会产生一定的网络通信成本和延迟，这对网络的整体吞吐率会有一定的影响。



在设计应用时，应尽量减少一次 `Query/Invoke` 中的查询和写入账本的次数。

- 在一些对吞吐有很高要求的特殊场景下，可以在业务层对多个 Key 及对应的 Value 进行合并，将多次读写操作变成一次操作。

### 3 减少链码的运算量

当链码被调用时，会在 peer 的账本上挂一把读锁，保证链码在处理该笔交易时，账本的状态不发生改变，当新的区块产生时，peer 将账本完全锁住，直到完成账本状态的更新操作。

如果链码在处理交易时花费了大量时间，会让 peer 验证区块等待更长的时间，从而降低整体的吞吐量。

在编写链码时，链码中最好只包含简单的逻辑、校验等必要的运算，将不太重要的逻辑放到链码外进行。

## Java SDK设计

### 1 复用channel && client 对象

SDK 在初始化 channel 对象阶段会有一定的资源及时间消耗，同时每一个 channel 对象都会建立自己的事件监听连接，向 peer 获取最新的区块及验证结果，从而消耗较多的网络带宽。

1. 应用程序在针对一个业务通道进行操作的时候，如果创建过多 channel 对象，可能会影响业务的响应时间，甚至会由于 TCP 连接数过多而引发业务阻塞。
2. 在应用程序中，如果针对一个业务通道频繁发送交易，则创建该通道的第一个 channel 对象后应尽量复用。
3. 如果 channel 对象长时间闲置，可以使用`channel.shutdown(true)`释放资源。

通过 HFCAClient 产生本地用户时，其中包含了用户私钥的生成和`Enroll`操作，也有一定的时间消耗。

### 2 只将交易发送给必要的背书节点

假设每个组织都会有2个 peer 背书节点，如果一个业务通道内有 N 个组织，在使用 SDK 提交 Proposal 的时候，会默认发送给所有的 peer 背书节点（2*N个）。

- 这时每个 peer 节点都要处理一遍Proposal，影响整体的吞吐量。
- 当个别peer处理缓慢时，会拖慢交易的响应时间。

如果用户不需要在应用端对各个 peer 返回的读写集做一致性验证，可根据链码的背书策略选择性地提交 Proposal 到必要的 peer 节点，这样可节约 peer 的计算资源，提高性能。例如：

- 如果链码背书策略为 `OR ('org1MSP.peer' , 'org2MSP.peer' , 'org3MSP.peer')`，则应用可选择6个 peer 节点中的任意一个，提交 proposal 获取背书返回即可；
- 如果链码背书策略为 `OutOf(2 , 'org1MSP.peer' , 'org2MSP.peer' , 'org3MSP.peer')`，则应用可选择6个 peer 节点中的任意2个，且来自不同组织，提交 proposal 获取背书返回即可。



方法二：也可以使用 Fabric 提供的 discovery 功能，自动选择必要的 peer 节点发送 Proposal。

```java
Channel.DiscoveryOptions discoveryOptions = Channel.DiscoveryOptions.createDiscoveryOptions();

discoveryOptions.setEndorsementSelector(ServiceDiscovery.EndorsementSelector.ENDORSEMENT_SELECTION_RANDOM); // 随机选择一个满足背书策略的组合发送请求

discoveryOptions.setForceDiscovery(false); // 使用 discovery 的缓存，默认2分钟刷新一次
discoveryOptions.setInspectResults(true); // 关闭 SDK 的背书策略检查，由我们的逻辑进行判断

Collection<ProposalResponse> transactionPropResp = channel.sendTransactionProposalToEndorsers(transactionProposalRequest, discoveryOptions);
```



### 3 异步等待必要的节点事件

应用端将 peer 返回的 proposal 读写集发送到 oderer 后，fabric 会进行一系列的排序-出块-验证-写入等操作，根据通道的出块配置，最终交易写入会有一定的延迟。

Java SDK 中默认会等待所有 `eventSource` 为 true 的节点事件，当所有节点验证均通过时，才会返回成功。这在一些业务场景下是可以优化的，一般选择自己所在组织的任意一个 peer 节点接受事件可以满足绝大多数场景下的需求。

- Java SDK 的 `channel.sendTransaction` 方法返回 `CompletableFuture`，应用可使用多线程操作，当一个线程 `sendTransaction` 到 orderer 后则继续处理其他交易，另一个线程监听到 `TransactionEvent` 后进行相应的业务处理。
- Java SDK 还提供了 `NOfEvents` 类，来控制 events 的接收策略，以判断发送到 orderer 的交易是否最终成功。建议将 `NOfEvents` 设为1，也就是只要任意一个节点返回 event 即可。应用不需要等待每一个peer都发出 `TransactionEvent` 才算交易成功。
- 如果应用不需要处理 transaction event，可通过 `Channel.NOfEvents.createNoEvents()` 创建 `nofNoEvents` 这种特殊的 `NOfEvents` 对象。将这个对象配置进 `TransactionOptions` 后，Orderer接收到应用发送的交易后会立即返回 `CompletableFuture`，但 `TransactionEvent` 会被置为null。

示例：通过使用 `NOfEvents` 来配置当收到任意一个节点验证通过的事件时，即返回成功：

```java
Channel.TransactionOptions opts = new Channel.TransactionOptions();
Channel.NOfEvents nOfEvents = Channel.NOfEvents.createNofEvents();
Collection<EventHub> eventHubs = channel.getEventHubs();
if (!eventHubs.isEmpty()) {
    nOfEvents.addEventHubs(eventHubs);
}
nOfEvents.addPeers(channel.getPeers(EnumSet.of(Peer.PeerRole.EVENT_SOURCE)));
nOfEvents.setN(1);
opts.nOfEvents(nOfEvents);
channel.sendTransaction(successful, opts).thenApply(transactionEvent -> {
    logger.info("Orderer response: txid" + transactionEvent.getTransactionID());
    logger.info("Orderer response: block number: " + transactionEvent.getBlockEvent().getBlockNumber());
    return null;
}).exceptionally(e -> {
    logger.error("Orderer exception happened: ", e);
    return null;
}).get(60, TimeUnit.SECONDS);
```



除了使用 `NOfEvents`来配置交易成功的验证方式，也可以在配置文件 `connection.json` 中指定接收哪些 peer 节点的 `eventSource`，下属示例中，只接收 peer1 节点的 `eventSource` 事件:

```json
{
    "channels": {
        "mychannel": {
            "orderers": [
                "orderer.example.com"
            ],
            "peers": {
                "peer0.org1.example.com": {
                    "endorsingPeer": true,
                    "chaincodeQuery": true,
                    "ledgerQuery": true,
                    "eventSource": true,
                    "discover": true
                },
                "peer0.org2.example.com": {
                    "endorsingPeer": true,
                    "chaincodeQuery": true,
                    "ledgerQuery": true,
                    "discover": true
                }
            }
        }
    },
}
```

