# 链码启动的过程？

通过lscc完成

## 1.x的缺点？

## 2.x的不同

不再使用lscc完成启动一个peer容器的过程，转而使用_lifecycle来完成

最终链码会支持external run的过程，抛弃docker的形式。

chaincode是server，提供service。

可扩展性强：

- 不再被动的和peer绑定在一起，容错性强。
- 不同的组织可以扩展同一份chaincode。
- 

---

peer 的几个角色

- anchor peer：一个组织里暴露给其他组织本组织内的成员的
- leading peer：一个组织和ordering service沟通
  一共有几个leading peer？一个组织一个？还是一个联盟一个？
- peer

