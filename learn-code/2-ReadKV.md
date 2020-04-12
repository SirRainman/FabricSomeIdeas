# 读取k-v state db 源码解析

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

