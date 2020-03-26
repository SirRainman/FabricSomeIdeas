# 什么是MSP？

**MSP和CA之间的关系：**

- CA产生代表用身份的identity，MSP则包含的是一组permissioned identity。（按我的理解就是MSP决定了哪些identity是permissioned）。也就是说每个identity都一个role，他不能干role规定之外的事。
- MSP由CA创建，CA专门为组织创建证书和MSP。（这句话的源头：[设置排序节点：创建组织连接](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/orderer_deploy.html)）

MSP 能够通过列举member的identity的方式来识别：哪些root CA 和intermediate CA可以定义一个域内的可信成员member

* 原文：The MSP identifies which Root CAs and Intermediate CAs are accepted to define the members of a trust domain by listing the identities of their members。
* MSP也可以识别哪些CA有权利给member发放有效力的证书。

MSP列举了哪些member可以参与到区块链网络（Channel, org, etc.）中。

MSP可以给一个identity赋予一些权限，说白了MSP就是管理了一些具有某些权限的permissioned identity列表。比如每个人都有一个身份证，但是这并不代表你有参与人民代表大会的权利，而MSP维护了一个可参与人民代表大会的permissioned identity list。



# 为什么需要MSP？

前提：Fabric是一种只有被信任的节点才能参与到其中的区块链平台，因此参与者必须证明自己的身份。

* 通过生成公钥和私钥的方法解决了怎样证明自己身份的问题。

* MSP用来解决：在不公开私钥的前提下，使网络上的其他节点信任该节点的问题。MSP定义了网络的参与者应该信任的一些组织。另外MSP给一些member提供了一些权限。


举例：在商店的支付场景中：CA类似于银行的角色，他负责发行银行卡；MSP类似政府财政部门，负责管理该商店只能用哪些银行卡进行支付。

假如你要假如到fabric区块链中，MSP能够是你参与到permissioned 区块链中：
 1. 首先你要有一个由CA颁发的identity
 2. 成为某一个Org的一个member，你需要被链上的其他成员认可。MSP就是怎样将你的identity链接到该组织的membership中的一种技术。通过将member的公钥添加到org的MSP中，实现membership。（The MSP is how the identity is linked to the membership of an organization. Membership is achieved by adding the member’s public key (also known as certificate, signing cert, or signcert) to the organization’s MSP.）
 3. 将MSP要么添加到consortium中，要么添加到channel中。
 4. 确保MSP包含在了网络中的policy definition里。



# MSP domains

MSP在区块链网络中的两个domain（译为层次？）出现

* Local MSP: locally on an actor's node(能够管理一个节点的一些权限？)
* Channel MSP: in channel configuration（能够管理一个channel的配置信息？）

本地和通道 MSP 之间的关键区别：

- 不在于它们如何工作（它们都将身份转换为角色）
- 而在于它们的**作用域**。



每个节点的信任域（例如组织）由节点的本地 MSP（例如 ORG1 或 ORG2）定义。

通过将组织的 MSP 添加到通道配置中，可以表示该组织加入了通道。

下图为区块链管理员安装和实例化智能合约时所发生的情况，通道由 ORG1 和 ORG2 管理。

![](./images/MSP-domains.png)

1. 管理员 `B` 使用身份连接到peer节点上，该身份是由`RCA1`颁发的，存储在本地 MSP中。
2. 当 `B` 试图在节点上安装智能合约时，该peer节点检查其本地 MSP `ORG1-MSP`，以验证 `B` 的身份确实是 `ORG1` 的成员。验证成功后才会允许安装。
3. 随后，`B` 希望在通道上实例化智能合约。
4. 因为这是一个通道操作，所以通道上的所有组织都必须同意该操作。
5. 因此，节必须检查该通道的 MSP。
6. 节点成功提交此命令



**本地 MSP 只被定义在它们应用到的节点或用户的文件系统上**。

因此，从物理和逻辑上来说，每个节点或用户只有一个本地 MSP。

但是，由于通道 MSP 对通道中的所有节点都可用，因此在逻辑上他们只在通道配置中定义一次。

然而，**通道 MSP 也在通道中每个节点的文件系统上实例化，并通过共识来保持同步**。

因此，虽然每个节点的本地文件系统上都有各通道 MSP 的副本，但从逻辑上讲，通道 MSP 驻留在通道或网络上并由通道或网络进行维护。

## Local MSP

Local MSP 是为client，和node（peer 和 orderer）设计的：

- 节点本地 MSP 为该节点定义权限（例如，节点管理员是谁）。
- 用户的本地 MSP 允许用户端在其交易中作为通道的成员（例如在链码交易中）或作为系统中特定角色的所有者（例如在配置交易中的某组织管理员）对自己进行身份验证。

MSP定义了谁拥有该级别的管理或参与权（节点管理员不一定是通道管理员，反之亦然），因此一个node必须由一个local msp定义一些权限。

## Channel MSP

channel msp 定义了通道级别的管理权，和参与权，每个参与通道的组织都必须有一个 MSP：

* 通道上的 Peer 节点和排序节点将共享通道 MSP，因此能够正确地对通道参与者进行身份验证。
* 如果某个组织希望加入通道，则需要在通道配置中加入一个包含该组织成员信任链的 MSP。否则，由该组织身份发起的交易将被拒绝。

![](./images/ChannelMSPConfig.png)

*A channel config.json file includes two organization MSPs*

The system channel MSP 包含了参与到同一个ordering service的所有组织的MSP。这里所指的ordering service 是指一个orderer node集群所提供的服务。

## Existence of  Local MSP and Channel MSP

- Local MSP定义在client或node(peer / order)中的文件系统上，因此对于每一个节点来将，都只有一个Local MSP，来管理它的权限。
- Channel MSP也定义在各个节点的文件系统上，于local msp所不同的是，同一个Channel的各个节点所维护的Channel MSP是相同的一致的。

![Exitence](./images/ExistenceMSP.png)

In this figure:

- the network system channel is administered by ORG1, 
- but another application channel can be managed by ORG1 and ORG2. 
- The peer is a member of and managed by ORG2, whereas ORG1 manages the orderer of the figure. 
- ORG1 trusts identities from RCA1, whereas ORG2 trusts identities from RCA2. 
- while ORG1 administers the network, ORG2.MSP does exist in the network definition.



- **网络 MSP** ：一个网络的配置文件，通过定义参与组织的 MSP 定义了谁是这个网络中的成员，并且定义了这些成员中的哪些被授权来执行相关管理任务 （比如，创建一个通道）。
- **通道 MSP**：对于一个通道来说，保持其成员MSP的不同至关重要。通道为一群特定组织提供了一个彼此间私有的通信方式，这些组织又对这个通道进行管理。该通道 MSP 中所解释的通道策略定义了谁有能力参与该通道上的某些操作，比如，添加组织或者实例化链码。注意，管理通道的权限和管理网络配置通道（或任何其他通道）的能力之间没有必然联系。管理权仅存在于被管理的范围内。
- **Peer 节点 MSP**：这个本地 MSP 是在每个节点的文件系统上定义的，并且每个节点都有一个单独的 MSP 实例。从概念上讲，该MSP和通道 MSP 执行着完全一样的操作，但仅适用于其被定义的节点。在节点上安装一个链码，这个就是使用节点的本地 MSP 来判定谁被授权进行某个操作的例子。
- **排序节点 MSP**： 就和Peer 节点 MSP 一样，排序节点的本地 MSP 也是在节点的文件系统上定义的，并且只会应用于这个节点。同时，排序节点也是由某个单独的组织所有，因此具有一个单独的 MSP 用于罗列它所信任的操作者或者节点。

# Org - MSP

org使用一个MSP管理org中的所有的member

## Organizational Units(OUs) an MSP

一个Org可能由于业务原因分为多个OU，如ORG1.MANUFACTURING and ORG1.DISTRIBUTION。

当一个CA给一个节点颁发X.509证书时，证书里的OU字段表示了该identity属于哪一个业务，从而MSP决定了该identity具有什么样的权限，从而实现了权限管理。

## Node OU 

没怎么看懂，仔细看一下



# MSP Structure 

![](./images/StructureMSP.png)

![](./images/msp-files.png)

- **根 CA**：该文件夹包含了根CA自主签名的X.509证书的列表，其中的根CA是受MSP代表的组织所信任的。在这个 MSP 文件夹中至少要有一个根 CA X.509 证书。

  这是最重要的一个文件夹，因为它指出了所有可用于证明成员属于对应组织的其他证书的来源 CA。

- **中间 CA**：该文件夹包含了受这个组织信任的中间 CA 对应的 X.509 证书列表。其中每个证书的签发方必须是MSP中的某个根CA或中间CA，若是中间CA，则该中间 CA 的证书签发 CA 信任链必须最终能够连上一个受信任的根 CA。

  中间 CA 可能代表了组织中不同的一个分支（就像 `ORG1` 有 `ORG1-MANUFACTURING` 和 `ORG1-DISTRIBUTION`一样）， 也可能代表了这个组织自身（比如当一个商业 CA 被用来管理组织的身份时）。在后边这个情况中，中间 CA 可以被用来代表组织的分支。从[这里](https://hyperledger-fabric.readthedocs.io/zh_CN/release-1.4/msp.html)你或许能看到更多关于 MSP 配置最佳实践的信息。注意，对于一个工作网络来说，也有可能没有任何的中间 CA，这种情况下，这个文件夹就是空的。

  就像根 CA 文件夹一样，中间CA文件夹定义了可用于证明组织成员身份的证书的签发CA。

- **组织单元 （OU）**：组织单元被列在 `$FABRIC_CFG_PATH/msp/config.yaml` 文件中，包含了组织单元的一个列表，其中的成员被认为是由这个 MSP 所代表的组织的一部分。当你想要把一个组织的成员限定为持有一个其中包含某特定组织单元的身份（由MSP指定的某个CA签发）的成员时，它是很有用的。

  指定 OU 不是必须的。如果没有列出任何 OU，MSP中的所有身份（由根 CA 和中间 CA 文件夹指出）都会被认为是组织的成员。

- **管理员**：该文件夹包含了一个身份列表，其中的身份为该组织定义了哪些操作者担任管理员。对于标准的 MSP 类型来说，在这个列表中应该有一个或者多个 X509 证书。

  值得注意的是，仅凭借某操作者担任管理员这一点并不能说明它可以管理某些特定的资源。对于一个给定身份来说，它在管理系统方面的实际能力是由管理系统资源的相关策略决定的。比如，一个通道策略可能会指明 `ORG1-MANUFACTURING` 管理员有权利来向通道中添加新的组织，然而 `ORG1-DISTRIBUTION` 管理员却没有这个权利。

  虽然 X.509 证书具有 `ROLE` 属性（比如，明确规定一个操作者是一个 `admin`），但是该属性指的是操作者在其组织内所扮演的角色，而不是在区块链网络上。这一点与`OU` 属性的目的类似，若已经定义`OU` 属性，则其指的是操作者在组织中的位置。

  如果某个通道策略允许来自一个（或某些）组织的任何管理员执行某些通道功能的话，那么`ROLE` 属性就**能够**用来授予该通道级别上的管理权力。这样的话，一个组织层面的角色可以授予一个网络级别的角色。

- **撤销证书**：如果一个参与者的身份被撤销，那么该身份的识别信息（不是指身份本身）就会被储存在这个文件夹中。对基于 X.509 的身份来说，这些标识符就是主体密钥标识符 （Subject Key Identifier，SKI） 和权限访问标识符（Authority Access Identifier，AKI）的字符串对，并且无论何时使用 X.509 证书，都会检查这些标识符，以确保证书未被撤销。

  虽然这个列表在概念上跟 CA 的证书撤销列表 （CRL） 是一样的，但是它还和从组织中撤销成员有关。这样一来，本地或通道 MSP 的管理员通过广播被撤销证书的发行CA的最新CRL，就可以迅速将这个参与者或者节点从组织中撤销。罗列这一列表并不是必须的，只有在证书要撤销的时候才会用到。

- **节点身份**：这个文件夹包含了节点的身份，比如，与`KeyStore`内容结合使用的加密材料将允许节点在向同通道或同网络上其他参与者发送的信息中验证自己的身份。对基于 X.509 的身份来说， 该文件夹包含了一个 **X.509 证书**。Peer节点会把这一证书放置到交易提案的响应中，比如，来表明该节点已经为此交易背书，在接下来的验证阶段会根据交易的背书策略来验证这一背书。

  本地 MSP 中必须拥有”节点身份“文件夹，节点中有且仅有一个X.509 证书。而通道 MSP中不使用该文件夹 。

- **私钥的 `KeyStore`**：这个文件夹是为 Peer 节点或者排序节点（或者在客户端的本地 MSP 中） 的本地 MSP 定义的，其中包含了节点的**签名秘钥**。这个秘钥与**节点身份**文件夹里的节点身份能以密码方式匹配，可用来对数据进行签名，比如在背书阶段对一个交易提案的响应进行签名。

  本地 MSP 必须有这个文件夹，而且该文件夹必须包含且仅包含一个私钥。很显然，这个文件夹的访问权限必须限定在对这个节点有管理权限的用户的身份。

  因为通道 MSP 的目标只是提供身份验证功能，而不是签名的能力，所以**通道 MSP** 的配置中不包含这个文件夹。

- **TLS 根 CA**：该文件夹包含了根CA的自主签名X.509证书的列表，其中的根CA是受该组织信任来**进行TLS通信**的。一个 TLS 通信的例子就是，Peer 节点为接受到更新过的账本， 需要连接到一个排序节点，这时就会发生TLS通信。

  MSP TLS 信息和网络内部的节点（Peer 节点和排序节点）有关联，换句话说，它与使用该网络的应用程序和管理员无关。

  这个文件夹中必须有至少一个 TLS 根 CA X.509 证书。

- **TLS 中间 CA**：这个文件夹包含了**在通信时**这个 MSP 所代表的的组织所信任的中间 CA 证书列表。当商业 CA 被用于一个组织的 TLS 证书时，该文件夹特别有用。跟成员的中间 CA 类似，指定中间 TLS CA 是可选项。

  关于 TLS 的更多信息，点击[这里](https://hyperledger-fabric.readthedocs.io/zh_CN/release-1.4/enable_tls.html)。















