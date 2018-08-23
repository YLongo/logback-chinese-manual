logback 设计的目标之一是审计与调试复杂的分布式应用。大部分的分布式系统需要同时处理多个客户端。在一个系统典型的多线程实现中，不同的线程处理不同的客户端。一种可能但是不建议的方式是在每个客户端实例化一个新的且独立的 logger，来区分一个客户端与另一个客户端的日志输出。这种方式会导致 logger 急剧增加并且会增加维护成本。

一种轻量级的技术是给每个为客户端服务的 logger 打一个标记。Neil Harrison  在 *Patterns for Logging Diagnostic Messages* in Pattern Languages of Program Design 3, edited by R. Martin, D. Riehle, and F. Buschmann (Addison-Wesley, 1997) 这本书中描述了这种方法。logback 在 SLF4J API 利用了这种技术的变体：诊断上下文映射 (MDC)。

为了给每个请求打上唯一的标记，用户需要将上下文信息放到 `MDC` (Mapped Diagnostic Context 的缩写) 中。下面列出了 MDC 类中主要的部分。完成的方法列表请查看 [MDC javadocs](http://www.slf4j.org/api/org/slf4j/MDC.html)。

```java
package org.slf4j;

public class MDC {
  // 将上下文的值作为 MDC 的 key 放到线程上下的 map 中
  public static void put(String key, String val);

  // 通过 key 获取上下文标识
  public static String get(String key);

  // 通过 key 移除上下文标识
  public static void remove(String key);

  // 清除 MDC 中所有的 entry
  public static void clear();
}
```

`MDC` 类中只包含静态方法。它让开发人员可以在 *诊断上下文* 中放置信息，而后通过特定的 logback 组件去获取。`MDC` 在 *每个线程的基础上* 管理上下文信息。通常，当为一个新客户端启动服务时，开发人员会将特定的上文信息插入到 MDC 中。例如，客户端 id，客户端 IP 地址，请求参数等。如果 logback 组件配置得当的话，会自动在每个日志条目中包含这些信息。

请注意，logback-classic 实现的 MDC，假设值以适当的频率放置。还需注意的一点是，子线程不会自动继承父线程的 MDC。

下面的 [SimpleMDC](https://logback.qos.ch/xref/chapters/mdc/SimpleMDC.html) 说明了这点。

>   Example: *[SimpleMDC.java](https://logback.qos.ch/xref/chapters/mdc/SimpleMDC.html)*

```java
package chapters.mdc;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

import ch.qos.logback.classic.PatternLayout;
import ch.qos.logback.core.ConsoleAppender;

public class SimpleMDC {
  static public void main(String[] args) throws Exception {

    // 你可以选择在任何时候将值放入 MDC 中    
    MDC.put("first", "Dorothy");

    [ SNIP ]
    
    Logger logger = LoggerFactory.getLogger(SimpleMDC.class);
    
    MDC.put("last", "Parker");
    
    logger.info("Check enclosed.");
    logger.debug("The most beautiful two words in English.");

    MDC.put("first", "Richard");
    MDC.put("last", "Nixon");
    logger.info("I am not a crook.");
    logger.info("Attributed to the former US president. 17 Nov 1973.");
  }

  [ SNIP ]

}
```

main 方法在启动的时候在 `MDC` 中将 *Dorothy* 关联到 *first* 上。你可以在 `MDC` 放置尽可能多的键值对，反正你开心就好。多次插入同一个 key，新值会覆盖旧值。然后代码继续配置 logback。

为了简洁，我们隐藏了配置文件 [simpleMDC.xml](http://github.com/qos-ch/logback/blob/master/logback-examples/src/main/java/chapters/mdc/simpleMDC.xml) 中的其它配置。下面的一些相关的配置。

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender"> 
  <layout>
    <Pattern>%X{first} %X{last} - %m%n</Pattern>
  </layout> 
</appender>
```

注意 *X%* 转换符在 `PatternLayout` 中的用法。*X%* 转换符使用了两次，一次用于 key 为 *first*，一次用于 key 为 *last*。在获得了 `SimpleMDC.class` 中的 logger 之后，通过代码将 *Parker* 关联到 *last* 上。然后通过 logger 打印了两条不同的信息。最后，通过代码重新设置了 `MDC` 为不同的值，并打印了几条日志请求。运行 SimpleMDC 将会输出：

```java
Dorothy Parker - Check enclosed.
Dorothy Parker - The most beautiful two words in English.
Richard Nixon - I am not a crook.
Richard Nixon - Attributed to the former US president. 17 Nov 1973.
```

`SimpleMDC` 展示了正确配置 logback layout 就可以自动输出 `MDC` 的信息。而且，`MDC` 中信息可以被多个 logger 使用。

### 高级用法

