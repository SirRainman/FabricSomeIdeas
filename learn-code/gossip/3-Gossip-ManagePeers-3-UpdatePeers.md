# 3 更新节点信息

Gossip Service服务器中的discover模块会周期性的发送/接收gossip msg，从而完成对网络中存活和死亡的peer节点的管理。

## 1 入口