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