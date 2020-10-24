# 1 搭建Cello过程中的问题

1. cello是如何在一个容器中创建另外一个容器的？

   1. 使用**/var/run/docker.sock**与Docker守护进程通信，从而能够管理Docker的Docker容器。
   2. **/var/run/docker.sock**是Docker守护进程(Docker daemon)默认监听的Unix域套接字(Unix domain socket)，容器中的进程可以通过它与Docker守护进程进行通信。

2. 为什么在虚拟机上搭建cello成功，在物理机上搭建失败？
   1. 客户端报错：0.async.js:1 POST http://10.176.24.50:8081/v2/chaincodes net::ERR_CONNECTION_RESET

   2. 服务器端报错：

      1. ```
         2020-10-21 14:50:54,816 ERROR 237 [-/undefined/-/35ms POST /v2/chaincodes] nodejs.ECONNRESETError: read ECONNRESET
             at _errnoException (util.js:992:11)
             at TCP.onread (net.js:618:25)
         code: "ECONNRESET"
         errno: "ECONNRESET"
         syscall: "read"
         headerSent: true
         name: "ECONNRESETError"
         pid: 237
         hostname: cello-user-dashboard
         ```

   3. 可能的原因：

      1. 客户端与服务端成功建立了长连接连接静默一段时间（无 HTTP 请求）
      2. 服务端因为在一段时间内没有收到任何数据，主动关闭了 TCP 连接
      3. 客户端在收到 TCP 关闭的信息前，发送了一个新的 HTTP 请求
      4. 服务端收到请求后拒绝，客户端报错 ECONNRESET
      5. 总结一下就是：服务端先于客户端关闭了 TCP，而客户端此时还未同步状态，所以存在一个错误的暂态（客户端认为 TCP 连接依然在，但实际已经销毁了）

# 2 Cello的架构

<!--画一下架构图-->



## 2.1 面向operator

面向管理员：

![cello](http://haoimg.hifool.cn/img/Cell.jpg)

## 2.2 面向user

面向用户：

![image-20201021182713443](http://haoimg.hifool.cn/img/image-20201021182713443.png)

# 3 Cello 解析

## 3.1 Cello的优点

1. 把CA结合的非常好，每个组织都有一个CA

## 3.2 Cello的缺点

1. 不支持指定peer进行查询
   1. 不支持负载均衡查询
2. 不支持查询账本-block-transaction



# 4 我们的架构之于Cello的优势在哪里？