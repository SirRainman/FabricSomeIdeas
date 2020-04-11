# 写在前面

## 需要解决的问题

多个联盟之间，需要进行信息共享。当本联盟需要访问其他联盟中的信息时，只需要访问本联盟中其他联盟提供的接口就可以了。

接口的体现形式：

1. 在本联盟中设置访问其他链盟的客户端，通过其他联盟的客户端访问其他联盟中的信息。
2. 在本联盟中设置其他联盟的轻节点。



之所以是通过其他联盟的客户端，或通过轻节点的方案，是因为：

1. 一个联盟可能并不希望将自己的数据，放到其他联盟中
2. 若一个联盟需要访问其他联盟中的数据，就要维护其他联盟中的所有数据，成本太大。



因此，问题最终转化为

* 在不存储其他联盟中的数据的情况下，如果快速且有效的查询其他联盟中的信息



## 其他联盟客户端方案分析：

优点：

1. 在授权之后，可以得到某个证照的具体信息，信息完整度高。

缺点：

1. 时延长：在查询过程中，有两个查询周期（客户端在本联盟中查询不到结果后，通过其他联盟的客户端进行二次查询）

## 轻节点的方案分析：

优点：

1. 只需要验证其他联盟中是否有一笔历史交易就可以了，效率高。

缺点：

1. 只可以返回是否具有其他联盟中，是否存在一笔我们想要验证的历史交易。

## 需要解决的问题

Fabric中没有轻节点的概念，fabric中节点可以分为：client/peer/orderer。

1. 概念性问题：如果要在fabric中设计一个轻节点，这个轻节点的类型是不是更偏向于peer节点？



轻节点的功能是：保存一个链中所有节点的区块头部分，不保存区块体部分。可以验证一笔交易是否存在于链上。

以太坊轻节点问题：

1. 当需要验证某一笔交易是否存在于链上时，以太坊的轻节点是怎么做的？
2. 为什么轻节点只保存区块头？他保存区块头有什么作用？
3. 以太坊轻节点只保存区块头部分，轻节点怎么确认该笔交易在哪一个区块上？
4. 以太坊轻节点在确认某一笔交易在链上时，他需要向全节点发送一个对该历史交易的验证请求，然后全节点该笔交易在某一个区块上的merkel proof返回给轻节点，轻节点再根据Merkel proof进行验证。
   在这个过程中，轻节点还需要向全节点发送一次验证请求。如果迁移到了fabric中，该轻节点还需要向peer发送一次验证请求的话，那么这个方案和直接像peer节点进行查询一笔历史交易的代价有什么不同？





# Fabric交易的生命周期

![](../images/peers-app.png)

app提交交易流程：

1. Hyperledger Fabric SDK 通过 APIs 使应用程序能够连接到 peers。
   application生成交易提案proposal，其中包含本次交易要调用的合约标识、合约方法和参数信息以及客户端签名等。
2. 在交易提案能够被网络所接受之前，app必须得到背书策略所指定的peers的背书。peer使用proposal来调用 chaincode，从而生成交易的proposal response。
   1. peer 根据 proposal 的信息，调用用户上传的链码
   2. 链码处理请求，将请求转化位对账本的读集合与写集合。
   3. peer 对读集合和写集合进行签名，并将proposal response 返回给app SDK。
3. application接收到了一定数量经过背书后的proposal responses，并将读集合与写集合和不同节点的签名拼接在一起，组成envelope（在这里envelope才是一个真正的tx）。
4. application提交envelope到order节点后，sdk并监听peer节点的块事件。
   1. 在orderer集群中envelope会被排序，当orderer收集到足够多的envelope后，生成新的区块。
   2. orderer将区块广播到peer集群中的分布式账本中。
      注：不是每个 Peer 节点都需要连接到一个排序节点，只需要一个peer主节点连接到order就可以了，然后Peer中的主节点可以使用 gossip 协议将区块关联到其他节点。（gossip的另外一个应用就是组织间的peer的相互通信）
   3. 每个peer节点将独立地以确定的方式验证区块，以确保账本保持一致。具体来说，通道中每个peer节点都将验证区块中的每个交易，以确保得到了所需组织的节点背书，也就是peer节点的背书和背书策略相匹配，并且不会因最初认可该事务时可能正在运行的其他最近提交的事务而失效。无效的交易仍然保留在排序节点创建的区块中，但是节点将它们标记为无效，并且不更新账本的状态。
5. 在peer集群收到区块并进行验证之后，peer向sdk发送新收到的区块和验证结果。
6. sdk根据事件的验证结果，判断交易是否成功上链。

## 现阶段需要解决的问题：

需要弄清fabric中，查询/某一笔交易的过程。

1. fabric中可以直接查询一个历史交易吗？
   fabric是可以通过txid查询到一个交易的具体信息。具体是通过调用部署在peer节点上的系统链码qscc中的getTransactionByID()方法实现
2. fabric查询一个对象的信息是从key-value数据库中得到的吗？
   是的，一般是使用couchdb的语法，从couchdb的数据库中读取到一个对象的最终状态。



# 源码解析

## 1 客户端提出交易请求

byfn.sh-networkUp() 中在cli容器中执行交易脚本

```yaml
docker exec cli scripts/script.sh $CHANNEL_NAME $CLI_DELAY $CC_SRC_LANGUAGE $CLI_TIMEOUT $VERBOSE $NO_CHAINCODE
```

cli执行script.sh，下面向peer0.org1发送了一个查询请求

```yaml
chaincodeQuery 0 1 100
```

utils.sh-chaincodeQuery实现

```yaml
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}' >&log.txt
```

主要原理是调用了bin/peer工具生成一个proposal

## 2 peer 使用proposal 调用链码

本例中的源码在chaincode/abstore/go/abstore.go中，从下面可以看到链码被调用后，返回值为一个proposal response

```go
func (t *ABstore) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	fmt.Println("ABstore Invoke")
	function, args := stub.GetFunctionAndParameters()
	if function == "invoke" {
		// Make payment of X units from A to B
		return t.invoke(stub, args)
	} else if function == "delete" {
		// Deletes an entity from its state
		return t.delete(stub, args)
	} else if function == "query" {
		// the old "Query" is now implemtned in invoke
		return t.query(stub, args)
	}

	return shim.Error("Invalid invoke function name. Expecting \"invoke\" \"delete\" \"query\"")
}
```

在chaincode/abstore/go/vendor/github.com/hyperledger/fabric-protos-go/peer/proposal_response.pb.go中，我们可以看到返回值proposal response的格式为

```go
type Response struct {
	// A status code that should follow the HTTP status codes.
	Status int32 `protobuf:"varint,1,opt,name=status,proto3" json:"status,omitempty"`
	// A message associated with the response code.
	Message string `protobuf:"bytes,2,opt,name=message,proto3" json:"message,omitempty"`
	// A payload that can be used to include metadata with this response.
	Payload              []byte   `protobuf:"bytes,3,opt,name=payload,proto3" json:"payload,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```

### 2.1 生成读写集

由'{"Args":["query","a"]}' 可知调用函数为query，args为a。

```go
// query callback representing the query of a chaincode
func (t *ABstore) query(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	var A string // Entities
	var err error

	if len(args) != 1 {
		return shim.Error("Incorrect number of arguments. Expecting name of the person to query")
	}

	A = args[0]

	// Get the state from the ledger
	Avalbytes, err := stub.GetState(A)
	if err != nil {
		jsonResp := "{\"Error\":\"Failed to get state for " + A + "\"}"
		return shim.Error(jsonResp)
	}

	if Avalbytes == nil {
		jsonResp := "{\"Error\":\"Nil amount for " + A + "\"}"
		return shim.Error(jsonResp)
	}

	jsonResp := "{\"Name\":\"" + A + "\",\"Amount\":\"" + string(Avalbytes) + "\"}"
	fmt.Printf("Query Response:%s\n", jsonResp)
	return shim.Success(Avalbytes)
}
```

在query()中我们可以看到关键语句主要有两句：

```go
Avalbytes, err := stub.GetState(A) //获取对象A的value值

shim.Success(Avalbytes) //返回proposal response
```

在chaincode/abstore/go/vendor/github.com/hyperledger/fabric-chaincode-go/shim/stub.go中，我们可以看到getState的具体实现：

```go
// GetState documentation can be found in interfaces.go
func (s *ChaincodeStub) GetState(key string) ([]byte, error) {
	// Access public data by setting the collection to empty string
	collection := ""
	return s.handler.handleGetState(collection, key, s.ChannelId, s.TxID)
}
```

其中chaincodestub的结构为：

```go
type ChaincodeStub struct {
	TxID                       string
	ChannelId                  string
	chaincodeEvent             *pb.ChaincodeEvent
	args                       [][]byte
	handler                    *Handler
	signedProposal             *pb.SignedProposal
	proposal                   *pb.Proposal
	validationParameterMetakey string

	// Additional fields extracted from the signedProposal
	creator   []byte
	transient map[string][]byte
	binding   []byte

	decorations map[string][]byte
}
```

其中Handler的主要定义在chaincode/abstore/go/vendor/github.com/hyperledger/fabric-chaincode-go/shim/handler.go中，

```go
// Handler handler implementation for shim side of chaincode.
type Handler struct {
	// serialLock is used to prevent concurrent calls to Send on the
	// PeerChaincodeStream. This is required by gRPC.
	serialLock sync.Mutex
	// chatStream is the client used to access the chaincode support server on
	// the peer.
	chatStream PeerChaincodeStream

	// cc is the chaincode associated with this handler.
	cc Chaincode
	// state holds the current state of this handler.
	state state

	// Multiple queries (and one transaction) with different txids can be executing in parallel for this chaincode
	// responseChannels is the channel on which responses are communicated by the shim to the chaincodeStub.
	// need lock to protect chaincode from attempting
	// concurrent requests to the peer
	responseChannelsMutex sync.Mutex
	responseChannels      map[string]chan pb.ChaincodeMessage
}
```

在这个文件中，我们可以看到handler的handleGetState()方法的实现，其中如果执行成功会返回，responseMsg.Payload

```go

// handleGetState communicates with the peer to fetch the requested state information from the ledger.
func (h *Handler) handleGetState(collection string, key string, channelId string, txid string) ([]byte, error) {
	// Construct payload for GET_STATE
	payloadBytes := marshalOrPanic(&pb.GetState{Collection: collection, Key: key})

	msg := &pb.ChaincodeMessage{Type: pb.ChaincodeMessage_GET_STATE, Payload: payloadBytes, Txid: txid, ChannelId: channelId}
	responseMsg, err := h.callPeerWithChaincodeMsg(msg, channelId, txid)
	if err != nil {
		return nil, fmt.Errorf("[%s] error sending %s: %s", shorttxid(txid), pb.ChaincodeMessage_GET_STATE, err)
	}

	if responseMsg.Type == pb.ChaincodeMessage_RESPONSE {
		// Success response
		return responseMsg.Payload, nil
	}
	if responseMsg.Type == pb.ChaincodeMessage_ERROR {
		// Error response
		return nil, fmt.Errorf("%s", responseMsg.Payload[:])
	}

	// Incorrect chaincode message received
	return nil, fmt.Errorf("[%s] incorrect chaincode message %s received. Expecting %s or %s", shorttxid(responseMsg.Txid), responseMsg.Type, pb.ChaincodeMessage_RESPONSE, pb.ChaincodeMessage_ERROR)
}
```

### 2.2将读写集转化为proposal response返回

