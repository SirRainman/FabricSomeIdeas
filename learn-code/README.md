# 源码架构

## 组成

fabric
|-- bccsp
|-- build
|-- ci
|-- cmd
|-- common
|-- core
|-- discovery
|-- docs
|-- gossip
|-- idemix
|-- images
|-- integration
|-- internal
|-- msp
|-- orderer
|-- pkg
|-- protoutil
|-- sampleconfig
|-- scripts
|-- vagrant
`-- vendor

## 规范

## 一些缩写

> 引用：https://blog.csdn.net/idsuf698987/article/details/74912362
>
> MSP:  Membership service provider 会员服务提供者
> BCCSP: blockchain（前两个字母BC） cryptographic service provider 区域链加密服务提供者
> ab: atomic broadcast原子（操作）广播
> Spec: Specification，规格标准，详细说明
> KV: key-value 键-值
> CDS: Chaincode Deployment Spec
> CIS: Chaincode Invocation Spec
> mgmt: management
> SW: software-based
> AB: Atomic Broadcast
> GB: genesis block，创世纪的block，也就是区域链中的第一个块
> CC或cc: chaincode
> SCC或scc: system chaincode
> cscc: configer system chaincode
> lscc: lifecycle system chaincode
> escc: endorser system chaincode
> vscc: validator system chaincode
> qscc: querier system chaincode
> alg: algorithm 算法
> mcs: mspMessage Crypto Service
> mock: 假装，学样子，模仿的意思，基本上是服务于xxx_test.go的，即用于测试的
> Gossip: 一种使分布结点达到状态最终一致的算法
> attr: attribute
> FsBlockStore: file system block store
> vdb: versioned database 也就是状态数据库
> RTEnv: runtime environment运行环境
> pkcs11: pcks#11，一种公匙加密标准，有一套叫做Cryptoki的接口，是一组平台设备无关的API
> sa: Security Advisor
> FSM: finite state machine 有限状态机
> FS: filesystem 文件系统
> blk: block
> cli: command line interface 命令行界面
> CFG: FABRIC_CFG_PATH中的，应该是config的意思
> mgr: manager
> cpinfo: checkpoint information，检查点信息
> DevMode: development mode，开发模式
> Reg: register，注册，登记
> hdr: header
> impl: implement
> oid: ObjectIdentifier，对象标识符
> ou或OU: organizational unit
> CRL: certificate revocation list，废除证书列表
> prop: proposal，申请，交易所发送的申请
> ACL: Access Control List，访问控制列表
> rwset: read/write set，读写集
> tx，Tx: transaction，交易
> CSP: cryptographic service provider，BCCSP的后三个字母，加密服务提供者
> opt: option，选项
> opts: options，多个选项
> SKI: 当前证书标识，所谓标识，一般是对公匙进行一下hash
> AKI: 签署方的SKI，也就是签署方的公匙标识
> HSM: Hardware Security Modules
> ks: KeyStore，Key存储，这个key指的是用于签名的公匙私匙
> oid: OBJECT IDENTIFIER，对象身份标识

# 读源码过程中需要解决的问题

1. peer容器、链码容器之间的调用关系
2. peer容器、链码容器、couch db容器之间的调用关系
3. 用户查询的完整流程
4. peer是怎样利用gossip协议存区块的？
5. 

