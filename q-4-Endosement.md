[Endorsement官方文档](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/endorsement-policies.html#)

# Setting chaincode-level endorsement policies

每个链码都有一个背书策略与之相关联，该背书策略适用于此链码中定义的所有智能合约。背书策略非常重要，它指明了区块链网络中哪些组织必须对一个既定智能合约所生成的交易进行签名，以此来宣布该交易**有效**。

Chaincode-level endorsement policies are agreed to by channel members when they approve a chaincode definition for their organization. 

A sufficient number of channel members need to approve a chaincode definition to meet the `Channel/Application/LifecycleEndorsement` policy, which by default is set to a majority of channel members, before the definition can be committed to the channel. 

Once the definition has been committed, the chaincode is ready to use. Any invoke of the chaincode that writes data to the ledger will need to be validated by enough channel members to meet the endorsement policy.

You can specify an endorsement policy for a chainocode using the Fabric SDKs. For an example, visit the [How to install and start your chaincode](https://hyperledger.github.io/fabric-sdk-node/master/tutorial-chaincode-lifecycle.html) in the Node.js SDK documentation. 

You can also create an endorsement policy from your CLI when you approve and commit a chaincode definition with the Fabric peer binaries by using the `—-signature-policy` flag.

In addition to the specifying an endorsement policy from the CLI or SDK, a chaincode can also use policies in the channel configuration as endorsement policies. You can use the `–channel-config-policy`flag to select a channel policy with format used by the channel configuration and by ACLs. If you do not specify a policy, the chaincode definition will use the `Channel/Application/Endorsement` policy by default, which requires that a transaction be validated by a majority of channel members. 

# Setting collection-level endorsement policies

[介绍 Collective-level endorsement policies](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/endorsement-policies.html#)

[怎么在私有数据中实现？](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/private-data-arch.html)



---

# 还有两部分我没看