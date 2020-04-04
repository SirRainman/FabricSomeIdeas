[Policy官方文档](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/policies/policies.html#what-is-a-policy)

# 什么是 policy？

policy 是一系列的**规则**，这些规则**定义了一些情况在什么样的条件下会达成怎样的结果**。

fabric policy则定义了当对区块链网络，channel（添加或删减channel中的member，改变block的结构等）或者smart contract进行改动时，**members是怎样达成一致的决定，以及达成什么样的决定**。

# 为什么需要policy？

因为fabric相较于其他公链的区块链平台而言，他是一个permissioned chain，只要该平台的节点们达成一致就可以修改policy，而其他的平台不可以随便的更改policy，即便是要改policy也很困难。通过这种方式可以比较方便的更改policy。

- Policy定义了哪些org可以加入到区块链网络中，哪些org可以修改一个区块链网络。
- Policy定义了哪些org可以访问一个区块链网络中的资源（如chaincode等）。
- Policy定义了当一个想要修改区块链网络的提案（如修改channel，修改chaincode）提出时，该提案必须由多少个org同意才能被通过。

另外就是没有规矩不成方圆，policy定义的是规则，按规章走总没错。

---

# 怎么实现policy？

在Fabric的不同的功能层面中，几乎都有policy的影子。

![](./images/FabricPolicyHierarchy.png)

## Policy domains

![](./images/FabricPolicyHierarchy-2.png)

The domains

- extend different **privileges and roles** to different organizations, by allowing the founders of the ordering service the ability to establish the initial rules and membership of the consortium. 
- allow the **organizations** that join the consortium to 
  - **create** private application channels, 
  - **govern** their own business logic, 
  - and **restrict access** to the data that is put on the network.



The **system channel configuration** and a portion of each **application channel configuration** provides the **ordering organizations** control over :

- which organizations are members of the consortium, 

- how blocks are delivered to channels, 
- and the consensus mechanism used by the nodes of the ordering service.



The **system channel configuration** provides members of the consortium 

- the ability to **create channels**. 



**Application channels** and **ACLs** are the mechanism that consortium organizations use to 

- add or remove members from a channel 
- and restrict access to data and smart contracts on a channel.

## System Channel

1. fabric区块链网络的启动开始于ordering system channel。
2. 对于一项ordering service而言，它必须有一个ordering system channel，并且这个ordering system channel 必须是第一个被创建的。
3. ordering system channel 中包含了属于ordering service的一些组织。
4. Ordering system channel 的configuration blocks 中的policy定义了：
   - 共识协议
   - 怎样产生新区块
   - 哪些consortium联盟中的member可以创建新的channel。

## Application Channel

Application channels 用来提供consortium联盟中各个org组织间的私密通信。

Application channel 中的policy定义了：

1. 在channel中添加或删除一个member的能力。
2. 在提交chaincode时，哪些组织必须同意该chaincode。

当一个Application channel 被创建时，他会继承来自system channel的全部的配置信息，这些信息可以在每个不同的application channel中被重新定义。

## ACLs - access control lists

[更多的案例在这里](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/access_control.html)

ACLs可以通过将资源与现有的policy联系在一起的方式

- 实现资源的权限控制(注意这里是访问资源的权限，如 system chaincode中的一些函数, "GetBlockByNumber()")，又如对通道的访问与管理权限等。

ACL 具体是指定义在application channel configuration配置文件中的一些policy，通过这些policy实现对资源的控制。

例：configtx-just-for-example.yaml

```yaml
Application: &ApplicationDefaults
	ACLs: &ACLsDefault
        # This section provides defaults for policies for various resources
        # in the system. These "resources" could be functions on system chaincodes
        # (e.g., "GetBlockByNumber" on the "qscc" system chaincode) or other resources
        # (e.g.,who can receive Block events). This section does NOT specify the resource's
        # definition or API, but just the ACL policy for it.
        #
        # Users can override these defaults with their own policy mapping by defining the
        # mapping under ACLs in their channel definition

        #---New Lifecycle System Chaincode (_lifecycle) function to policy mapping for access control--#

        # ACL policy for _lifecycle's "CheckCommitReadiness" function
        # These ACLs define that access to _lifecycle/CheckCommitReadiness is restricted 
        # to identities satisfying the policy defined at the canonical 
        # path /Channel/Application/Writers
        _lifecycle/CheckCommitReadiness: /Channel/Application/Writers

        # ACL policy for _lifecycle's "CommitChaincodeDefinition" function
        _lifecycle/CommitChaincodeDefinition: /Channel/Application/Writers

        # ACL policy for _lifecycle's "QueryChaincodeDefinition" function
        _lifecycle/QueryChaincodeDefinition: /Channel/Application/Readers
		
		# 还有很多，就不举例了
		
    # Organizations is the list of orgs which are defined as participants on
    # the application side of the network
    Organizations:

    # Policies defines the set of policies at this level of the config tree
    # For Application policies, their canonical path is
    #   /Channel/Application/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        # The number of organizations that need to approve a definition before it can be successfully committed 
        # to the channel is governed by the Channel/Application/LifecycleEndorsement policy.
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
        Endorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"

    Capabilities:
        <<: *ApplicationCapabilities
```

## Smart contract endorsement policies

在一个chaincode package中定义的所有的smart contracts，每个smart contract 都有一个背书策略，这个背书策略规定：

- 为了使某一个tx合法，必须有哪些peer 执行该tx 和 判定该tx合法。

因此，背书策略通过一个org中的peer定义了：

- 哪些peer必须对一个tx proposal的执行进行背书。

## Modification policies

Modification policies 定义了：

- policy是怎样更新的，modification policy 规定了这个更新请求必须经过哪些identity的同意才能生效。

因此，每个channel configuration中都有一个指针，该指针指向一个modification policy配置文件，因为这个modification policy规定了该channel的policy是怎样更新的。

# 实践：fabric policy

If you want to change anything in Fabric, the policy associated with the resource describes **who** needs to approve it. 

签名的方法有两种：In Hyperledger Fabric, explicit sign offs in policies are expressed using the `Signature` syntax and implicit sign offs use the `ImplicitMeta` syntax.

## Signature policies

```yaml
 - &Org1
        # DefaultOrg defines the organization which is used in the sampleconfig
        # of the fabric.git development environment
        Name: Org1MSP

        # ID to load the MSP definition as
        ID: Org1MSP

        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp

        # Policies defines the set of policies at this level of the config tree
        # For organization policies, their canonical path is usually
        #   /Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.peer', 'Org1MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org1MSP.admin')"
            Endorsement:
                Type: Signature
                Rule: "OR('Org1MSP.peer')" # peer that is a member of Org1MSP is required to sign.
```



## ImplicitMeta policies

`ImplicitMeta` policies are only valid in the context of channel configuration which is based on a tiered hierarchy of policies in a configuration tree. 

**ImplicitMeta policies aggregate the result of policies deeper in the configuration tree that are ultimately defined by Signature policies.** 

They are `Implicit` because they are constructed implicitly based on the current organizations in the channel configuration, and they are `Meta` because their evaluation is not against specific MSP principals, but rather against other sub-policies below them in the configuration tree.

例：**these ImplicitMeta policies are evaluated based on their underlying Signature sub-policies** which we saw in the snippet above.

```yaml
################################################################################
#
#   SECTION: Orderer
#
#   - This section defines the values to encode into a config transaction or
#   genesis block for orderer related parameters
#
################################################################################
Orderer: &OrdererDefaults

    # Orderer Type: The orderer implementation to start
    OrdererType: etcdraft

    # Organizations is the list of orgs which are defined as participants on
    # the orderer side of the network
    Organizations:

    # Policies defines the set of policies at this level of the config tree
    # For Orderer policies, their canonical path is
    #   /Channel/Orderer/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        # BlockValidation specifies what signatures must be included in the block
        # from the orderer for the peer to validate it.
        BlockValidation:
            Type: ImplicitMeta
            Rule: "ANY Writers"
```

The following diagram illustrates the tiered policy structure for an application channel and shows how the `ImplicitMeta` channel configuration admins policy, named `/Channel/Admins`, is resolved when the sub-policies named `Admins` below it in the configuration hierarchy are satisfied where each check mark represents that the conditions of the sub-policy were satisfied.

![](./images/implicitmeta-policy.png)

As you can see in the diagram above, `ImplicitMeta` policies, Type = 3, use a different syntax, `"<ANY|ALL|MAJORITY> <SubPolicyName>"`, for example:

```yaml
MAJORITY sub policy: Admins
```

The diagram shows a sub-policy `Admins`, which refers to all the `Admins` policy **below** it in the configuration tree. You can create your own sub-policies and name them whatever you want and then define them in each of your organizations.

As mentioned above, a key benefit of an `ImplicitMeta` policy such as `MAJORITY Admins` is that **when you add a new admin organization to the channel, you do not have to update the channel policy.** Therefore `ImplicitMeta` policies are considered to be **more flexible as the consortium members change**. 

Recall that **`ImplicitMeta` policies ultimately resolve the `Signature` sub-policies underneath them** in the configuration tree as the diagram shows.

You can also define an **application level implicit policy to operate across organizations**, in a channel for example, and either require that ANY of them are satisfied, that ALL are satisfied, or that a MAJORITY are satisfied. This format lends itself to much better, more natural defaults, so that each organization can decide what it means for a valid endorsement.

Further granularity and control can be achieved if you include [`NodeOUs`](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/policies/msp.html#organizational-units) in your organization definition. 

- **Organization Units (OUs) are defined in the Fabric CA client configuration file** and can be associated with an identity when it is created. 
- In Fabric, **`NodeOUs` provide a way to classify identities in a digital certificate hierarchy**. 
  - For instance, an organization having specific `NodeOUs` enabled could require that a ‘peer’ sign for it to be a valid endorsement, 
  - whereas an organization without any might simply require that any member can sign.

## Fabric chaincode lifecycle

The new chaincode lifecycle process allows multiple organizations to vote on how a chaincode will be operated before it can be used on a channel. 

The new chaincode lifecycle process flow includes two steps where policies are specified: 

1. when chaincode is **approved** by organization members
2. when it is **committed** to the channel.

The `Application` section of the `configtx.yaml` file includes the default chaincode lifecycle endorsement policy. 

```yaml
################################################################################
#
#   SECTION: Application
#
#   - This section defines the values to encode into a config transaction or
#   genesis block for application related parameters
#
################################################################################
Application: &ApplicationDefaults

    # Organizations is the list of orgs which are defined as participants on
    # the application side of the network
    Organizations:

    # Policies defines the set of policies at this level of the config tree
    # For Application policies, their canonical path is
    #   /Channel/Application/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
        Endorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
```

- The `LifecycleEndorsement` policy **governs who needs to *approve a chaincode definition***.
- The `Endorsement` policy is **the *default endorsement policy for a chaincode***.

## Chaincode endorsement policies

The endorsement policy is specified for a **chaincode** when it is **approved and committed** to the channel using the Fabric chaincode lifecycle (that is, one endorsement policy covers all of the state associated with a chaincode). 

The endorsement policy can be specified either by reference to an endorsement policy defined in the channel configuration or by explicitly specifying a Signature policy.

### Implicit endorsement policy 

If an endorsement policy is not explicitly specified during the approval step, the default `Endorsement` policy `"MAJORITY Endorsement"` is used which means that a majority of the peers（我理解为anchor peer） belonging to the different channel members (organizations) need to **execute and validate** a transaction against the chaincode in order for the transaction to be considered valid. 

This default policy allows organizations that join the channel to become automatically added to the chaincode endorsement policy. 

### Signature endorsement policy

Signature policies allow you to include `principals` which are simply a way of matching an identity to a role. 

Principals are described as ‘MSP.ROLE’

- `MSP` represents the required MSP ID (the organization)
-  `ROLE` represents one of the four accepted roles: Member, Admin, Client, and Peer. 
  - A role is associated to an identity when a user enrolls with a CA. 
  - You can customize the list of roles available on your Fabric CA.

Some examples of valid principals are:

- ‘Org0.Admin’: an administrator of the Org0 MSP
- ‘Org1.Member’: a member of the Org1 MSP
- ‘Org1.Client’: a client of the Org1 MSP
- ‘Org1.Peer’: a peer of the Org1 MSP
- ‘OrdererOrg.Orderer’: an orderer in the OrdererOrg MSP

There are cases where it may be necessary for a particular state (a particular key-value pair, in other words) to have a different endorsement policy. 

This **state-based endorsement** allows the default chaincode-level endorsement policies to be overridden by a different policy for the specified keys.







