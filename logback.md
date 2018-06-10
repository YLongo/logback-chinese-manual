---
title: logack 中文手册
date: 2018-02-06 10:36:21
toc: true
typora-copy-images-to: \images
---

logback 中文手册

<!-- more -->

## 简介

Logback 继承自 log4j。

Logback 的架构非常的通用，适用不同的使用场景。Logback 被分成三个不同的模块：logback-core，logback-classic，logback-access。

logback-core 是其它两个模块的基础。logback-classic 模块可以看作是 log4j 的一个优化版本，它天然的支持 SLF4J，所以你可以随意的从其它日志框架（例如：log4j 或者 java.util.logging）切回到 logack。

logback-access 可以与 Servlet 容器进行整合，例如：Tomcat、Jetty。它提供了 http 访问日志的功能。

## The logback manual

手册包括了最新版本的 logback，总共有 150 多页，以及许多具体的例子，主要包含以下基本的和高级的特性：

- logback 的整体架构
- 讨论 logback 最好的实践以及反模式
- logback 的 xml 配置方式
- appender
- encoder
- layout
- filter
- 上下文诊断
- Joran - logback 的配置系统

logback 手册尽可能详细的描述了 logback API，包括它的特性以及设计原理。logback 手册适用于那些使用 java 但是是 logback 新手的人，也适合那些对 logback 有一定经验的人。在手册的帮助下，新手可以快速上手。

# 第一章：logback 介绍

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

# 第二章：架构

## logback 的架构

跟[简介](#%E7%AE%80%E4%BB%8B)类似

## Logger, Appender 和 Layouts

Logback 构建在三个主要的类上：Logger，Appender 和 Layouts。这三个不同类型的组件一起作用能够让开发者根据消息的类型以及日志的级别来打印日志。

`Logger` 类作为 logback-classic 模块的一部分。`Appender` 与 `Layouts` 接口作为 logback-core 的一部分。作为一个通用的模块，logback-core 没有 logger 的概念。

### Logger 上下文

任何日志 API 的优势在于它能够禁止某些日志的输出，但是又不会妨碍另一些日志的输出。通过假定一个日志空间，这个空间包含所有可能的日志语句，这些日志语句根据开发人员设定的标准来进行分类。在 logback-classic 中，分类是 logger 的一部分，每一个 logger 都依附在 `LoggerContext` 上，它负责产生 logger，并且通过一个树状的层级结构来进行管理。

一个 Logger 被当作为一个实体，它们的命名是大小写敏感的，并且遵循一下规则：

> 命名层次结构
>
> 如果一个 logger 的名字加上一个 `.` 作为另一个 logger 名字的前缀，那么该 logger 就是另一个 logger 的祖先。如果一个 logger 与另一个 logger 之间没有其它的 logger ，则该 logger 就是另一个 logger 的父级。

例如：名为 `com.foo` 的 logger 是名为 `com.foo.Bar` 的 logger 的父级。名为 `java` 的 logger 是名为 `java.util` 的父级，是名为 `java.util.Vector` 的祖先。

root logger 作为 logger 层次结构的最高层。它是一个特殊的 logger，因为它是每一个层次结构的一部分。每一个 logger 都可以通过它的名字去获取。例：

```java
Logger rootLogger = LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME)
```

所有其它的 logger 通过 `org.slf4j.LoggerFactory` 类的静态方法 `getLogger` 去获取，这个方法需要传入一个 logger 的名字。下面是 `Logger` 接口一些基本的方法：

```java
package org.slf4j; 
public interface Logger { 
  public void trace(String message);
  public void debug(String message);
  public void info(String message); 
  public void warn(String message); 
  public void error(String message); 
}
```

### 有效等级又称为等级继承

Logger 能够被分成不同的等级。不同的等级（TRACE, DEBUG, INFO, WARN, ERROR）定义在 `ch.qos.logback.classic.Level` 类中。在 logback 中，类 `Level` 使用 final 修饰的，所以它不能用来被继承。一种更灵活的方式是使用 `Marker` 对象。

如果一个给定的 logger 没有指定一个层级，那么它就会继承离它最近的一个祖先的层级。更正式的说法是：

> 对于一个给定的名为 *L* 的 logger，它的有效层级为从自身一直回溯到 root logger，直到找到第一个不为空的层级作为自己的层级。

为了确保所有的 logger 都有一个层级，root logger 会有一个默认层级 --- DEBUG

以下四个例子指定不同的层级，以及根据继承规则得到的最终有效层级

*Example 1*

| logger 的名字 | 指定的层级 | 有效层级  |
| :--------: | :---: | :---: |
|    root    | DEBUG | DEBUG |
|     X      | none  | DEBUG |
|    X.Y     | none  | DEBUG |
|   X.Y.Z    | none  | DEBUG |

在这个例子中，只有 root logger 被指定了层级，所以 logger **X**，**X.Y**，**X.Y.Z** 的有效层级都是 DEBUG。

*Example 2*

| logger 的名字 | 指定的层级 | 有效层级  |
| :--------: | :---: | :---: |
|    root    | ERROR | ERROR |
|     X      | INFO  | INFO  |
|    X.Y     | DEBUG | DEBUG |
|   X.Y.Z    | WARN  | WARN  |

在这个例子中，每个 logger 都分配了层级，所以有效层级就是指定的层级。

*Example 3*

| logger 的名字 | 指定的层级 | 有效层级  |
| :--------: | :---: | :---: |
|    root    | DEBUG | DEBUG |
|     X      | INFO  | INFO  |
|    X.Y     | none  | INFO  |
|   X.Y.Z    | ERROR | ERROR |

在这个例子中，logger **root**，**X**，**X.Y.Z** 都分别分配了层级。logger **X.Y** 继承它的父级 logger **X**。

*Example 4*

| logger 的名字 | 指定的层级 | 有效层级  |
| :--------: | :---: | :---: |
|    root    | DEBUG | DEBUG |
|     X      | INFO  | INFO  |
|    X.Y     | none  | INFO  |
|   X.Y.Z    | none  | INFO  |

在这个例子中，logger **root**，**X** 都分配了层级。logger **X.Y**，**X.Y.Z** 的层级继承它们最近的父级 **X**。



### 方法打印以及基本选择规则

根据定义，打印的方法决定的日志的级别。例如：**L** 是一个 logger 实例，`L.info("...")` 的日志级别就是 INFO。

如果一条的日志的打印级别大于 logger 的有效级别，该条日志才可以被打印出来。这条规则总结如下：

> **基本选择规则**
>
> 日志的打印级别为 *p*，Logger 实例的级别为 *q*，如果 *p* >= *q*，则该条日志可以打印出来。

这条规则是 logbakc 的核心。各级别的排序为：**TRACE** < **DEBUG** < **INFO** < **WARN** < **ERROR**。

在下面的表格中，第一列表示的是日志的打印级别，用 *p* 表示。第一行表示的是 logger 的有效级别，用 *q* 表示。行列交叉处的结果表示由**基本选择规则**得出的结果。

![basic selection rule](/images/basic selection rule..png)

例子：

```java
package chapters.architecture;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import ch.qos.logback.classic.Level;

public class SelectionRule {

    public static void main(String[] args) {
        
        // ch.qos.logback.classic.Logger 可以设置日志的级别
        // 获取一个名为 "com.foo" 的 logger 实例
        ch.qos.logback.classic.Logger logger = 
                (ch.qos.logback.classic.Logger)LoggerFactory.getLogger("com.foo");
        // 设置 logger 的级别为 INFO
        logger.setLevel(Level.INFO);
        
        // 这条日志可以打印，因为 WARN >= INFO
        logger.warn("警告信息");
        // 这条日志不会打印，因为 DEBUG < INFO
        logger.debug("调试信息");
        
        // "com.foo.bar" 会继承 "com.foo" 的有效级别
        Logger barLogger = LoggerFactory.getLogger("com.foo.bar");
        // 这条日志会打印，因为 INFO >= INFO
        barLogger.info("子级信息");
        // 这条日志不会打印，因为 DEBUG < INFO
        barLogger.debug("子级调试信息");
    }
}
```

### 获取 Logger

通过 `LoggerFactory.getLogger()` 可以获取到具体的 logger 实例，名字相同则返回的 logger 实例也相同。

```java
Logger x = LoggerFactory.getLogger("wombat");
Logger y = LoggerFactory.getLogger("wombat");
```

**x**，**y** 是同一个 logger 对象。

可以通过配置一个 logger，然后在其它地方获取，而不需要传递引用。父级 logger 总是优于子级 logger，并且父级 logger 会自动寻找并关联子级 logger，即使父级 logger 在子级 logger 之后实例化。

logback 环境的配置会在应用初始化的时候完成。最优的方式是通过读取配置文件。

在每个类里面通过指定全限定类名为 logger 的名字来实例化一个 logger 是最好也是最简单的方式。因为日志能够输出这个 logger 的名字，所以这个命名策略能够看出日志的来源是哪里。虽然这是命名 logger 常见的策略，但是 logback 不会严格限制 logger 的命名，你完全可以根据自己的喜好来，你开心就好。

但是，根据类的全限定名来对 logger 进行命名，是目前最好的方式，没有之一。

### Appender 与 Layout

有选择的启用或者禁用日志的输出只是 logger 的一部分功能。logback 允许日志在多个地方进行输出。站在 logback 的角度来说，输出目的地叫做 appender。appender 包括console、file、remote socket server、MySQL、PostgreSQL、Oracle 或者其它的数据库、JMS、remote UNIX Syslog daemons 中。

一个 logger 可以有多个 appender。

logger 通过 `addAppender` 方法来新增一个 appender。对于给定的 logger，每一个允许输出的日志都会被转发到该 logger 的所有 appender 中去。换句话说，appender 从 logger 的层级结构中去继承叠加性。例如：如果 root logger 添加了一个 console appender，所有允许输出的日志至少会在控制台打印出来。如果再给一个叫做 ***L*** 的 logger 添加了一个 file appender，那么 ***L*** 以及 ***L*** 的子级 logger 都可以在文件和控制台打印日志。可以通过设置 additivity = false 来改写默认的设置，这样 appender 将不再具有叠加性。

appender 的叠加性规则如下：

> appender 的叠加性
>
> logger *L* 的日志输出语句会遍历 *L* 和它的子级中所有的 appender。这就是所谓的 appender 叠加性（appender additivity）
>
> 如果 *L* 的子级 logger 为 *P*，且 *P* 设置了 additivity = false，那么 *L* 的日志会在 *L* 所有 的 appender 包括 *P* 本身的 appender 中输出，但是不会在 *P* 的子级 appender 中输出。
>
> logger 默认设置 additivity = true。

|   Logger Name   | Attached Appenders | Additivity Flag |     Output Targets     |                 Comment                  |
| :-------------: | :----------------: | :-------------: | :--------------------: | :--------------------------------------: |
|      root       |         A1         |       不适用       |           A1           | root logger 为 logger 层级中的最高层，additivity 对它不适用 |
|        x        |     A-x1, A-x2     |      true       |     A1, A-x1, A-x2     |           x 与 root 的 appender            |
|       x.y       |        none        |      true       |     A1, A-x1, A-x2     |           x 与 root 的 appender            |
|      x.y.z      |       A-xyz1       |      true       | A1, A-x1, A-x2, A-xyz1 |        x 与 x.y 与 root 的 appender         |
|    security     |       A-sec        |      false      |         A-sec          | 因为 additivity = false，所以只有 A-sec 这个 appender |
| security.access |        node        |      true       |         A-sec          | 因为它的父级 logger security 设置了 additivity = false，所以只有 A-sec 这一个 appender |

通常，用户既想自定义日志的输出地，也想自定义日志的输出格式。通过给 appender 添加一个 *layout* 可以做到。layout 的作用是将日志格式化，而 appender 的作用是将格式化后的日志输出到指定的目的地。**PatternLayout** 能够根据用户指定的格式来格式化日志，类似于 C 语言的 printf 函数。

例：PatternLayout 通过格式化串 "%-4relative [%thread] %-5level %logger{32} - %msg%n" 会将日志格式化成如下结果：

```java
176  [main] DEBUG manual.architecture.HelloWorld2 - Hello world.
```

第一个参数表示程序启动以来的耗时，单位为毫秒。第二个参数表示当前的线程号。第三个参数表示当前日志的级别。第四个参数是 logger 的名字。“-” 之后是具体的日志信息。

### 参数化日志

考虑到 logback-classic 实现了 SLF4J 的 Logger 接口，一些打印方法可以接收多个传参。这些打印方法的变体主要是为了提高性能以及减少对代码可读性的影响。

对于一些 Logger 如下输出日志：

```java
logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
```

会产生构建消息参数的成本，是因为需要将整数转为字符串，然后再将字符串拼接起来。但是我们是不需要关心 debug 信息是否被记录（强行曲解作者的意思）。

为了避免构建参数带来的损耗，可以在日志记录之前做一个判断，如下：

```java
if(logger.isDebugEnabled()) { 
  logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
}
```

在这种情况下，如果 **logger**没有开启 debug 模式，不会有构建参数带来的性能损耗。换句话说，如果 logger 在 debug 级别，将会有两次性能的损耗，一次是判断是否启用了 debug 模式，一次是打印 debug 日志。在实际应用当中，这种性能上的损耗是可以忽略不计的，因为它所花费的时间小于打印一条日志的时间的 1%。

#### 更好的选择

有一种更好的方式去格式化日志信息。假设 **entry** 是一个 Object 对象：

```java
Object entry = new SomeObject();
logger.debug("The entry is {}", entry);
```

只有在需要打印 debug 信息的时候，才会去格式化日志信息，将 '{}' 替换成 entry 的字符串形式。也就是说在这种情况下，如果禁止了日志的打印，也不会有构建参数上的性能消耗。

下面两行输出的结果是一样的，但是一旦禁止日志打印，第二个变量的性能至少比第一个变量好上 30 倍。

```java
logger.debug("The new entry is " + entry + ".");
logger.debug("The new entry is {}", entry);
```

使用两个参数的例子如下：

```java
logger.debug("The new entry is {}, It replaces {}.", entry, oldEntry);
```

如果需要使用三个或三个以上的参数，可以采用如下的形式：

```java
Object[] paramArray = {newVal, below, above};
logger.debug("Value {} was inserted between {} and {}.", paramArray);
```

### 底层实现初探

在介绍了基本的 logback 组件之后，我们准备介绍一下，当用户调用日志的打印方法时，logback 所执行的步骤。现在我们来分析一下当用户通过一个名为 *com.wombat* 的 logger 调用了 **info()** 方法时，logback 执行了哪些步骤。

**第一步：获取过滤器链**

如果存在，则 **TurboFilter** 过滤器会被调用，Turbo 过滤器会设置一个上下文的阀值，或者根据每一条相关的日志请求信息，例如：**Marker**, **Level**， **Logger**， 消息，**Throwable ** 来过滤某些事件。如果过滤器链的响应是 *FilterReply.DENY*，那么这条日志请求将会被丢弃。如果是 *FilterReply.NEUTRAL*，则会继续执行下一步，例如：第二步。如果响应是 *FilterRerply.ACCEPT*，则会直接跳到第三步。

**第二步：应用[基本选择规则](#方法打印以及基本选择规则)**

在这步，logback 会比较有效级别与日志请求的级别，如果日志请求被禁止，那么 logback 将会丢弃调这条日志请求，并不会再做进一步的处理，否则的话，则进行下一步的处理。

**第三步：创建一个 LoggingEvent 对象 **

如果日志请求通过了之前的过滤器，logback 将会创建一个 ch.qos.logback.classic.LoggingEvent 对象，这个对象包含了日志请求所有相关的参数，请求的 logger，日志请求的级别，日志信息，与日志一同传递的异常信息，当前时间，当前线程，以及当前类的各种信息和 MDC。MDC 将会在[后续章节](#MDC)进行讨论。

**第四步：调用 appender**

在创建了 LoggingEvent 对象之后，logback 会调用所有可用 appender 的 doAppend() 方法。这些 appender 继承自 logger 上下文。

所有的 appender 都继承了 AppenderBase 这个抽象类，并实现了 doAppend() 这个方法，该方法是线程安全的。AppenderBase 的 doAppend() 也会调用附加到 appender 上的自定义过滤器。自定义过滤器能动态的动态的添加到 appender 上，在[过滤器](#过滤器)章节会详细讨论。

**第五步：格式化输出**

被调用的 appender 负责格式化日志时间。但是，有些 appender 将格式化日志事件的任务委托给 layout。layout 格式化 LoggingEvent 实例并返回一个字符串。对于其它的一些 appender，例如 SocketAppender，并不会把日志事件转变为一个字符串，而是进行序列化，因为，它们就不需要一个 layout。

**第六步：发送 LoggingEvent**

当日志事件被完全格式化之后将会通过每个 appender 发送到具体的目的地。

下图是 logback 执行步骤的 UML 图：

![点击查看大图](/images/underTheHoodSequence2.gif)

### 性能

记录日志经常被提到的一个点是它的计算代价。这是一个合理的考虑，因为一个中等大小的应用都可以产生成千上万的日志。我们的大部分努力都花在了测量以及调整 logback 的性能。但是用户还是应该知道以下有关性能的问题。

1. **当日志记录被关闭时记录日志的性能**

通过设置 root logger 的日志级别为 Level.OFF 来完全关闭日志的打印。当日志完全关闭的时候，日志请求的成本为方法的调用以及整数的比较。在 3.2Ghz 奔腾D 的电脑上的耗时大约为 20 纳秒。

任何方法的调用都有参数构建这个隐含的成本在里面。例如下面这个例子：

```java
x.debug("Entry number: " + i + "is " + entry[i]);
```

把整数 i、entry[i] 转变为字符串，并且连接在一起，而不管这条日志是否会被打印。

构建参数的成本取决于参数的大小，为了避免不必要的性能损耗，可以使用 SLF4J's 的参数化构建：

```java
x.debug("Entry number: {} is {}", i, entry[i]);
```

这种形式不会有构建参数的成本在里面。与上一个例子做比较，这个的速度会更快。只有当日志信息传递给了附加的 appender 时才会被格式化，而且格式化日志信息的组件也是被优化过的。

2. **当日记记录被打开时是否记录日志的性能**

在 logback 中，不需要遍历 logger 的层次结构。logger 在创建的时候就知道自己的有效级别。如果父级 logger 的级别被更改，则会通知所有子级 logger 注意这个更改。因此，在基于有效级别的基础上，logger 能够准实时的做出决定是否接受或者拒绝日志请求，而不需要考虑它的祖先的级别。

3. **日记记录的实际情况（格式化输出到指定设备）**

这是指格式化日志输出以及发送指定的目的地所需要的成本。我们尽可能快的让 layout（格式化）以及 appender 执行。在本地机器上，将日志输出到文件大概耗费 9-12 微秒的时间。当把日志输出到数据库或者远程服务器上时会上升到几毫秒。

尽管 logback 功能丰富，但是它最重要的目标之一是处理速度，这是仅次于可靠性的要求。为了提高性能，一些 logback 的组件被重写了几次

# 第三章：logback 的配置

我们开始通过多种配置 logback，以及许多示例的配置脚本。logback 依赖的配置框架 - [Joran](#Joran) 将会在之后的章节介绍

## 配置 logback

在应用程序当中使用日志语句需要耗费大量的精力。根据调查，大约有百分之四的代码用于打印日志。即使在一个中型应用的代码当中也有成千上万条日志的打印语句。考虑到这种情况，我们需要使用工具来管理这些日志语句。

可以通过编程或者配置 XML 脚本或者 Groovy 格式的方式来配置 logback。对于已经使用 log4j 的用户可以通过这个[工具](https://logback.qos.ch/translator/)来把 log4j.properties 转换为 logback.xml。

以下是 logback 的初始化步骤：

1. logback 会在类路径下寻找名为 logback-test.xml 的文件。
2. 如果没有找到，logback 会继续寻找名为 logback.groovy 的文件。
3. 如果没有找到，logback 会继续寻找名为 logback.xml 的文件。
4. 如果没有找到，将会通过 JDK 提供的 [ServiceLoader](https://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html) 工具在类路径下寻找文件 *META-INFO/services/ch.qos.logback.classic.spi.Configurator*，该文件的内容为实现了 [`Configurator`](https://logback.qos.ch/xref/ch/qos/logback/classic/spi/Configurator.html) 接口的实现类的全限定类名。
5. 如果以上都没有成功，logback 会通过 [BasicConfigurator](https://logback.qos.ch/xref/ch/qos/logback/classic/BasicConfigurator.html) 为自己进行配置，并且日志将会全部在控制台打印出来。

最后一步的目的是为了保证在所有的配置文件都没有被找到的情况下，提供一个默认的（但是是非常基础的）配置。

如果你使用的是 maven，你可以在 *src/test/resources* 下新建 logback-test.xml。maven 会确保它不会被生成。所以你可以在测试环境中给配置文件命名为 *logback-test.xml*，在生产环境中命名为 *logback.xml*。

`FAST START-UP` Joran 解析给定的配置文件大概需要耗费 100 毫秒。为了减少启动的世间安，你可以使用 [ServiceLoader](https://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html) 来加载自定义的 `Configurator`，并使用 [BasicConfigurator](https://logback.qos.ch/xref/ch/qos/logback/classic/BasicConfigurator.html) 作为一个好的起点（个人的理解是通过继承这个类）。

### 自动配置 logback

最简单的方式配置 logback 是让它去加载默认的配置。例：

```java
package chapters.configuration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyApp1 {

    public static final Logger LOGGER = LoggerFactory.getLogger(MyApp1.class);
    
    public static void main(String[] args) {
        LOGGER.info("Entering application.");
        
        Foo foo = new Foo();
        foo.doIt();
        LOGGER.info("Exiting application.");
    }   
}
```

`Foo` 的代码如下：

```jav
package chapters.configuration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Foo {
    
    public static final Logger LOGGER = LoggerFactory.getLogger(Foo.class);

    public void doIt() {
        LOGGER.debug("Did it again!");
    }
}
```

假设配置文件 *logback-test.xml* 或者 *logback.xml* 不存在，logback 会调用 [BasicConfigurator](https://logback.qos.ch/xref/ch/qos/logback/classic/BasicConfigurator.html) 进行最小的配置。最小的配置包含一个附加到 root logger 上的 `ConsoleAppender`，格式化输出使用 `PatternLayoutEncoder` 对模版 *%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n* 进行格式化。root logger 默认的日志级别为 `DEBUG`。

所以，*MyApp1* 的输出信息如下：

```java
16:59:20.161 [main] INFO chapters.configuration.MyApp1 - Entering application.
16:59:20.164 [main] DEBUG chapters.configuration.Foo - Did it again!
16:59:20.164 [main] INFO chapters.configuration.MyApp1 - Exiting application.
```

`MyApp1`  通过调用 `org.slf4j.LoggerFactory` 与 `org.slf4j.Logger` 这两个类与 logback 相关联，并检索会用到的 logger。除了配置 logback 的代码，客户端的代码不需要依赖 logback，因为 SLF4J 允许在它的抽象层下使用任何日志框架，所以非常容易将大量代码从一个框架迁移到另一个框架。

### 使用 *logback-test.xml* 或 *logback.xml* 自动配置

下面的配置等同于通过 `BasicConfigurator` 进行配置。

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender"> 
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

> 你需要将上面的配置文件命名为 logback.xml 或 logback-test.xml

运行 *MyApp1*，你将会看到相同的结果（你要是不相信，你可以更改模版，看是否生效）。

#### 在警告或错误的情况下自动打印状态信息

如果在解析配置文件的过程当中发生了错误，logback 会在控制台打印出它的内部状态数据。如果用户明确的定义了状态监听器，为了避免重复，logback 将不会自动打印状态信息。

在没有警告或错误的情况下，如果你想查看 logback 内部的状态信息，可以通过 `StatusPrinter` 类来调用 `print()` 方法查看具体的信息。

> 在 *MyApp1* 的基础上添加两行代码，并命名为 *MyApp2*

```java
package chapters.configuration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.core.util.StatusPrinter;

public class MyApp2 {

    public static final Logger LOGGER = LoggerFactory.getLogger(MyApp2.class);
    
    public static void main(String[] args) {
        
        LoggerContext lc = (LoggerContext)LoggerFactory.getILoggerFactory();
        StatusPrinter.print(lc);
        
        LOGGER.info("Entering application.");
        
        Foo foo = new Foo();
        foo.doIt();
        LOGGER.info("Exiting application.");
    }
}
```

输出信息如下：

```java
17:56:47,130 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.xml]
17:56:47,130 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.groovy]
17:56:47,131 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Found resource [logback.xml] at [file:/D:/E/project/logback-examples/target/classes/logback.xml]
17:56:47,224 |-INFO in ch.qos.logback.classic.joran.action.ConfigurationAction - debug attribute not set
17:56:47,225 |-INFO in ch.qos.logback.core.joran.action.AppenderAction - About to instantiate appender of type [ch.qos.logback.core.ConsoleAppender]
17:56:47,238 |-INFO in ch.qos.logback.core.joran.action.AppenderAction - Naming appender as [STDOUT]
17:56:47,251 |-INFO in ch.qos.logback.core.joran.action.NestedComplexPropertyIA - Assuming default type [ch.qos.logback.classic.encoder.PatternLayoutEncoder] for [encoder] property
17:56:47,328 |-INFO in ch.qos.logback.classic.joran.action.RootLoggerAction - Setting level of ROOT logger to DEBUG
17:56:47,329 |-INFO in ch.qos.logback.core.joran.action.AppenderRefAction - Attaching appender named [STDOUT] to Logger[ROOT]
17:56:47,329 |-INFO in ch.qos.logback.classic.joran.action.ConfigurationAction - End of configuration.
17:56:47,332 |-INFO in ch.qos.logback.classic.joran.JoranConfigurator@7d4793a8 - Registering current configuration as safe fallback point

17:56:47.340 [main] INFO  chapters.configuration.MyApp2 - Entering application.
17:56:47.342 [main] DEBUG chapters.configuration.Foo - Did it again!
17:56:47.342 [main] INFO  chapters.configuration.MyApp2 - Exiting application.

```

在输出信息中，可以清楚的看到内部的状态信息，又称之为 `Status` 对象，可以很方便的获取 logback 的内部状态。

#### 状态数据

你可以通过构造一个配置文件来打印状态信息，而不需要通过编码的方式调用 `StatusPrinter` 去实现。只需要在 *configuration* 元素上添加 *debug* 属性。配置文件如下所示。

> 注意：debug 属性只跟状态信息有关，并不会影响 logback 的配置文件，也不会影响 logger 的日志级别。

*Example: sample1.xml*

```xml
<configuration debug="true">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

> 需要将 sample1.xml 改名为 logback.xml 或 logback-test.xml，不然 logbak 找不到配置文件。以后这种情况不再重复申明。

如果配置文件的配置有问题，logback 会检测到这个错误并且在控制台打印它的内部状态。但是，如果配置文件没有被找到，logback 不会打印它的内部状态信息，因为没有检测到错误。通过编码方式调用 `StatusPrinter.print()` 方法会在任何情况下都打印状态信息。

`强制输出状态信息`：在缺乏状态信息的情况下，要找一个有问题的配置文件很难，特别是在生产环境下。为了能够更好的定位到有问题的配置文件，可以通过系统属性 "[logback.statusListenerClass](#"logback.statusListenerClass" system property)" 来设置 `StatusListener` 强制输出状态信息。系统属性 "logback.statusListenerClass" 也可以用来在遇到错误的情况下进行输出。

设置 `debug="true"` 完全等同于配置一个 `OnConsoleStatusListener` 。具体示例如下：

*Example: onConsoleStatusListener.xml*

```xml
<configuration>

    <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />
    <!-- 剩下的配置跟之前的相同 -->
</configuration>
```

设置 `debug="true"` 与配置 `OnConsoleStatusListener` 的效果完全一样。

### 通过系统属性指定默认的配置文件

通过系统属性 `logback.configurationFile` 可以指定默认的配置文件的路径。它的值可以是 URL，类路径下的文件或者是应用外部的文件。

```java
java -Dlogback.configurationFile=/path/to/config.xml chapters.configuration.MyApp1
```

> 注意：文件类型只能是 ".xml" 或者 ".groovy"，其它的拓展文件将会被忽略。

因为 `logback.configureFile` 是一个系统属性，所以也可以在应用内进行设置。但是必须在 logger 实例创建前进行设置。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import ch.qos.logback.classic.util.ContextInitializer;

public class ServerMain {

    static {
        System.setProperty(ContextInitializer.CONFIG_FILE_PROPERTY, "configurationFile.xml");
    }
    
    private static final Logger LOGGER = LoggerFactory.getLogger(ServerMain.class);
    
    public static void main(String[] args) {
        LOGGER.info("xxxxxxxx");
    }
}
```

### 当配置文件更改时，自动加载

为了让 logback 能够在配置文件改变的时候自动去扫描，需要在 `<configuration>` 标签上添加 `scan=true` 属性。

*Example*

```xml
<configuration scan="true">
    ...
</configuration>
```


默认情况下，一分钟扫描一次配置文件，看是否有更改。通过 `<configuration>` 标签上的 `scanPeriod` 属性可以指定扫描周期。扫描周期的时间单位可以是毫秒、秒、分钟或者小时。

*Example*：

```xml
<configuration scan="true" scanPeriod="30 seconds"
   ...
</configuration>
```

> 注意：如果没有指定时间单位，则默认为毫秒。

当设置了 `scan="true"`，会新建一个 [ReconfigureOnChangeTask](#https://logback.qos.ch/xref/ch/qos/logback/classic/joran/ReconfigureOnChangeTask.html) 任务用于监视配置文件是否变化。`ReconfigureOnChangeTask` 也会自动监视外部文件的变化。

如果更改后的配置文件有语法错误，则会回退到之前的配置文件。

#### 在堆栈中打包数据

> 注意：在 1.1.4 版本中，打包数据是默认被禁用的。

如果启用了打包数据，logback 会在堆栈的每一行显示 jar 包的名字以及 jar 的版本号。打包数据可以很好的解决 jar 版本冲突的问题。但是，这个的代价比较高，特别是在频繁报错的情况下。

`Example`：

```txt
14:28:48.835 [btpool0-7] INFO  c.q.l.demo.prime.PrimeAction - 99 is not a valid value
java.lang.Exception: 99 is invalid
  at ch.qos.logback.demo.prime.PrimeAction.execute(PrimeAction.java:28) [classes/:na]
  at org.apache.struts.action.RequestProcessor.processActionPerform(RequestProcessor.java:431) [struts-1.2.9.jar:1.2.9]
  at org.apache.struts.action.RequestProcessor.process(RequestProcessor.java:236) [struts-1.2.9.jar:1.2.9]
  at org.apache.struts.action.ActionServlet.doPost(ActionServlet.java:432) [struts-1.2.9.jar:1.2.9]
  at javax.servlet.http.HttpServlet.service(HttpServlet.java:820) [servlet-api-2.5-6.1.12.jar:6.1.12]
  at org.mortbay.jetty.servlet.ServletHolder.handle(ServletHolder.java:502) [jetty-6.1.12.jar:6.1.12]
  at ch.qos.logback.demo.UserServletFilter.doFilter(UserServletFilter.java:44) [classes/:na]
  at org.mortbay.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1115) [jetty-6.1.12.jar:6.1.12]
  at org.mortbay.jetty.servlet.ServletHandler.handle(ServletHandler.java:361) [jetty-6.1.12.jar:6.1.12]
  at org.mortbay.jetty.webapp.WebAppContext.handle(WebAppContext.java:417) [jetty-6.1.12.jar:6.1.12]
  at org.mortbay.jetty.handler.ContextHandlerCollection.handle(ContextHandlerCollection.java:230) [jetty-6.1.12.jar:6.1.12]
```
启用打包数据：

```xml
<configuration packagingData="true">
    ...
</configuration>
```
### 直接调用 `JoranConfigurator`

Logback 依赖的配置文件库为 Joran，是 logback-core 的一部分。logback 的默认配置机制为：通过 `JoranConfigurator` 在类路径上寻找默认的配置文件。你可以通过直接调用 `JoranConfigurator` 的方式来重写 logback 的默认配置机制。

`Example`：直接调用 `JoranConfigurator`

```java
package chapters.configuration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import ch.qos.logback.access.joran.JoranConfigurator;
import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.core.joran.spi.JoranException;
import ch.qos.logback.core.util.StatusPrinter;

public class MyApp3 {

	private static final Logger LOGGER = LoggerFactory.getLogger(MyApp3.class);
	
	public static void main(String[] args) {
		
		LoggerContext context = (LoggerContext)LoggerFactory.getILoggerFactory();
		JoranConfigurator joranConfigurator = new JoranConfigurator();
		joranConfigurator.setContext(context);
		context.reset();
		try {
			joranConfigurator.doConfigure(args[0]);
		} catch (JoranException e) {
			e.printStackTrace();
		}
		
		StatusPrinter.printInCaseOfErrorsOrWarnings(context);
		
		LOGGER.info("Entering application");
		
		Foo foo = new Foo();
		foo.doIt();
		LOGGER.info("Exiting application");
	}
}

```

> 注意：对于多个步骤的配置，`context.reset()` 不需要调用（不是很理解这句话的意思）。

### 查看内部状态信息

logback 通过 [StatusManager](https://logback.qos.ch/xref/ch/qos/logback/core/status/StatusManager.html) 的对象来收集内部的状态信息，这个对象可以通过 `LoggerContext` 来获取。

对于一个给定的 `StatusManager`，你可以获取 logback 上下文所有的状态信息。为了保持内存的使用在一个合理的水平，`StatusManager` 的默认实现包含两个部分：头部与尾部。头部存储第一个 *H* 状态的消息，尾部存储最后一个 *T* 状态的消息。目前 *H=T=150*，这个值在以后可能会改变。

logback-classic 包含一个名叫 ViewStatusMessagesServlet 的 servlet。这个 servlet 打印当前 LoggerContext 的 StatusManager 的内容，通过 html 进行输出。

*Example*：

![statusMessages](/images/statusMessages.png)

在 *WEB-INF/web.xml* 中添加如下代码：

```xml
<servlet>
    <servlet-name>ViewStatusMessages</servlet-name>
	<servlet-class>ch.qos.logback.classic.ViewStatusMessagesServlet</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>ViewStatusMessages</servlet-name>
	<url-pattern>/lbClassicStatus</url-pattern>
</servlet-mapping>
```

然后可以通过 `http://host/yourWebapp/lbClassicStatus` 进行访问。

### 监听状态信息

通过给 `StatusManager` 附加一个 `StatusListener`，可以对状态信息进行获取。特别是在配置好 logback 之后。注册一个状态监听器可以很方便的监听 logback 的内部状态，并且不需要人工的干预。

 `StatusListener` 有一个名为 `OnConsoleStatusListener` 的实现类，可以将状态信息在控制台打印出来。

*Example*：

```java
public class AddStatusListenerApp {

	public static void main(String[] args) {

		LoggerContext lc = (LoggerContext)LoggerFactory.getILoggerFactory();
		StatusManager statusManager = lc.getStatusManager();
		OnConsoleStatusListener onConsoleStatusListener = new OnConsoleStatusListener();
		statusManager.add(onConsoleStatusListener);
		
		Logger logger = LoggerFactory.getLogger("myApp");
		logger.info("Entering application");
		
		Foo foo = new Foo();
		foo.doIt();
		logger.info("Exiting application");
		
		StatusPrinter.print(statusManager);
	}
}
```

> 注意：注册的状态监听器只会获取注册之后产生的状态消息，而不会获取注册之前产生的消息。所以建议在最开始的时候直接进行配置。

可以在配置文件中配置多个状态监听器。

*Example*：

```xml
<configuration>
  <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />  

  ... the rest of the configuration file  
</configuration>
```

### 系统属性 "logback.statusListenerClass"

通过设置 java 的系统属性来配置状态监听器。

*Example*：

```bash
java -Dlogback.statusListenerClass=ch.qos.logback.core.status.OnConsoleStatusListener
```

logback 子级实现了几个监听器。[OnConsoleStatusListener](https://logback.qos.ch/xref/ch/qos/logback/core/status/OnConsoleStatusListener.html) 用于在控制台打印状态消息。[OnErrorConsoleStatusListener](https://logback.qos.ch/xref/ch/qos/logback/core/status/OnErrorConsoleStatusListener.html) 用于在控制台打印显示错误的状态信息。[NopStatusListener](https://logback.qos.ch/xref/ch/qos/logback/core/status/NopStatusListener.html) 会丢弃掉状态信息。

> 注意：在配置期间，任何的状态监听器被注册，或者通过 java 系统变量指定 `logback.statusListenerClass` 的值，[在警告或错误的情况下自动打印状态信息](#在警告或错误的情况下自动打印状态信息) 将会被禁用。

可以通过设置 java 系统变量 `logback.statusListenerClass` 的值来禁用一切状态信息的打印。

```bash
java -Dlogback.statusListenerClass=ch.qos.logback.core.status.NopStatusListener
```

## 停止 logback-classic

为了释放 logback-classic 所使用的资源，停止使用 logger context 是一个好注意。停止 context 将会关闭所有在 logger 上定义的 appender，并且有序的停止正在活动的线程。

```java
import org.sflf4j.LoggerFactory;
import ch.qos.logback.classic.LoggerContext;
...

LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
loggerContext.stop();
```

上面的代码在 web 应用中，通过调用 `ServletContextListener` 的 [contextDestroyed](http://docs.oracle.com/javaee/6/api/javax/servlet/ServletContextListener.html#contextDestroyed(javax.servlet.ServletContextEvent)) 方法来停止 logback-classic 并释放资源。1.1.10 版本后，`ServletContextListener` 会被自动加载。

####  通过 shutddown hook 停止 logback-classic

> 个人觉得 hook 可以理解为钩子或者开关，但是还是觉得照写会更好理解一点。

指定一个 JVM shutdown hook 可以非常方便的关闭 logback 并释放资源。

```xml
<configuration debug="true">
    <!-- 如果缺失 class 属性，则会默认加载 ch.qos.logback.core.hook.DefaultShutdownHook -->
    <shutdownHook/>
</configuration>
```

> 注意：可以通过 *class* 属性指定一个 shutdown hook 的名字。

默认的 shutdown hook 为 [DefaultShutdownHook](https://logback.qos.ch/apidocs/ch/qos/logback/core/hook/DefaultShutdownHook.html)，在一个指定的时间后（默认是 0）会停掉 context。但是允许 context 在 30s 内完成日志文件的打包。在独立的 java 应用程序中，在配置文件中添加 `<shutdownHook/>` 可以确保任何日志打包任务完成之后，JVM 才会退出。在 web 应用程序中，[webShutdownHook](https://logback.qos.ch/manual/configuration.html#webShutdownHook) 会自动安装，`<shutdownHook/>` 将会变的多余且没有必要。

#### 在 web 应用中使用 WebShutdownHook 停止 logback-classic

`SINCE 1.1.10` logback-classic 会自动要求 web 服务安装 [LogbackServletContainerInitializer](https://logback.qos.ch/apidocs/ch/qos/logback/classic/servlet/LogbackServletContainerInitializer.html)（实现了 `ServletContainerInitializer` 接口，在 servlet-api 3.x 或以后的版本才有效）。这个初始化程序将会依次实例化 [LogbackServletContextListener](https://logback.qos.ch/apidocs/ch/qos/logback/classic/servlet/LogbackServletContextListener.html) 的实例。在 web 应用停止或者重载的时候会停掉当前 logback-classic 的 context。

> 我表示不是很懂这种做法有何意义，难道应用都停止了，context 还会在运行？这就是作者说的非常多余跟没有必要吗？

可以在 web.xml 中禁止 `LogbackServletContextListener` 的实例化。

*Example*：

```xml
<web-app>
	<context-param>
    	<param-name>logbackDisableServletContainerInitializer</param-name>
        <param-value>true</param-value>
    </context-param>
    ...
</web-app>
```

`logbackDisableServletContainerInitializer` 也可以通过 java 系统属性或者系统的环境变量来设置。优先级为：web 应用 > java 系统属性 > 系统环境变量

## 配置文件的语法

logback 允许你重新定义日志的行为而不需要重新编译代码，你可以轻易的禁用调应用中某些部分的日志，或者将日志输出到任何地方。

logback 的配置文件非常的灵活，不需要指定 DTD 或者 xml 文件需要的语法。但是，最基本的结构为 `<configuration>` 元素，包含 0 或多个 `<appender>` 元素，其后跟 0 或多个 `<logger>` 元素，其后再跟最多只能存在一个的 `<root>` 元素。基本结构图如下：

![basicSyntax](/images/basicSyntax.png)

#### 标签名大小写敏感

在 logback 版本 0.9.17 之后，显示规定的标签名不区分大小写。例如：`<logger>`、`<Logger`、`<LOGGER>` 这些都是有效的标签名。xml 风格的规则仍然适用。如果你有一个开始标签为 &lt;xyz\>，那么必须要有一个结束标签 &lt;/xyz&gt;。&lt;/XyZ&gt; 则是错误的。根据[默认规则](#默认规则)，标签名字是大小写敏感的，除了第一个字母。所以，&lt;xyz> 与 &lt;Xyz> 是一样的，但是 &lt;xYz> 是错误的。默认规则遵循驼峰命名法。很难说清楚一个标签遵循什么规则，如果你不知道给定的标签遵循哪种规则，那么使用驼峰命名法总是正确的。

#### 配置 logger

现在你至少应该对[等级继承规则](#有效等级又称为等级继承)与[基本规则](方法打印以及基本选择规则) 有所了解.。

通过 `<logger>` 标签来过 logger 进行配置，一个 `<logger>` 标签必须包含一个 *name* 属性，一个可选的 *level* 属性，一个可选 *additivity* 属性。`additivity` 的值为 *true* 或 *false*。`level` 的值为 TRACE，DEBUG，INFO，WARN，ERROR，ALL，OFF，INHERITED，NULL。当 `level` 的值为 INHERITED 或 NULL 时，将会强制 logger 继承上一层的级别。

`<logger>` 元素至少包含 0 或多个 `<appender-ref>` 元素。每一个 appender 通过这种方式被添加到 logger 上。与 log4j 不同的是，logbakc-classic 不会关闭或移除任何之前在 logger 上定义好的的 appender。

#### 配置 root logger

root logger 通过 `<root>` 元素来进行配置。它只支持一个属性——`level`。它不允许设置其它任何的属性，因为 additivity 并不适用 root logger。而且，root logger 的名字已经被命名为 "ROOT"，也就是说也不支持 name 属性。level 属性的值可以为：TRACE、DEBUG、INFO、WARN、ERROR、ALL、OFF，但是不能设置为 INHERITED  或 NULL。

跟 `<logger` 元素类似，`<root>` 元素可以包含 0 或多个 `<appender-ref>` 元素。

#### 例子

如果我们不想看到属于 "chapters.configuration" 组件中任何的 DEBUG 信息。

*Example*：sample2.xml

```xml
<configuration>
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
		</encoder>
	</appender>
	
	<logger name="chapters.configuration" level="INFO" />

	<root level="DEBUG">
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```

运行 *MyApp3*，可以看到如下的输出信息：

```java
21:52:48.726 [main] INFO  chapters.configuration.MyApp3 - Entering application
21:52:48.728 [main] INFO  chapters.configuration.MyApp3 - Exiting application
```

可以看到，["chapters.configuration.Foo"](https://logback.qos.ch/xref/chapters/configuration/Foo.html) 类中的 debug 信息没有被输出。

你可以配置任何 logger 的日志级别。在下一个例子中，我们设置 *chapters.configurations* 的 logger 日志级别为 INFO，同时设置 *chapters.configuration.Foo* 的 logger 日志级别为 DEBUG。

*Example*：sample3.xml

```xml
<configuration>
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>
				%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
			</pattern>
		</encoder>
	</appender>
	
	<logger name="chapters.configuration" level="INFO" />
	
	<logger name="chapters.configuration.Foo" level="DEBUG" />
	
	<root level="DEBUG">
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```

运行 *MyApp3* 可以看到如下的输出信息：

```java
22:06:43.500 [main] INFO  chapters.configuration.MyApp3 - Entering application
22:06:43.502 [main] DEBUG chapters.configuration.Foo - Did it again!
22:06:43.502 [main] INFO  chapters.configuration.MyApp3 - Exiting application
```

下面的表格列出了 `JoranConfigurator` 通过 *sample3.xml* 配置 logback 后，logger 以及其对应的日志级别。

Logger name | 指定级别 | 有效级别
:--|:-:|:-:
root | DEBUG | DEBUG 
chapters.configuration | INFO | INFO 
chapters.configuration.MyApp3 | null | INFO 
chapters.configuration.Foo | DEBUG | DEBUG 

`MyApp3` 类中的两条日志消息的级别都为 INFO，Foo.doIt() 中的 DEBUG 信息也能够进行输出。

> 注意：root logger 的日志级别永远不会设置成一个非空的值，默认是 DEBUG。

[基本选择法](#方法打印以及基本选择规则) 依赖被调用 logger 的有效日志级别，而并不是 appender 所依附的 logger 的级别。logback 会首先判断日志语句是否可以被打印，如果可以，则会在 logger 的层级结构中查找 appender，而不管它们的级别如何（表示很费解，appender 本来就没有日志级别，为什么会关 appender 的事？）。下面的例子说明了这一点。

*Example*：sample4.xml

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <logger name="chapters.configuration" level="INFO" />
    
    <root level="OFF">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

如下表格展示了应用 sample4.xml 之后的各 logger 的日志级别。

|logger name|分配级别|有效级别|
|----|----|----|
|root|OFF|OFF|
|chapters.configuration|INFO|INFO|
|chapters.configuration.MyApp3|null|INFO|
|chapters.configuration.Foo|null|INFO|

ConsoleAppender 的名字为 *STDOUT*， *sample4.xml* 中唯一的 appender，它所依附的 root logger 的 level = OFF。但是，运行 *MyApp3* 还是得到日志输出：

```java
10:47:34.310 [main] INFO  chapters.configuration.MyApp3 - Entering application
10:47:34.313 [main] INFO  chapters.configuration.MyApp3 - Exiting application
```

很明显，root logger 没有影响到其他的 logger，因为 `chapters.configuration.MyApp3` 与 `chapters.configuration.Foo` 类的日志级别为 INFO。即使在 java 代码中没有直接引用 *chapters.configuration* 这个 logger，但是它是存在，因为它在配置文件中声明了。

#### 配置 appender

appender 通过 `<appender>` 元素进行配置，需要两个强制的属性 *name* 与 *class*。*name* 属性用来指定 appender 的名字，*class* 属性需要指定类的全限定名用于实例化。`<appender>` 元素可以包含 0 或一个 `<layout>` 元素，0 或多个 `<encoder>` 元素，0 或多个 `<filter>` 元素。除了这些公共的元素之外，`<appender>` 元素可以包含任意与 appender 类的 JavaBean 属性相一致的元素。

下图展示了一些常见的结构。

> 注意：对属性的支持不可见（没懂这句话是什么意思）。

![appenderSyntax](/images/appenderSyntax.png)

`<layout>` 元素强制一个 class 属性去指定一个类的全限定名，用于实例化。与 `<appender>` 元素一样，`<layout>` 元素也可以包含与 layout 实例相关的属性。如果 layout 的 class 是 `PatternLayout`，那么 class 属性可以被隐藏掉（参考：[默认类映射](#默认类映射)），因为这个很常见。.

`<encoder` 元素强制一个 class 属性去指定一个类的全限定名，用于实例化。如果 encoder 的 class 是 `PatternLayoutEncoder`，那么基于[默认类映射](#默认类映射)，class 属性可以被隐藏。 

通过多个 appender 输出日志就像定义多个 appender 以及将它们关联到 logger 上一样简单。

*Example*：multiple.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>myApp.log</file>
		<encoder>
			<pattern>
				%date %level [%thread] %logger{10} [%file:%line] %msg%n
			</pattern>
		</encoder>
	</appender>
	
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>
				%msg%n
			</pattern>
		</encoder>
	</appender>
	
	<root level="debug">
		<appender-ref ref="FILE" />
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```

这个配置文件定义了两个 appender：*FILE* 和 *STDOUT*。*FILE* appender 将日志输出到 *myApp.log* 文件。encoder 通过 `PatternLayoutEncoder` 输出日期、日志等级、线程名、logger 的名字、可以定位日志来源的文件以及所在行、具体的日志信息以及行分隔符。第二个 appender 是 `STDOUT`，将日志输出到控制台。它的 encoder 仅仅输出日志信息以及行分隔符。

appender 通过 *appender-ref* 元素附加到 root logger 上。每一个 appender 都有自己 encoder。encoder 通常不会设计成给所有的 appender 共享。对于 layout 也是如此。因此，logback 不会提供任何共享 encoder 和 layout 的语法。

#### 重复使用 appender

在默认的情况下，appender 是可以重复使用的：logger 可以通过附加到本身的 appender 输出日志，同样的也可以附加到起祖先的身上，并输出日志。因此，如果同一个 appender 附加到多个 logger 身上，那么就导致日志重复打印。

*Example*：duplicate.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
		</encoder>
	</appender>
	
	<logger name="chapters.configuration">
		<appender-ref ref="STDOUT" />
	</logger>
	
	<root level="debug">
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```

运行 *MyApp3*，将会输出如下结果：

```java
22:43:35.469 [main] INFO  chapters.configuration.MyApp3 - Entering application
22:43:35.469 [main] INFO  chapters.configuration.MyApp3 - Entering application
22:43:35.471 [main] DEBUG chapters.configuration.Foo - Did it again!
22:43:35.471 [main] DEBUG chapters.configuration.Foo - Did it again!
22:43:35.472 [main] INFO  chapters.configuration.MyApp3 - Exiting application
22:43:35.472 [main] INFO  chapters.configuration.MyApp3 - Exiting application
```

注意日志重复输出了，因为 appender *STDOUT* 附加到了两个 logger 身上：root 以及 *chapters.configuration*。因为 root logger 是所有 logger 的祖先，*chapters.configuration* 是 *chapters.configuration.MyApp3* 以及 *chapters.configuraion.Foo* 的父级。每一次日志请求都会被打印两次，一次是通过 *STDOUT*，一次是通过 *root*。

appender 的叠加性并不是为新用户设置的陷阱。它是 logback 非常方便的一个特性。例如，你可以让系统中所有的日志输出到控制台上，而其它特定的日志输出到特定的 appender 中。

*Example*： restricted.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>myApp.log</file>
		<encoder>
			<pattern>%date %level [%thread] %logger{10} [%file:%line] %msg%n</pattern>
		</encoder>
	</appender>
	
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%msg%n</pattern>
		</encoder>
	</appender>
	
	<logger name="chapters.configuration">
		<appender-ref ref="FILE" />
	</logger>
	
	<root level="debug">
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```

在这个例子中，控制台会打印所有的日志，而只有属于 *chapters.configuration* 的 logger 以及它的子级 logger 的日志才会输出到 *myApp.log* 文件。

#### 重写默认的累加行为

如果默认的累积行为对你来说不适合，你可以设置 additivity = false。

*Example*：additivityFlag.xml

```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>foo.log</file>
        <encoder>
            <pattern>%date %level [%thread] %logger{10} [%file : %line] %msg%n</pattern>
        </encoder>
    </appender>
    
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    
    <logger name="chapters.configuration.Foo" additivity="false">
        <appender-ref ref="FILE" />
    </logger>
    
    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

在这个例子中，*FILE* appender 附加到了名为 *chaoters.configuration.Foo* 的 logger 上。而且，*chapters.configuration.Foo* 设置了 additivity = false，那么这个 logger 的日志将会通过 *FILE* 这个 appender 输出，但是它的父级 logger 将不会输出属于这个 logger 的日志。运行 *MyApp3*，属于 *chapters.configuration.MyApp3* 这个 logger 的日志将会在控制台输出，但是属于 *chapters.configuration.Foo* 这个 logger 的日志只会在 *foo.log* 这个文件看到。

### 设置 context 的名字

在之前的[章节](#Logger 上下文)中提到，每一个 logger 都会附加到一个 logger context 上去。默认这个 logger context 的名字为 "default"。但是你可以通过 `<contextName>` 设置其它的名字。但是如果设置过一次就不能[再设置](#https://logback.qos.ch/apidocs/ch/qos/logback/core/ContextBase.html#setName(java.lang.String))。当多个应用输出日志到同一个目的地，设置 logger context 的名字可以更好的区分。

*Example*：contextName.xml

```xml
<configuration>
    <contextName>myAppName</contextName>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d %contextName [%t] %level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

### 变量替换

**`注意`**：早期版本使用的是属性替换而不是变量替换

#### 变量的定义

logback 支持变量的定义以及替换，变量有它的作用域。而且，变量可以在配置文件中，外部文件中，外部资源文件中，甚至动态定义。

*Example*：variableSubstitution1.xml

```xml
<configuration>
	<property name="USER_NAME" value="/data/logs" />

	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>${USER_NAME}/myApp.log</file>
		<encoder>
			<pattern>%msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="debug">
		<appender-ref ref="FILE" />
	</root>	
</configuration>
```

这个例子中，在配置文件的开始定义了一个变量，之后通过引用这个变量指定了日志文件的路径。

*Example*：variableSubstitution2.xml

```xml
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>${USER_HOME}/myApp.log</file>
		<encoder>
			<pattern>%msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="debug">
		<appender-ref ref="FILE" />
	</root>	
</configuration>
```

这个例子中，在 java 的系统变量中定义一个同样的变量名，达到的效果是一样的。可以通过如下的方式去运行：

```
java -DUSER_HOME="/data/logs" MyApp3
```

当需要定义多个变量时，可以将这些变量放到一个单独的文件中。

*Example*：variableSubstitution3.xml

```xml
<configuration>
	<property file="F:\project\logback-examples\src\main\resources\variables1.properties"/>
	
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>${USER_HOME}/myApp.log</file>
		<encoder>
			<pattern>%msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="debug">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

这个配置文件包含了一个对外部文件的引用：*variables1.properties*。这个外部文件包含一个变量：

*Example*：variables1.properties

```properties
USER_HOME=/data/logs
```

也可以引用 classpath 下的资源文件：

```xml
<configuration>
	<property resource="resource1.properties" />
	
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>${USER_HOME}/myApp.log</file>
		<encoder>
			<pattern>%msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="debug">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

#### 作用域

属性的作用域分别为本地（local scope）、上下文（context scope）、系统（system scope）。默认为本地作用域。

`本地（local scope）`：本地范围内的属性存在配置文件的加载过程中。配置文件每加载一次，变量就会被重新定义一次。

`上下文（context scope）`：上下文范围内的属性会一直存在上下文被清除。

`系统（system scope）`：系统范围内的属性，会插入到 JVM 的系统属性中，跟随 JVM 一同消亡。

在进行变量替换的时候，会先从本地范围去找，再从上下文去找，再从系统属性中去找，最后会去系统的环境变量中去找。

可以通过 `<property>`、`<define>`、`<insertFromJNDI>` 元素的 *scope* 属性来设置变量的作用范围。*scope* 属性可能的值为：local，context，system。如果没有指定，则默认为 local。

*Example*：contextScopedVariable.xml

```xml
<configuration>
    <property scope="context" name="nodeId" value="firstNode"/>
    
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>/data/${nodeId}/myApp.log</file>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

在这个例子中，*nodeId* 这个变量被定义在上下文范围，它在每个日志事件，甚至通过序列化发送到远程服务器上都有效。

### 变量的默认值

在某些情况下，如果某个变量没有被声明，或者为空，默认值则非常有用。在 bash shell 中，默认值可以通过 **":-"** 来指定。例如：假设变量 *aName* 没有被定义，*"${aNme:-golden}"* 会被解释成 "golden" 。

### 变量的嵌套

变量的名字、默认值、以及值都可以引用其它的变量。

#### 嵌套值

一个变量的值可以包含对其它变量的引用。

*Example*：variables2.properties

```properties
USER_HOME=/data/logs
fileName=myApp.log
destination=${USER_HOME}/${fileName}
```

*Example*：variableSubsitution4.xml

```xml
<configuration>
    <!-- 注: 官网的例子不是 resource 而是 file -->
    <property resource="variables2.properties" />
    
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>${destination}</file>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

#### 名字嵌套

变量的名字可以包含对其它变量的引用。例如：如果变量 *userid=alice*，那么 "${${userid}.password}" 就是对变量名为 "alice.passowrd" 的引用。

#### 默认值嵌套

一个变量的默认值可以引用另一个变量。例如：假设变量 "id" 没有被定义，变量 "userid" 的值为 "alice"，那么表达式 "${id:-${userid}}" 的值为 "alice"。

### HOSTNAME 属性

`HOSTNAME` 在配置期间会被自动定义为上下文范围内。

*Example*：

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${HOSTNAME} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

### CONTEXT_NAME 属性

通过名字可以看出来，`CONTEXT_NAME` 属性对应当前上下文的名字。

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONTEXT_NAME} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

### 动态定义属性

可以通过 `<define>` 元素动态的定义变量。这个元素需要两个强制的属性：*name*、*class*。*name* 属性用来定义变量的名字，*classs* 属性用来引用实现了 [PropertyDefiner](https://logback.qos.ch/xref/ch/qos/logback/core/spi/PropertyDefiner.html) 接口的类。`PropertyDefiner` 实例的 `getPropertyValue()` 的返回值就是变量的值。还可以通过 *scope* 属性指定变量的[作用域](#作用域)。

```xml
<configuration>

  <define name="rootLevel" class="chapters.configuration.PropertyDefiner1">
    <shape>round</shape>
    <color>brown</color>
    <size>24</size>
  </define>
 
  <root level="${rootLevel}"/>
</configuration>
```

shape，color，size 都是 "chapters.configuration.PropertyDefiner1" 的属性。只要在实现类里面，各属性有对应的 set 方法，logback 就可以通过配置文件给各属性注入对应的值。

目前，logback 已经有了几个简单的实现类：

| 类名 | 描述 |
| :-: | :-- |
| [`CanonicalHostNamePropertyDefiner`](https://logback.qos.ch/apidocs/ch/qos/logback/core/property/CanonicalHostNamePropertyDefiner.html) | 将变量的值设置为本地的主机名。注意：获取主机名可能需要花费几秒的时间。 |
| [`FileExistsPropertyDefiner`](https://logback.qos.ch/apidocs/ch/qos/logback/core/property/FileExistsPropertyDefiner.html) | 如果通过 `path` 属性指定的文件存在，则设置变量为 "true"，否则设置为 "false"。 |
| [`ResourceExistsPropertyDefiner`](https://logback.qos.ch/apidocs/ch/qos/logback/core/property/FileExistsPropertyDefiner.html) | 如果通过 `resource` 属性指定的资源文件在类路径中存在，则设置变量为 "true"，否则设置为 "false"。 |

### 配置文件中的条件处理

开发者通常需要在多个环境中切换配置文件，例如：开发，测试和生产。这些配置文件有大量相同的地方，只有少数地方不同。为了避免重复，logback 在配置文件中支持通过 `<if>`、`<then>`、`<else>` 元素作为条件语句来区分不同的环境。条件处理需要 [Janino](https://logback.qos.ch/setup.html#janino) 环境的支持。

*Example*：

```xml
    <if condition="条件表达式">
        <then>
            ...
        </then>
	</if>
	
	<if condition="条件表达式">
        <then>
            ...
        </then>
        <else>
            ...
        </else>
	</if>
```

条件表达式只能是上下文变量或者系统变量。因为值是通过参数传递的，`property()` 方法或者其等价的 `p()` 方法属性的值。例如：如果要获取变量 "k" 的值，可以通过 `property("k")` 或者 `p("k")` 来获取。如果 "k" 没有定义，那么方法将会返回空字符串。所以不需要去判断是否为 null。

`isDefined()` 方法可以用来判断变量是否已经被定义。例如：可以通过 `isDefined("k")` 来判断 k 是否已经定义。还可以通过 `isNull()` 方法来判断变量是否为 null。例如：`isNull("k")`。

```xml
<configuration debug="true">
	<if condition='property("HOSTNAME").contains("volong")'>
		<then>
			<appender name="CON" class="ch.qos.logback.core.ConsoleAppender">
				<encoder>
					<pattern>%d %-5level %logger{35} - %msg %n</pattern>
				</encoder>
			</appender>
			<root>
				<appender-ref ref="CON" />
			</root>
		</then>
	</if>
	
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>${randomOutputDir}/conditional.log</file>
		<encoder>
			<pattern>${HOSTNAME} %d %-5level %logger{35} - %msg %n</pattern>
		</encoder>
	</appender>
	
	<root level="ERROR">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

条件处理语句与嵌套的 if-else 语句在 `<configuration>` 元素内都是可以使用的。但是 xml 的语法非常的繁琐，不适合作为通用变成语言的基础。因此，不建议使用过多的条件语句，因为别人看了难以理解，对你自己也是如此。

### 从 JNDI 中获取变量

在某些情况下，如果从 JNDI 中获取变量的值。`<insertFromJNDI>` 元素能够获取存储在 JNDI 中的元素并插入到本地的上下文中，然后通过 `as` 属性获取具体的值。还可以通过[作用域](#作用域)将变量插入到不同的作用域中。

*Example*：insertFromJNDI.xml

```xml
<!-- appName 的配置在 web.xml 中 
	使用的是 eclipse 的 jetty 插件运行的
	运行时候的配置：右键点击项目 -> Run Configurations -> Jetty 选项卡下选择 Show Advanced Options -> 选中 JNDI Support -> 在运行就可以了
-->
<configuration>
	<insertFromJNDI env-entry-name="java:comp/env/appName" as="appName" />
	<contextName>${appName}</contextName>

	<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%d %contextName %level %msg %logger{50}%n</pattern>
		</encoder>
	</appender>

	<root level="DEBUG">
		<appender-ref ref="CONSOLE" />
	</root>
</configuration>
```

### 引入文件

通过 `<include>` 元素可以引入外部的配置文件。

*Example*：containingConfig.xml

```xml
<configuration>
    <include file="src/main/resources/includedConfig.xml" />
    
    <root level="DEBUG">
        <appender-ref ref="includedConsole" />
    </root>
</configuration>
```

目标文件必须是由 `<included>` 元素包裹的。

*Example*：includedConfig.xml

```xml
<included>
    <appender name="includedConsole" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d - %m%n</pattern>
        </encoder>
    </appender>
</included>
```

可以通过如下几个属性引入文件：

- **通过文件引入**：

  可以通过 `file` 属性引入外部文件。可以通过相对路径或者绝对路径来引入。相对路径是指相对应用程序的路径。

- **通过资源文件引入**

  可以通过 `resource` 属性来引入位于 classpath 路径下的资源文件。

  ```xml
  <include resource="includedConfig.xml"/>
  ```


- **通过 url 引入文件**

  可以通过 `url` 属性来引入外部文件。

  ```xml
  <include url="http://some.host.com/includedConfig.xml"/>
  ```

如果 logback 没有通过 `include` 元素找到指定的配置文件，会在控制台打印出内部状态信息。如果引入的外部配置文件是可选的，可以设置 `optional=true`。

```xml
<include optional="true" ..../>
```

## 添加上下文监听器

[LoggerContextListener](https://logback.qos.ch/xref/ch/qos/logback/classic/spi/LoggerContextListener.html) 接口的实例监听上下文生命周期内的事件。

`JMXConfigurator` 是 `LoggerContextListener` 接口的一个实现。

### 更改传播级别

在 0.9.25 版本，logback-classic 通过 `LoggerContextListener` 的实现类 [LevelChangePropagator](https://logback.qos.ch/xref/ch/qos/logback/classic/jul/LevelChangePropagator.html) 来更改 logback-classic 中的 logger 传播到 java.util.logging 中的日志级别。这些传播消除了禁止打印日志时的性能损耗。[LogRecord](http://download.oracle.com/javase/1.5.0/docs/api/java/util/logging/LogRecord.html?is-external=true) 实例仅仅会在允许打印日志的情况下通过 SFL4J 传播到 logback。

配置 `LevelChangePropagator`：

```xml
<configuration debug="true">
  <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator"/>
  .... 
</configuration>
```

`resetJUL ` 属性会重置 j.u.l 中的所有 logger 的等级配置。但是之前配置的将不会受到影响。

```xml
<configuration debug="true">
  <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
    <resetJUL>true</resetJUL>
  </contextListener>
  ....
</configuration>
```

# 第四章：Appenders

## 什么是 Appender

logback 将写入日志事件的任务委托给一个名为 appender 的组件。Appender 必须实现 [`ch.qos.logback.core.Appender`](https://logback.qos.ch/xref/ch/qos/logback/core/Appender.html) 接口。该接口的方法如下：

```java
package ch.qos.logback.core;
  
import ch.qos.logback.core.spi.ContextAware;
import ch.qos.logback.core.spi.FilterAttachable;
import ch.qos.logback.core.spi.LifeCycle;
  

public interface Appender<E> extends LifeCycle, ContextAware, FilterAttachable {

	public String getName();
  	public void setName(String name);
  	void doAppend(E event);
}
```

`doAppender()` 方法接收一个泛型参数 *E* 作为唯一的参数。*E* 的实际参数类型取决于 logback 模块。在 logback-classic 模块里面，*E* 的类型是 [ILoggingEvent](https://logback.qos.ch/apidocs/ch/qos/logback/classic/spi/ILoggingEvent.html) 。在 logback-access 模块里面，*E* 的类型是 [AccessEvent](https://logback.qos.ch/apidocs/ch/qos/logback/access/spi/AccessEvent.html)。`doAppend()` 是 logback 框架里面最重要的模块。它的责任是将日志事件进行格式化，然后输出到对应的设备上。

Appender 都是实体类，这样可以确保它们通过名字被引用。`Appender` 接口继承了  `FilterAttachable` 接口。使得一个或多个过滤器可以附加到 appender 实例上。

Appender 最基本的责任是将日志事件进行输出。然而，它们可以委托 `Layout` 或者 `Encoder` 对象来对日志事件进行格式化。每一个 layout/encoder 有且只与一个 appender 相关联。





















