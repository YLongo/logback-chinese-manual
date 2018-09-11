在从以 socket 为基础的 appender 到远程 receiver 传递日志事件时，logback 支持使用安全套接字层。当使用支持 SSL 的 appender 以及响应的 receiver 时，通过安全通道来传递日志事件。

## SSL 与组件的角色

logback 的组件，例如 appender 以及 receiver 在网络连接初始化时可能承当服务器的角色或者客户端的角色。当充当服务器角色时，logback 组件被动的监听来自远程客户端组件的连接。相反地，充当客户端角色的 logback 组件会初始化一个连接到远程服务器组件。例如，一个充当客户端角色的 appender 连接充当服务端角色的 receiver。或者一个充当客户端角色的 receiver 连接充当服务端角色的 appender。

组件的角色通常由组件的类型决定。例如，`SSLServerSocketAppender` 是一个充当服务端角色的 appender 组件，但是 `SSLSocketAppender` 是一个充当客户端角色的 appender 组件。因此开发者或者应用管理人员可以配置 logback 组件来支持想要的网络连接初始化方向。

网络连接初始化方法在 SSL 上下文中非常重要，因为在 SSL 中，服务端组件连接客户端必须具有 X.509 证书来标识自身。客户端组件，在连接服务端时，使用服务端的证书来验证服务端是否可信。