在从以 socket 为基础的 appender 到远程 receiver 传递日志事件时，logback 支持使用安全套接字层。当使用支持 SSL 的 appender 以及响应的 receiver 时，通过安全通道来传递日志事件。

## SSL 与组件的角色

logback 的组件，例如 appender 以及 receiver 在网络连接初始化时可能承当服务器的角色或者客户端的角色。当充当服务器角色时，logback 组件被动的监听来自远程客户端组件的连接。相反地，充当客户端角色的 logback 组件会初始化一个连接到远程服务器组件。例如，一个充当客户端角色的 appender 连接充当服务端角色的 receiver。或者一个充当客户端角色的 receiver 连接充当服务端角色的 appender。

组件的角色通常由组件的类型决定。例如，`SSLServerSocketAppender` 是一个充当服务端角色的 appender 组件，但是 `SSLSocketAppender` 是一个充当客户端角色的 appender 组件。因此开发者或者应用管理人员可以配置 logback 组件来支持想要的网络连接初始化方向。

网络连接初始化方法在 SSL 上下文中非常重要，因为在 SSL 中，服务端组件连接客户端必须具有 X.509 证书来标识自身。客户端组件，在连接服务端时，使用服务端的证书来验证服务端是否可信。开发人员或者应用管理者必须清楚的知道 logback 组件的角色，才能正确配置服务器的 keystore (包含服务器 X.509 证书) 以及客户端的 truststore (包含用于验证服务器信任时的自签名根证书)。

当为 **相互认证** 配置 SSL，服务器组件与客户端组件必须拥有有效的 X.509 证书，它们的信任由它们各自对等的声明。相互认证配置在服务器组件中，因此开发人员或者应用管理者必须知道哪一个组件充当的是服务器的角色。

在本章，我们使用术语 *服务器组件* 或者简单的 *服务器* 来表示 logback 中充当服务器角色的 appender 或者 receiver。使用 *客户端组件* 或者建的 *客户端* 来表示充当客户端角色的组件。

## SSL 以及 X.509 证书

为了是用支持 SSL 的 logback 组件，你需要一个 X.509 的证书 (一个私钥，相应的证书，以及 CA 认证链) 去标识你的组件充当了一个 SSL 服务器。如果你想要使用双向认证，那么充当 SSL 客户端的组件还需要一个证书。

虽然你可以使用由商业认证机构 (CA) 颁发的证书，但是你也可以使用内部 CA 颁发的证书，甚至是自签名的证书。下面是必须满足的条件：

1.  服务端组件必须配置一个 key store，包含服务器私钥，相应的证书，以及 CA 认证链 (如果没有使用自签名证书)
2.  客户端组件必须配置一个 trust store，包含可信任的根 CA 证书，或者服务器自签名根证书

## 为 SSL 配置 logback 组件

Java 安全套接字拓展 (JSSE) 以及 Java 加密体系 (JCA) 用来实现 logback 的 SSL，支持诸多的配置选项。可插拔的提供框架允许替换或增强内置的 SSL 以及平台的加密功能。为了满足你的对安全的需要，支持 SSL 的 logback 组件提供完全指定 SSL 引擎以及密码提供者配置方面的能力。

### 使用 JSSE 系统属性的基本 SSL 配置

幸运的是，支持 SSL 的 logback 组件几乎所有的 SSL 属性的配置都有合理的默认值。在多数的情况下，只需要配置一些 JSSE 的系统属性。

本节剩余的部分将讲述在大多数的环境中都需要被指定的 JSSE 属性。查看[定制 JSSE](https://docs.oracle.com/javase/1.5.0/docs/guide/security/jsse/JSSERefGuide.html#InstallationAndCustomization)来获取更多关于设置 JSSE 系统属性的信息来定制 JSSE。

如果你使用任何 logback 中支持 SSL 的 appender 或者 receiver 组件充当服务端角色 (例如，`SSLServerSocketReceiver`，`SSLServerSocketAppender`，`SimpleSSLSocketServer`)。你需要提供 key store 包含的密钥以及证书的 location，type 以及 password 来配置 JSEE 系统属性。

#### 服务端 key store 配置的系统属性

| 属性名                           | 描述                                                    |
| -------------------------------- | ------------------------------------------------------- |
| `javax.net.ssl.keyStore`         | 指定包含服务端组件的密钥以及证书的文件的文件系统的路径  |
| `javax.net.ssl.keyStoreType`     | 指定 key store 类型。如果这个属性没有指定，将默认为 JKS |
| `javax.net.ssl.keyStorePassword` | 指定访问 key store 的密码                               |

查看下面 [Examples]([Examples](https://logback.qos.ch/manual/usingSSL.html#Examples)) 部分，在应用启动的时候通过利用 logback 支持 SSL 的服务端组件来设置这些系统属性。

如果你的服务端组件使用的证书是商业认证机构 (CA) 认证，**你可能不需要在你应用的客户端组件中提供任何 SSL 配置。**当在你的服务端组件使用商业签名证书，仅仅只需要在运行服务端组件的 JVM 上设置系统的 key store 属性。

如果你使用自签名的服务器证书或者你服务器证书是认证机构 (CA) 签名但是它们的根证书不再 Java 平台默认的 trust store 中 (例如，你的组织有自己的内部签名机构)。你需要配置 JSSE 系统属性，以提供包含你服务器证书的 trust store 或者认证机构 (CA) 颁发的受信任根证书签名的服务器证书的 location，type，以及 password。**这些属性需要在每个利用支持 SSL 的客户端组件中的应用中设置。**

#### 客户端 trust store 配置的系统属性

| 属性名                             | 描述                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `javax.net.ssl.trustStore`         | 指定包含服务端组件的证书或者受信任根证书的文件的文件系统的路径 |
| `javax.net.ssl.trustStoreType`     | 指定 trust store 类型。如果这个属性没有被指定，则默认为 JKS  |
| `javax.net.ssl.trustStorePassword` | 指定访问 trust store 的密码                                  |

查看下面 [Examples]([Examples](https://logback.qos.ch/manual/usingSSL.html#Examples)) 部分，在应用启动的时候通过利用 logback 支持 SSL 的客户端组件来设置这些系统属性。

### 高级 SSL 配置

在特定的情况下，使用 JSEE 系统属性的基本 SSL 配置是不恰当的。比如，你想要在 web 应用中使用 `SSLServerSocketReceiver` 组件，你可能想要使用不同的证书为你的远程日志客户端标识你的日志服务器，而不是 web 服务器为了 web 客户端使用证书来标识自身。

