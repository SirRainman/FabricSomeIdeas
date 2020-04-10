# World State：存储结果

Firstly, there’s a **world state** – **a database** that holds **current values** of a set of ledger states. 

The world state makes it easy for a program to **directly access the current value of a state** **rather than** having to calculate it by **traversing the entire transaction log**. 

Ledger states are, by default, expressed as **key-value** pairs. The world state can **change frequently**, as states can be created, updated and deleted.

# Blockchain ledger：存储过程 

Secondly, there’s a **blockchain** – **a transaction log** that records all the changes that have resulted in the current the world state. 

Transactions are collected inside blocks that are appended to the blockchain – enabling you to understand **the history of changes that have resulted in the current world state**. 

![](./images/ledger.png)

Block 分为三部分：

- header
- data
- metadata

## Block Header

![](./images/blockheader.png)

block header分为三部分：

- block number: block的序号，表明是第几个block，从0开始
- previous hash: 前一个block的hash值
- current hass: 当前块的hash值

## Block Data

里面存储的是，一系列的交易记录

### Transactions

![](./images/transaction.png)

一笔交易也有几部分构成：

- **Header**

  captures some essential **metadata about the transaction** – for example, the name of the relevant chaincode, and its version.

- **Signature**

  contains **a cryptographic signature**, created by the client application. This field is **used to check that the transaction details have not been tampered with**, as it requires the application’s private key to generate it.

- **Proposal**

  包含了传递给智能合约的参数，这些参数由application提供，参数中包含了一些与现有的world state的相关数据，然后智能合约根据这些参数将创建proposed ledger update。

- **Response**

  captures **the before and after values of the world state**, as a **Read Write set** (RW-set). 
  It’s the **output of a smart contract**, and if the transaction is successfully validated, it will be **applied to the ledger to update the world state**.

- **Endorsements**

  a list of **signed transaction responses** from each required organization sufficient to satisfy the endorsement policy. 
  each endorsement effectively encodes its organization’s particular transaction response – meaning that there’s no need to include any transaction response that doesn’t match sufficient endorsements as it will be rejected as invalid, and not update the world state.

## Block Metadata

This section contains **the certificate and signature of the block creator** which is used to verify the block by network nodes. 

Subsequently, the **block committer adds a valid/invalid indicator for every transaction** into a bitmap that also resides in the block metadata, as well as **a hash of the cumulative state updates** up until and including that block, in order to detect a state fork. 

Unlike the block data and header fields, this section is **not an input to the block hash** computation.

