
## 什么是 logback

logback 继承自 log4j，它建立在有十年工业经验的日志系统之上。它比其它所有的日志系统更快并且更小，包含了许多独特并且有用的特性。

## 天才第一步

### 要求

logback-classic 模块需要在 classpath 添加 slf4j-api.jar、logback-core.jar 以及 logback-classic.jar。

*Example 1.1: Basic template for logging*

```java
package chapters.introduction;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HelloWorld1 {

    public static void main(String[] args) {
        Logger logger = LoggerFactory.getLogger("chapters.introduction.HelloWorld1");
        logger.debug("hello world");
    }
}
```

类 `HelloWorld1` 定义在包 `chapters.introduction` 中，它导入了两个类 `Logger`、`LoggerFactory`，这两个类定义在 SLF4J API 中，在 org.slf4j 包中。

在这个例子中，`main()` 包含了一个 DEBUG 级别的日志语句，输出信息为 "hello world"

运行 `HelloWord1` 将会在控制台看到一行日志。由于 logback 默认的配置策略：当没有默认的配置时，logback 将会在 root logger 中新增一个 `ConsoleAppender` 

```
11:58:56.662 [main] DEBUG chapters.introduction.HelloWorld1 - hello world
```

Logback 通过一个内部的状态系统来报告它本身的状态信息。发生在 logback 生命周期中的事件可以通过 `StatusManager` 来获取。

*Example: Printing Logger Status*

```java
package chapters.introduction;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.core.util.StatusPrinter;

public class HelloWorld2 {

    public static void main(String[] args) {
        Logger logger = LoggerFactory.getLogger("chapters.introduction.HelloWorld2");
        logger.debug("Hello world");
        
        // 打印内部的状态
        LoggerContext lc = (LoggerContext)LoggerFactory.getILoggerFactory();
        StatusPrinter.print(lc);
    }
}
```

运行结果如下：

```java
12:23:49.324 [main] DEBUG chapters.introduction.HelloWorld2 - Hello world
12:23:49,258 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.xml]
12:23:49,258 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.groovy]
12:23:49,258 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.xml]
12:23:49,262 |-INFO in ch.qos.logback.classic.BasicConfigurator@2e5d6d97 - Setting up default configuration.
```

Logback 没有找到 *logback-test.xml* 和 *logback.xml* 配置文件，所以通过默认的配置策略-添加一个基本的 `ConsoleAppender` 来进行配置。`Appender` 类被看作为输出的目的地。Appenders 包括 console，files，Syslog，TCP Sockets，JMS 等等其它的日志输出目的地。用户可以根据自己的情况轻松的创建自己的 Appender。

如果发生了错误，logback 会自动在控制台打印自己内部的状态信息。

前面给的例子相对简单。实际上在大型的应用中，日志记录不会有太大的区别。日志记录的一般形式不会有改变，只是配置方式会有不同。可能你想要根据自己的需求来配置 logback，接下来的章节中将会介绍如何进行配置。

在上面的例子中，我们通过 `StatusPrinter.print()` 打印了 logback 自身的内部状态。logback 的内部状态对查找 logback 相关的问题非常的有用。

通过如下的三个步骤可以启用 logback 来记录日志：

1. 配置 logback 环境。你可以通过简单或者复杂的方式来做，这个在后面会叙述到。
2. 如果你想在每个类中打印日志，那么你需要将当前类的全称或者当前类当作参数，调用  `org.slf4j.LoggerFactory.getLogger()` 方法。
3. 使用实例 logger 来调用不同的方法来打印日志。例：debug()，info()，warn()，error()。通过这些方法将会在配置好的 appender 中输出日志。

## 构建 logback

logback 使用 Maven 作为构建工具。

如果你安装了 Maven，你可以在 logback 的解压文件夹中运行 `mvn install` 来构建 logback 以及它所包含的模块。Maven 会自动下载 logback 需要的其它类库。

Logback 的压缩包包含了完整的源码，所以你可以根据自己的需要随意更改。而且只要你使用 LGPL 跟 EPL 协议，你甚至可以发布你更改后的版本。
