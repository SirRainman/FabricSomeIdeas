# 0 解决的问题

提高交易吞吐量：3000 -> 20000 -> *maybe: 50000

Fabric 中的性能瓶颈---共识机制&&验证机制，通过优化排序、计算和I/O开销来提升性能。



通过分析Fabric transaction flow，可以发现性能瓶颈：

1. 共识交易顺序时网络开销太大

   解决方案：只传送txID

2. 节点验证交易时串行验证

   解决方案：并行化验证交易请求&&区块

3. 验证交易的读写集需要快速访问world state

   解决方案：将world state放到内存中

4. 交易流程不需要区块链日志

   问：这个日志是什么日志？

   解决方案：在交易流程结束时将其存储到专用存储和数据分析服务器

5. 当某一结点同时担任提交者和背书者的角色时，他们会争夺计算资源

   解决方案：将不同角色的任务交给不同的硬件进行处理，减少了资源竞争问题，还可以针对不同的任务进行定制化硬件。

6. 

# 1 技术路线

背景知识：

Commit peer在收到来自orderer的block后：

1. 验证收到消息的合法性
2. 验证块中每个交易的区块头和签名
3. 验证交易的读写集
4. 更新世界状态
5. 将区块链日志存储在文件系统中



## 1.1 分离元数据

1. **从数据中分离元数据**：Fabric进行达成对交易的顺序的共识时，使用交易的ID进行排序，但是传入的参数却是整笔交易，事实上只需要传入每笔交易的ID就好了。从而减少了网络传输的数据量，减少了占用缓冲区的大小，从而提高吞吐量。

   问题：交易的顺序达成之后，其他的orderers只有txID，那他们怎么存区块？

## 1.2 并行和缓存

1. **并行和缓存**：Fabric的节点进行区块和交易头的验证（包括检查交易发起者的权限，执行认可策略和语法验证），其中某几步是可以并行处理计算的，但实际上使用串行化处理方式。作者大胆的缓存多个区块，将可以并行化的步骤独立了出来，使用线程池技术完成对客户端请求的处理。

为什么要进行缓存？

- Fabric使用gRPC在网络中让节点进行通信。
- Protocol Buffers被用来进行序列化。
- 在不同验证层次，都需要进行对序列化后的区块进行解析的操作，因为解析的开销很大，所以作者决定把第一次解析后的区块放到缓冲区中。

缓存区块：使用环形缓冲区缓冲区块，当一个区块被解析之后，就把他添加到环形缓冲区中，直到改区块被添加到链上。

## 1.3 快速数据访问

1. **利用内存的结构在关键路径上实现快速数据访问**：Fabric所维护的世界状态是通过KV数据库实现的，作者认为这不够快，他将世界状态放到了内存的hash table中，可以实现快速的数据访问。作者认为，虽然这种方式在数据持久化上存在短板，但这种牺牲是可以通过区块链本身的分布式账本的优势进行补充。

痛点：在验证每笔交易的读写集时，因为设计到了状态修改，因此必须串行的读取和修改来保证数据一致性，所以在读取world state时，必须要快。

## 1.4 资源分离

1. **资源分离**：当某一结点同时担任提交者和背书者的角色时，他们会争夺资源，作者将他们均摊到两个节点上。

Commit peer执行验证区块，然后将合法区块发送给Endorser Peer群集，这些Endorser Peer仅将区块涉及到的更改应用于其世界状态而无需进一步验证这些区块。此步骤允许我们释放Peer的资源。

## 1.5 分布式存储

1. 分布式存储：当一个节点存储的区块太大时，将区块链分开存储到不同的节点上
