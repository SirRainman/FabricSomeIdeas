

## 案例：Channel creation

参考：[官网文档](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.0/configtx.html)

下面这一部分没看全，建议看官网文档，最后一部分

当一个orderer接收到一个给不存在的channel进行更新的命令时，那么这个orderer就会假设该命令是一个创建通道的命令，那么就会执行一下的命令：

1. 首先，通过搜索确定consortium值，该orderer识别出应该在哪个consortium执行创建通道的命令。

   ```bash
   peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx >&log.txt
   
   # 其中，在channel.tx中对通道进行了一些配置
   ```

2. 在channel.tx中对application进行了一些配置

   ```bash
   TwoOrgsChannel:
   	Consortium: SampleConsortium
   	<<: *ChannelDefaults 
   	Application:
   		<<: *ApplicationDefaults
   		Organizations:
   			- *Org1
   			- *Org2
   		Capabilities:
   			<<: *ApplicationCapabilities
   ```

3. orderer要确定channel.tx中，是否有关于consortium的配置信息，以及新的channel中是否有application member
4. orderer会根据配置信息生成一些template configuration，生成一个application用户组，以及指定一些背书策略
5. The orderer then applies the `CONFIG_UPDATE` as an update to this template configuration. Because the `CONFIG_UPDATE` applies modifications to the `Application` group (its `version` is `1`), the config code validates these updates against the `ChannelCreationPolicy`. If the channel creation contains any other modifications, such as to an individual org’s anchor peers, the corresponding mod policy for the element will be invoked.
6. The new `CONFIG` transaction with the new channel config is wrapped and sent for ordering on the ordering system channel. After ordering, the channel is created.