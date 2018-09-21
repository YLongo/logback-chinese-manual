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

在特定的情况下，使用 JSEE 系统属性的基本 SSL 配置是不恰当的。比如，你想要在 web 应用中使用 `SSLServerSocketReceiver` 组件，你可能想要使用不同的证书为你的远程日志客户端标识你的日志服务器，而不是 web 服务器为了 web 客户端使用证书来标识自身。你可能想要在日志服务器上验证 SSL 客户端，确保只有真正授权的远程 logger 才可以连接。或者你的组织对于在组织所在的网络上使用 SSL 协议以及密码套件有严格的策略。为了满足这些需求，你需要使用 logback SSL 高级配置选项。

在配置 logback 组件支持 SSL 时，你需要在组件的配置中指定 `ssl` 属性来指定 SSL 的配置。

例如，如果你想要使用 `SSLServerSocketReceiver` 以及为日志服务器认证配置 key store 属性，你可以使用类似如下的配置：

```xml
<configuration>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>
  
  <root level="debug">
    <appender-ref ref="CONSOLE" />
  </root>

  <receiver class="ch.qos.logback.classic.net.server.SSLServerSocketReceiver">
    <ssl>
      <keyStore>
        <location>classpath:/logging-server-keystore.jks</location>
        <password>changeit</password>
      </keyStore>
    </ssl>
  </receiver> 

</configuration>
```

该配置将 key store 的位置指定为位于应用类路径下的  *logging-server-keystore.jks*。你也可以通过 `file:` URL 来指定 key store 的位置。

如果你想要在你的应用中使用 `SSLSocketAppender`，但是不想要改变默认 trust store 使用的 JSSE `javax.net.ssl.trustStore` 属性。你可以通过如下的方式进行配置：

```xml
<configuration>
  <appender name="SOCKET" class="ch.qos.logback.classic.net.SSLSocketAppender">
    <ssl>
      <trustStore>
        <location>classpath:/logging-server-truststore.jks</location>
        <password>changeit</password>
      </trustStore>
    </ssl>
  </appender>
  
  <root level="debug">
    <appender-ref ref="SOCKET" />
  </root>

</configuration>
```

该配置将 key store 的位置指定为位于应用类路径下的  *logging-server-keystore.jks*。你也可以通过 `file:` URL 来指定 key store 的位置。

#### SSL 配置属性

JSSE 公开 了大量的可配置选项。logback 的 SSL 支持几乎所有这些可用的选项在支持 SSL 组件的配置中指定。在使用 XML 配置时，在组件的配置中，SSL 属性被嵌套在 `<ssl>` 元素中引入。这个配置元素对应 [`SSLConfiguration`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/SSLConfiguration.html) 类。

当为你的组件配置 SSL 时，你仅仅只需要配置那些默认值不适合的 SSL 属性。过度指定 SSL 配置经常会导致一些难以诊断的问题。

下面的表格展示了顶层的 SSL 配置属性。许多顶层的属性还有额外的子属性，这些子属性将会顶层属性之后描述。

| 属性名                  | 类型                                                         | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **keyManagerFactory**   | [`KeyManagerFactoryFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/KeyManagerFactoryFactoryBean.html) | 指定用于创建 [`KeyManagerFactory`](http://docs.oracle.com/javase/1.5.0/docs/api/javax/net/ssl/KeyManagerFactory.html) 的配置。如果这个属性没有配置，那么将会使用 Java 平台默认的 factory。见 [Key Manager Factory 配置](https://logback.qos.ch/manual/usingSSL.html#KeyManagerFactoryFactoryBean) |
| **keyStore**            | [`KeyStoreFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/KeyStoreFactoryBean.html) | 指定用于创建 [`KeyStore`](http://docs.oracle.com/javase/1.5.0/docs/api/java/security/KeyStore.html) 的配置。通过这个属性创建的 KeyStore 必须包含唯一的 X.509 证书 (包含密钥，相应的证书以及 CA 认证链)。这个证书由本地 SSL 提供给远程的 SSL。<br />当配置一个 SSL 客户端时 (例如，`SSLSocketAppender`)，仅仅只有在远程配置需要验证客户端身份时才需要该属性。<br />当配置一个 SSL 服务端时 (例如，`SimpleSSLSocketServer`)，该属性指定的 key store 包含了服务端证书。如果没有配置这个属性，JSSE 的 `javax.net.ssl.keyStore` 系统属性必须配置用于提供服务端 key store 的位置。查看 [定制 JSEE](https://docs.oracle.com/javase/1.5.0/docs/guide/security/jsse/JSSERefGuide.html#InstallationAndCustomization) 来获取更多关于 JSEE 系统属性的信息。<br />更多的信息查看 [Key Store 配置](https://logback.qos.ch/manual/usingSSL.html#KeyStoreFactoryBean)。 |
| **parameters**          | [`SSLParametersConfiguration`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/SSLParametersConfiguration.html) | 在 SSL 会话中，指定各种参数。见下面的 [SSL 参数配置](https://logback.qos.ch/manual/usingSSL.html#SSLParametersConfiguration)。 |
| **protocol**            | `String`                                                     | 指定创建 [`SSLContext`](http://docs.oracle.com/javase/1.5.0/docs/api/javax/net/ssl/SSLContext.html) 的 SSL 协议。见[命名规范](https://docs.oracle.com/javase/1.5.0/docs/guide/security/jsse/JSSERefGuide.html#AppA)。如果没有配置该参数，那么将使用 Java 平台默认的协议。 |
| **provider**            | `String`                                                     | 指定创建 [`SSLContext`](http://docs.oracle.com/javase/1.5.0/docs/api/javax/net/ssl/SSLContext.html) 的 JSSE 提供者。如果没有配置该参数，那么将使用 Java 平台默认的 JSSE 提供者。 |
| **secureRandom**        | [`SecureRandomFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/SecureRandomFactoryBean.html) | 指定创建 [`SecureRandom`](http://docs.oracle.com/javase/1.5.0/docs/api/java/security/SecureRandom.html) (安全的随机数生成器) 的配置。如果没有配置该参数，那么将使用 Java 平台默认的生成器。见下面的 [安全随机数生成配置](https://logback.qos.ch/manual/usingSSL.html#SecureRandomFactoryBean)。 |
| **trustManagerFactory** | [`TrustManagerFactoryFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/TrustManagerFactoryFactoryBean.html) | 指定创建 [`TrustManagerFactory`](http://docs.oracle.com/javase/1.5.0/docs/api/javax/net/ssl/TrustManagerFactory.html) 的配置。如果没有指定该参数，那么将使用 Java 平台默认的 factory。见下面的 [受信任的管理工厂](https://logback.qos.ch/manual/usingSSL.html#TrustManagerFactoryFactoryBean)。 |
| **trustStore**          | [`KeyStoreFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/KeyStoreFactoryBean.html) | 指定创建 [`KeyStore`](http://docs.oracle.com/javase/1.5.0/docs/api/java/security/KeyStore.html) 的配置，用于验证远程 SSL 的身份。通过这个属性创建的 KeyStore 应该包含一个或多个*信任锚 (trust anchors)* — keystore 中被标记为受信任的自签名证书。通常，trust store 包含自签名的 CA 证书。<br />通过该属性指定的 trust store 会覆盖任何通过 JSSE 的 `javax.net.ssl.trustStore` 以及平台默认的 trust store。 |

#### Key Store 配置

[`KeyStoreFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/KeyStoreFactoryBean.html) 指定创建包含 X.509 证书的 [`KeyStore`](http://docs.oracle.com/javase/1.5.0/docs/api/java/security/KeyStore.html) 所需要的配置。这个 factory bean 的属性能够用于 [SSL 配置](https://github.com/Volong/logback-chinese-manual/blob/master/15%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%EF%BC%9A%E4%BD%BF%E7%94%A8%20SSL.md#%E9%AB%98%E7%BA%A7-ssl-%E9%85%8D%E7%BD%AE) 中的 [keyStore](https://logback.qos.ch/manual/usingSSL.html#ssl.keyStore) 以及 [trustStore](https://logback.qos.ch/manual/usingSSL.html#ssl.trustStore) 属性。

| 属性名       | 类型     | 描述                                                         |
| ------------ | -------- | ------------------------------------------------------------ |
| **location** | `String` | 指定 key store 的位置 URL。使用 `file:` 指定 keystore 在文件系统中的位置。使用 `classpath:` 指定 keystore 在类路径下的位置。如果 URL 没有指定具体的策略，那么将使用 `classpath:`。 |
| **password** | `String` | 指定访问 keystore 的密码                                     |
| **provider** | `String` | 指定用于创建 `KeyStore` 的 JCA 提供者的名字。如果没有指定该属性，那么将使用 Java 平台默认的 keystore 提供者。 |
| **type**     | `String` | 指定 `KeyStore` 的类型。见 [命名规范](http://docs.oracle.com/javase/1.5.0/docs/guide/security/CryptoSpec.html#AppA)。如果没有指定该属性，那么将使用 Java 平台默认的 keystore 类型。 |

#### Key Manager Factory 配置

[`KeyManagerFactoryFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/KeyManagerFactoryFactoryBean.html) 指定创建 [`KeyManagerFactory`](http://docs.oracle.com/javase/1.5.0/docs/api/javax/net/ssl/KeyManagerFactory.html) 需要的配置。通常，没有必要详细的配置 key manager factory，因为平台默认的 factory 就可以满足大部分的需求。

| 属性名        | 类型     | 描述                                                         |
| ------------- | -------- | ------------------------------------------------------------ |
| **algorithm** | `String` | 指定 `KeyManagerFactory` 算法的名字。见 [命名规范](http://docs.oracle.com/javase/1.5.0/docs/guide/security/CryptoSpec.html#AppA)。如果没有指定该属性，那么将会使用 Java 平台默认的 key manager 算法。 |
| **provider**  | `String` | 指定生成 `SecureRandom` 生成器的 JCA 提供者的名字。如果没有指定该属性，那么将会使用 Java 平台默认的 JSSE 提供者。 |

#### Secure Random Generator 配置

[`SecureRandomFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/SecureRandomFactoryBean.html) 指定创建 [`SecureRandom`](http://docs.oracle.com/javase/1.5.0/docs/api/java/security/SecureRandom.html) 生成器所需要的配置。通常，没有必要详细的配置 secure random 生成器，因为平台默认的生成器就可以满足大部分的需求。

| 属性名        | 类型     | 描述                                                         |
| ------------- | -------- | ------------------------------------------------------------ |
| **algorithm** | `String` | 指定 `SecureRandom` 算法的名字。见 [命名规范](http://docs.oracle.com/javase/1.5.0/docs/guide/security/CryptoSpec.html#AppA)。如果没有指定该属性，那么将会使用 Java 平台默认的随机数生成器 (random number generation) 算法。 |
| **provider**  | `String` | 指定用于创建 `SecureRandom` 生成器的 JCA 提供者。如果没有指定该属性，那么将会使用 Java 平台默认的 JSSE 提供者。 |

#### SSL 参数配置

[`SSLParametersConfiguration`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/SSLParametersConfiguration.html) 允许定制受允许的 SSL 协议，密码套件，以及客户端认证选项。

| Property Name            | Type      | Description                                                  |
| ------------------------ | --------- | ------------------------------------------------------------ |
| **excludedCipherSuites** | `String`  | 指定以逗号分隔的 SSL 密码套件的名字或者模式列表，用来在会话期间进行禁用。这个属性通过 SSL 引擎过滤密码套件。任何被该属性匹配到的密码套件都会被禁用。<br />以逗号分割的各个字段可以是字符串或者正则表达式。<br />查看密码套件的[命名规则](https://docs.oracle.com/javase/1.5.0/docs/guide/security/jsse/JSSERefGuide.html#AppA) |
| **includedCipherSuites** | `String`  | 与上一个属性除了作用相反，其它都是一样 (懒得翻译了)。        |
| **excludedProtocols**    | `String`  | 这个是表示需要排除的 SSL 协议。与上一个属性，除了作用不同，其它都一样。 |
| **includedProtocols**    | `String`  | 与上一个协议作用相反，其它一致。                             |
| **needClientAuth**       | `boolean` | 设置属性为 `true` 表示，服务端需要一个有效的客户端证书。当客户端组件为 `SSLSocketAppender` 时，该属性会被忽略。 |
| **wantClientAuth**       | `boolean` | 设置属性为 `true` 表示，服务端想要一个有效的客户端证书。当客户端组件为 `SSLSocketAppender` 时，该属性会被忽略。(与上一个属性的差异，需要自己动手去实践。因为我也不清楚😂) |

#### Trust Manager Factory 配置

[`TrustManagerFactoryFactoryBean`](https://logback.qos.ch/xref/ch/qos/logback/core/net/ssl/TrustManagerFactoryFactoryBean.html) 指定创建 [`TrustManagerFactory`](http://docs.oracle.com/javase/1.5.0/docs/api/javax/net/ssl/TrustManagerFactory.html) 所需要的配置。通常，没有必要明确的配置 trust manager factory，平台默认的 factory 就可以满足大部分的需要。

| 属性名        | 类型     | 描述                                                         |
| ------------- | -------- | ------------------------------------------------------------ |
| **algorithm** | `String` | 指定 `TrustManagerFactory` 算法的名字。见 [命名标准](https://docs.oracle.com/javase/1.5.0/docs/guide/security/jsse/JSSERefGuide.html#AppA)。如果没有指定该属性，那么将使用 Java  平台默认的 key manager 算法。 |
| **provider**  | `String` | 指定创建 `SecureRandom` 生成器的 JCA 提供者的名字。如果没有指定该属性，那么将使用 Java 平台默认的 JSSE 提供者。 |

## 示例

### 使用 JSSE 系统属性

JSEE 系统属性可以用来指定包含服务端 X.509 证书的 keystore 的位置以及密码。或者包含客户端组件验证服务端信任的自签名根 CA 证书的 truststore 的位置以及密码。

#### 指定服务端 keystore

在运行服务端组件时，需要指定包含服务端证书的 keystore 的位置以及密码。一种方法是使用 JSSE 系统属性。下面的例子通过命令行的方式启动 `SimpleSSLSocketServer`：

```shell
java -DkeyStore=/etc/logback-server-keystore.jks \
     -DkeyStorePassword=changeit -DkeyStoreType=JKS \
     ch.qos.logback.net.SimpleSSLSocketServer 6000 /etc/logback-server-config.xml
```

