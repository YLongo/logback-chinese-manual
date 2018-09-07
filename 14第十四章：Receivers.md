## 什么是 Receiver

*receiver* 是 logback 的一个组件，用于接收远程 appender 的日志事件，根据本地策略打印接收到的日志事件。结合使用基于套接字的 appender 与 receiver，可以构建复杂的拓扑图，通过网络分发应用程序的日志事件。

一个 receiver 继承 [`ch.qos.logback.classic.net.ReceiverBase`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/ReceiverBase.html) 类。由于 receiver 继承了这个类，所以它也是 logback 组件 [LifeCycle](https://logback.qos.ch/xref/ch/qos/logback/core/spi/LifeCycle.html) 的一部分，而且它也是一个 [ContextAware](https://logback.qos.ch/xref/ch/qos/logback/core/spi/ContextAware.html)。

过去为了支持日志事件通过网络传输，logback 提供了 `SocketAppender` 与 `SimpleSocketServer`。appender 充当一个客户端，初始化到服务器应用程序的网络连接，再通过网络传输日志事件。receiver 组件以及相应的 appender 提供了很大的灵活性。在 *logback.xml* 中配置一个 receiver 组件，跟配置其它的组件一样。这允许配置 receiver 组件利用 [Joran](https://logback.qos.ch/manual/onJoran.html) 所有的功能。而且，任何应用都可以通过简单的配置一个或多个 receiver 组件从远程 appender 接收日志事件。

连接开始可以发生在 appender 与 recevier 之间的任何一方。一个 receiver 可以充当服务器的角色，被动的监听远程 appender 客户端的连接。或者，receiver 充当客户端的角色，初始化与远程 appender 服务器的连接。不管 appender 与 receiver 的角色是什么，*日志事件总是从 appender 流向 receiver。*

允许 receiver 初始化到 appender 的连接的灵活性在特定的情况下特别的有用：

-   由于安全的原因，中心日志服务器位于网络防火墙的后面，不允许即将到来的连接。使用 receiver 组件充当客户端的角色，中心日志服务器 (防火墙内) 可以初始化对应用程序 (防火墙外) 的连接。
-   