## 什么是 Receiver

*receiver* 是 logback 的一个组件，用于接收远程 appender 的日志事件，根据本地策略打印接收到的日志事件。结合使用基于套接字的 appender 与 receiver，可以构建复杂的拓扑图，通过网络分发应用程序的日志事件。

一个 receiver 继承 [`ch.qos.logback.classic.net.ReceiverBase`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/ReceiverBase.html) 类。由于 receiver 继承了这个类，所以它也是 logback 组件 [LifeCycle](https://logback.qos.ch/xref/ch/qos/logback/core/spi/LifeCycle.html) 的一部分，而且它也是一个 [ContextAware](https://logback.qos.ch/xref/ch/qos/logback/core/spi/ContextAware.html)。

过去为了支持日志事件通过网络传输，logback 提供了 `SocketAppender` 与 `SimpleSocketServer`。appender 充当一个客户端，初始化到服务器应用程序的网络连接，再通过网络传输日志事件。receiver 组件以及相应的 appender 提供了很大的灵活性。在 *logback.xml* 中配置一个 receiver 组件，跟配置其它的组件一样。这允许配置 receiver 组件利用 [Joran](https://logback.qos.ch/manual/onJoran.html) 所有的功能。而且，任何应用都可以通过简单的配置一个或多个 receiver 组件从远程 appender 接收日志事件。

连接开始可以发生在 appender 与 recevier 之间的任何一方。一个 receiver 可以充当服务器的角色，被动的监听远程 appender 客户端的连接。或者，receiver 充当客户端的角色，初始化与远程 appender 服务器的连接。不管 appender 与 receiver 的角色是什么，*日志事件总是从 appender 流向 receiver。*

允许 receiver 初始化到 appender 的连接的灵活性在特定的情况下特别的有用：

-   由于安全的原因，中心日志服务器位于网络防火墙的后面，不允许即将到来的连接。使用 receiver 组件充当客户端的角色，中心日志服务器 (防火墙内) 可以初始化对应用程序 (防火墙外) 的连接。
-   开发者工具与企业管理应用可以获取正在运行中的程序的日志事件流。通常，logback 通过要求接收方应用程序 (例如，IDE 中正在运行的开发者工具) 去充当服务器的角色，被动的监听来自远程 appender 的连接来支持这个 (例如在 logback Beagle 中)。这提高了管理的困难，特别是当工具运行在开发者的工作站中的时候，有可能是通过移动设备。但是，可以使用 logback receiver 组件充当客户端角色来实现这些工具。初始化一个到远程 appender 的连接，接收日志事件用于本地显示，过滤以及报警。

一个 logback 的配置可以包含任何数量的 receiver 组件，充当任意组合的服务器或者客户端的角色。唯一的限制是每个充当服务器角色的 receiver 必须监听在不同的端口，每个充当客户端角色的 receiver 将会准备的连接到一个远程的 appender 上。

## Receivers 充当服务器角色

一个 receiver 可以被配置为充当服务器的绝对，被动的监听远程 appender 即将到来的连接。这个功能类似于使用独立的 `SimpleSocketServer` 应用，特别是在使用 receiver 组件时，*任何* 应用仅仅只需要在 *logback.xml* 中配置 receiver 就可以使用 logback-classic 接收来自远程 appender 的日志事件。

![serverSocketReceiver.png](images/serverSocketReceiver.png)

logback 包含两个 receiver 组件用于充当服务器角色。[`ServerSocketReceiver`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/server/ServerSocketReceiver.html) 以及它支持 SSL 的子类型 [`SSLServerSocketReceiver`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/server/SSLServerSocketReceiver.html)。这两个 receiver 组件都被设计成用来接收来自于 `SocketAppender` (或者 `SSLSocketAppender`) 客户端的连接。

`ServerSocketReceiver` 组件提供以下配置属性：

| 属性名      | 类型               | 描述                                                         |
| ----------- | ------------------ | ------------------------------------------------------------ |
| **address** | `String`           | receiver 监听的本地网络接口。如果没有指定这个属性，那么将监听所有的网络接口。 |
| **port**    | `int`              | receiver 监听的 TCP 端口。如果这个属性没有被指定，将会使用一个默认的值。(译者注：默认的端口为 4560) |
| **ssl**     | `SSLConfiguration` | 仅仅支持 `SSLServerSocketReceiver`。这个属性提供了 receiver 使用的 SSL 配置。详情请见 [第十五章](https://logback.qos.ch/manual/usingSSL.html) |

### 使用 ServerSocketReceiver

下面的配置使用 `ServerSocketReceiver` 组件，它使用最小的本地 appender 以及 logger 配置。接收来自远程 appender 的日志事件，将会匹配到 root logger，并输出到本地的 console appender。

>   Example: receiver1.xml

```xml
<configuration debug="true">

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>

  <receiver class="ch.qos.logback.classic.net.server.ServerSocketReceiver">
    <port>${port}</port>
  </receiver>

</configuration>
```

receiver 组件的 *class* 属性表明了我们想要使用的 receiver 子类型。在这个例子中，我们使用 `ServerSocketReceiver`。

我们示例中的服务器应用在功能与设计上与 `SimpleSocketServer` 非常的类似。它仅仅接收 logback 配置文件的路径作为命令行的参数，并运行给定的配置。虽然我们的示例程序很简单，但是请记住，你可以在*任何*应用中配置 logback 的 `ServerSocketReceiver` (或者 `SSLServerSocketReceiver`) 组件。

我们可以在 *logback-examples* 文件夹中通过以下方式运行示例中服务器应用：

```shell
java -Dport=6000 chapters.receivers.socket.ReceiverExample \ 
      src/main/java/chapters/receivers/socket/receiver1.xml
```

我们可以使用配置了 `SocketAppender` 的客户端应用来连接运行中的 receiver。我们示例当中的客户端应用仅仅加载了 logback 配置，就可以连接我们示例中 receiver 的 socket appender。然后，它会等待用户的数据，以消息的形式转发给 receiver。我们可以通过如下的方式来运行客户端应用示例：

```shell
java -Dhost=localhost -Dport=6000 \
      chapters.receivers.socket.AppenderExample \
      src/main/java/chapters/receivers/socket/appender1.xml
```

### 使用 SSLServerSocketReceiver

下面的配置重复使用了最小的 appender 以及 logger 配置，但是使用了支持 SSL 的 receiver 组件，用于充当服务器角色。

> Example: receiver2.xml

```xml
<configuration debug="true">

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>

  <receiver class="ch.qos.logback.classic.net.server.SSLServerSocketReceiver">
    <port>${port}</port>
    <ssl>
      <keyStore>
        <location>${keystore}</location>
        <password>${password}</password>
      </keyStore>
    </ssl>
  </receiver>

</configuration>
```

这个配置与之前使用 `ServerSocketReceiver` 的示例本质的区别是通过 *class* 属性指定了 `SSLServerSocketReceiver` 以及内嵌的 `ssl` 属性，使用变量替换来指定 key store 中包含 receiver 的私钥以及证书的位置以及密码。关于 logback 组件配置 SSL 属性的详细信息请查看[第十五章](https://logback.qos.ch/manual/usingSSL.html)。

我们可以使用相同的示例服务器配置来运行这个配置，仅仅添加几个额外的配置属性：

```shell
java -Dport=6001 \
      -Dkeystore=file:src/main/java/chapters/appenders/socket/ssl/keystore.jks \
      -Dpassword=changeit \
      chapters.receivers.socket.ReceiverExample \
      src/main/java/chapters/receivers/socket/receiver2.xml
```

*keystore* 属性被用来在命令行指定文件 URL 来标识 key store 的位置。你也可以使用[第十五章](https://logback.qos.ch/manual/usingSSL.html)所说的类路径 URL。

我们可以使用配置了 `SSLSocketAppender` 的客户端应用来连接运行中的 receiver。我们使用在之前示例中使用过的简单示例客户端应用，通过一个开启了 SSL appender 的配置文件，以如下方式运行：

```shell
java -Dhost=localhost -Dport=6001 \
      -Dtruststore=file:src/main/java/chapters/appenders/socket/ssl/truststore.jks \
      -Dpassword=changeit \
      chapters.receivers.socket.AppenderExample \
      src/main/java/chapters/receivers/socket/appender2.xml
```

注意，在我们的示例中，使用了自签名的 X.509 证书，这仅仅适用于测试。**在生产环境中的设置中，你应用获取一个合适的 X.509 证书，用于标识你的开启了 SSL 支持的 logback 组件。**查看[第十五章](https://logback.qos.ch/manual/usingSSL.html)获取更多的信息。

## Receivers 充当客户端角色

配置 receiver 充当客户端角色，初始化一个到远程 appender 的连接。远程 appender 必须是服务器类型，例如 `ServerSocketAppender`。

![socketReceiver](images/socketReceiver.png)



logback 包含两个 receiver 组件用于充当客户端角色：[`SocketReceiver`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SocketReceiver.html) 以及它的支持 SSL 子类型的 [`SSLSocketReceiver`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SSLSocketReceiver.html)。这两个组件都被设计成初始化一个连接到远程 appender，也就是 `ServerSocketAppender` (或者 `SSLServerSocketAppender`)。

`SocketReceiver` 子类型支持如下的配置属性：

| 属性名                | 类型               | 描述                                                         |
| --------------------- | ------------------ | ------------------------------------------------------------ |
| **remoteHost**        | `String`           | 远程服务器 socket appender 的 hostname 或者地址              |
| **port**              | `int`              | 远程服务器 socket appender 的端口号                          |
| **reconnectionDelay** | `int`              | 在某次连接失败之后，在尝试重连时的等待时间。默认值为 3000 (30 秒)。 |
| **ssl**               | `SSLConfiguration` | 仅仅支持 `SSLSocketReceiver`，这个属性提供了用于这个  receiver 的 SSL 配置。如[第十五章](https://logback.qos.ch/manual/usingSSL.html)描述的那样。 |

### 使用 SocketReceiver

用于 `SocketReceiver` 的配置跟之前使用 `ServerSocketReceiver` 的示例非常的相似。不同的地方在于客户端与服务端的角色反转了。`SocketReceiver` 类型的 receiver 为客户端，远程 appender 充当服务器的角色。

>   Example: receiver3.xml

```xml
<configuration debug="true">
    
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">    
    <encoder>
      <pattern>%date %-5level [%thread] %logger - %message%n</pattern>
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>  

  <receiver class="ch.qos.logback.classic.net.SocketReceiver">
    <remoteHost>${host}</remoteHost>
    <port>${port}</port>
    <reconnectionDelay>10000</reconnectionDelay>
  </receiver>

</configuration>
```

这个配置将会使 logback 连接通过 *host* 与 *port* 变量替换指定的主机与端口上的 `ServerSocketAppender`。将会通过 console appender 本地输出 (根据这里的配置文件) 从远程 appender 接收到的日志事件。

你可以在 *logback-examples/* 文件夹下，通过以下命令运行示例中的配置文件：

这个示例仅仅加载配置文件，然后仅仅等待来自远程 appender 的日志事件。如果你在远程 appender 没有运行的情况下运行这个示例，那么你将会周期性的看到*连接被拒绝*的日志消息输出。receiver 将会周期性的尝试重新连接远程 appender，直到连接成功或者 logger 上下文关闭。尝试的延迟间隔是可以通过 `reconnectionDelay` 属性来配置的，如示例配置中展示的一样。

```shell
java -Dhost=localhost -Dport=6000 \
      chapters.receivers.socket.ReceiverExample \
      src/main/java/chapters/receivers/socket/receiver3.xml
```

我们示例中的 receiver 连接之前使用过的同一个远程 apennder。这个示例加载一个包含 `ServerSocketAppender` 的配置，然后等待用户的输入，输入的消息将会被传递给已经连接上的 receiver。我们可以通过如下方式运行示例 appender 应用：

```shell
java -Dport=6000 \
      chapters.receivers.socket.AppenderExample \
      src/main/java/chapters/receivers/socket/appender3.xml
```

如果在 receiver 没有连接上的情况下输入消息，那么消息将会被丢弃。

### 使用 SocketSSLReceiver

`SSLSocketReceiver` 需要的配置跟使用 `SocketReceiver` 非常的类似。本质的区别在于通过 class 指定的 receiver，以及通过内嵌的 `ssl` 属性去指定 SSL 配置属性。下面是一个基础的示例：

>   Example: receiver4.xml

```xml
<configuration debug="true">

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">    
    <encoder>
      <pattern>%date %-5level [%thread] %logger - %message%n</pattern>
    </encoder>         
  </appender>

  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>  
 
  <receiver class="ch.qos.logback.classic.net.SSLSocketReceiver">
    <remoteHost>${host}</remoteHost>
    <port>${port}</port>
    <reconnectionDelay>10000</reconnectionDelay>
    <ssl>
      <trustStore>
        <location>${truststore}</location>
        <password>${password}</password>
      </trustStore>
    </ssl>
  </receiver>

</configuration>
```

除了在上一个例子中展示的配置属性之外，*class* 属性现在指定了 `SSLSocketReceiver`。配置文件中包含了指定 trust strore 的位置与密码的 SSL 配置。用于验证远程 appender 是可以受信任的。查看[第十五章](https://logback.qos.ch/manual/usingSSL.html)获取更多关于配置 SSL 属性的信息。

通过如下命令来运行示例配置：

```shell
java -Dhost=localhost -Dport=6001 \
      -Dtruststore=file:src/main/java/chapters/appenders/socket/ssl/truststore.jks \
      -Dpassword=changeit \
      chapters.receivers.socket.ReceiverExample \
      src/main/java/chapters/receivers/socket/receiver4.xml
```

一旦启动，receiver 尝试去连接指定的远程 appender。如果 appender 没有运行，那么你将会在日志输出中周期性的看到 "连接被拒" 的信息。在延迟了通过 `reconnectionDelay ` 指定的周期时间后，receiver 将会周期性的重试到远程 appender 的连接。

我们示例中的 receiver 会连接之前使用过的远程 appender。这个示例加载包含了 `SSLServerSocketAppender` 的配置，然后等待用户的输入，输入的信息将会被传递给连接上的 receiver。通过如下方式运行示例 appender 应用：

```shell
java -Dport=6001 \
      -Dkeystore=file:src/main/java/chapters/appenders/socket/ssl/keystore.jks \
      -Dpassword=changeit \
      chapters.receivers.socket.AppenderExample \
      src/main/java/chapters/receivers/socket/appender4.xml
```

如果在 receiver 没有连接上的时候输入信息，那么信息将会被丢弃。

需要再次注意的是，我们的示例使用的是仅适用于测试的自签名 X.509 证书。**在生产环境中，你应该获取适当的 X.509 证书来标识你的支持 SSL 的 logback 组件。**更多细节请查看[第十五章](https://logback.qos.ch/manual/usingSSL.html)。