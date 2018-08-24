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

MDC 在客户端服务器架构中最为重要。通常，服务器上的多个线程为多个客户端提供服务。尽管 MDC 中的方法是静态的，但是是以每个线程为基础来进行管理的，允许每个服务线程都打上一个 `MDC` 标记。`MDC` 中的 `put()` 与 `get()` 操作仅仅只影响当前线程中的 `MDC`。其它线程中 `MDC` 不会受到影响。给定的 `MDC` 信息在每个线程的基础上进行管理。每个线程都有一份 `MDC` 的拷贝。因此，在对 `MDC` 进行编程时，开发人员没有必要去考虑线程安全或者同步问题。它自己会安全的处理这些问题。

下面有一个稍微高级一点的例子。它显示了如何在客户端-服务端上设置 `MDC`。例 7.2 展示的是服务端实现 `NumberCruncher` 接口。`NumberCruncher` 接口仅仅只包含一个方法 `factor()`。客户端使用 RMI 技术，调用服务器应用程序上的 `factor()` 方法来获取一个整数的不同因子。

> Example 7.2 *NumberCruncher.java*

```java
package chapters.mdc;

import java.rmi.Remote;
import java.rmi.RemoteException;

public interface NumberCruncher extends Remote {
  int[] factor(int number) throws RemoteException;
}
```

例 7.3 展示了 `NumberCruncherServer` 实现了 `NumberCruncher` 接口。它的主要方法是在本地主机上导出一个 RMI 注册表，并在一个众所周知的端口号上接受请求 (反正我是不知道😓)。

> *Example 7.3* *NumberCruncherServer.java*

```java
package chapters.mdc;

import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;
import java.util.Vector;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.classic.joran.JoranConfigurator;
import ch.qos.logback.core.joran.spi.JoranException;


/**
 * A simple NumberCruncher implementation that logs its progress when
 * factoring numbers. The purpose of the whole exercise is to show the
 * use of mapped diagnostic contexts in order to distinguish the log
 * output from different client requests.
 * */
public class NumberCruncherServer extends UnicastRemoteObject
  implements NumberCruncher {

  private static final long serialVersionUID = 1L;

  static Logger logger = LoggerFactory.getLogger(NumberCruncherServer.class);

  public NumberCruncherServer() throws RemoteException {
  }

  public int[] factor(int number) throws RemoteException {
    // The client's host is an important source of information.
    try {
      MDC.put("client", NumberCruncherServer.getClientHost());
    } catch (java.rmi.server.ServerNotActiveException e) {
      logger.warn("Caught unexpected ServerNotActiveException.", e);
    }

    // The information contained within the request is another source
    // of distinctive information. It might reveal the users name,
    // date of request, request ID etc. In servlet type environments,
    // useful information is contained in the HttpRequest or in the  
    // HttpSession.
    MDC.put("number", String.valueOf(number));

    logger.info("Beginning to factor.");

    if (number <= 0) {
      throw new IllegalArgumentException(number +
        " is not a positive integer.");
    } else if (number == 1) {
      return new int[] { 1 };
    }

    Vector<Integer> factors = new Vector<Integer>();
    int n = number;

    for (int i = 2; (i <= n) && ((i * i) <= number); i++) {
      // It is bad practice to place log requests within tight loops.
      // It is done here to show interleaved log output from
      // different requests. 
      logger.debug("Trying " + i + " as a factor.");

      if ((n % i) == 0) {
        logger.info("Found factor " + i);
        factors.addElement(new Integer(i));

        do {
          n /= i;
        } while ((n % i) == 0);
      }

      // Placing artificial delays in tight loops will also lead to
      // sub-optimal results. :-)
      delay(100);
    }

    if (n != 1) {
      logger.info("Found factor " + n);
      factors.addElement(new Integer(n));
    }

    int len = factors.size();

    int[] result = new int[len];

    for (int i = 0; i < len; i++) {
      result[i] = ((Integer) factors.elementAt(i)).intValue();
    }

    // clean up
    MDC.remove("client");
    MDC.remove("number");

    return result;
  }

  static void usage(String msg) {
    System.err.println(msg);
    System.err.println("Usage: java chapters.mdc.NumberCruncherServer configFile\n" +
      "   where configFile is a logback configuration file.");
    System.exit(1);
  }

  public static void delay(int millis) {
    try {
      Thread.sleep(millis);
    } catch (InterruptedException e) {
    }
  }

  public static void main(String[] args) {
    if (args.length != 1) {
      usage("Wrong number of arguments.");
    }

    String configFile = args[0];

    if (configFile.endsWith(".xml")) {
      try {
        LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
        JoranConfigurator configurator = new JoranConfigurator();
        configurator.setContext(lc);
        lc.reset();
        configurator.doConfigure(args[0]);
      } catch (JoranException je) {
        je.printStackTrace();
      }
    }

    NumberCruncherServer ncs;

    try {
      ncs = new NumberCruncherServer();
      logger.info("Creating registry.");

      Registry registry = LocateRegistry.createRegistry(Registry.REGISTRY_PORT);
      registry.rebind("Factor", ncs);
      logger.info("NumberCruncherServer bound and ready.");
    } catch (Exception e) {
      logger.error("Could not bind NumberCruncherServer.", e);

      return;
    }
  }
}
```

`factor(int number)` 的实现特别的相关。它在启动的时候将客户端的名字通过 key *client* 放到 `MDC` 中。客户端通过请求将整数放到 `MDC` 中的 *number* 这个 key 下。在计算出整数的不同因子之后，然后将结果返回给客户端。在返回结果之前，通过调用 `MDC.remove()` 清除 *client* 与 *number* 的值。通常，`put()` 操作应该由 `remove()` 操作来平衡。否则的话，`MDC` 将会保留旧值。我们推荐无论何时，在 finally 代码块中调用 `remove` 操作，用来确保该操作的执行，而不用关心代码的具体执行路径。

在解释完理论之后，我们准备运行这个例子。通过一下命令运行这个服务：

```bash
java chapters.mdc.NumberCruncherServer src/main/java/chapters/mdc/mdc1.xml
```

*mdc1.xml*  的内容如下：

> Example: *mdc1.xml* 

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <layout>
      <Pattern>%-4r [%thread] %-5level C:%X{client} N:%X{number} - %msg%n</Pattern>
    </layout>       
  </appender>
  
  <root level="debug">
    <appender-ref ref="CONSOLE"/>
  </root>  
</configuration>
```

注意 *%X* 转换符在 `Pattern` 选项中使用。

通过以下命令执行 `NumberCruncherClient` 应用：

```bash
java chapters.mdc.NumberCruncherClient hostname
```

*hostname* 是 `NumberCruncherServer` 运行的主机名。

在客户端执行多个实例，第一个客户端请求服务器对 129 进行因数分解，随后，第二个客户端请求服务器对 71 进行因数分解。服务器的输出如下：

```java
70984 [RMI TCP Connection(4)-192.168.1.6] INFO  C:orion N:129 - Beginning to factor.
70984 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 2 as a factor.
71093 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 3 as a factor.
71093 [RMI TCP Connection(4)-192.168.1.6] INFO  C:orion N:129 - Found factor 3
71187 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 4 as a factor.
71297 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 5 as a factor.
71390 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 6 as a factor.
71453 [RMI TCP Connection(5)-192.168.1.6] INFO  C:orion N:71 - Beginning to factor.
71453 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 2 as a factor.
71484 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 7 as a factor.
71547 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 3 as a factor.
71593 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 8 as a factor.
71656 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 4 as a factor.
71687 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 9 as a factor.
71750 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 5 as a factor.
71797 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 10 as a factor.
71859 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 6 as a factor.
71890 [RMI TCP Connection(4)-192.168.1.6] DEBUG C:orion N:129 - Trying 11 as a factor.
71953 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 7 as a factor.
72000 [RMI TCP Connection(4)-192.168.1.6] INFO  C:orion N:129 - Found factor 43
72062 [RMI TCP Connection(5)-192.168.1.6] DEBUG C:orion N:71 - Trying 8 as a factor.
72156 [RMI TCP Connection(5)-192.168.1.6] INFO  C:orion N:71 - Found factor 71
```

从上面的输出可以看出客户端运行在一个叫做 *orion* 的主机上。尽管服务器在独立的线程中几乎同时处理客户端的请求，但是可以通过 `MDC` 来区分每个客户端的日志输出。示例中的是通过 *number* 来进行标识。

细心的读者可能已经注意到，可以通过线程名来区分每个请求。如果服务端重复使用线程，那么使用线程名可能会导致一下误解。也就是一个给定线程在服务完请求之后，接着去服务下一个请求，在这种情况下，很难去区分每个请求。因为 `MDC` 是受应用开发人员的控制，所以不会受这个影响。(是吗？开发人员就这么可靠吗？)

### 自动获取 `MDC`

正如我们之前看到的，对多个客户端进行请求时，`MDC`  非常有用。在 web 应用中管理用户认证，一个简单的处理方式是在 `MDC` 中设置用户名，在用户退出时进行移除。不幸的是，使用这个技术并不总是可靠的。因为 `MDC` 是在单线程的基础进行管理，服务器重复使用线程可能会导致 `MDC` 包含错误的信息。

为了在处理请求时，`MDC` 中包含的信息一直都是正确的。一种可能的处理方式是在开始处理之前存储用户名，结束时进行移除。[`Filter`](http://java.sun.com/javaee/5/docs/api/javax/servlet/Filter.html) 可以用来处理这种情况。

在 servlet 过滤器 `doFilter` 方法中，我们可以通过 request (或者 cookie) 获取用户的相关信息，并存到 `MDC` 中。其它过滤器或者 servlet 后续的处理会从 `MDC` 中受益。最后，当我们的 servlet 过滤器再次获得控制时，就有机会去清除 MDC 中的数据。

下面是一个过滤器的实现：

>   Example: *UserServletFilter.java*

```java
package chapters.mdc;

import java.io.IOException;
import java.security.Principal;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.slf4j.MDC;

public class UserServletFilter implements Filter {

  private final String USER_KEY = "username";
  
  public void destroy() {
  }

  public void doFilter(ServletRequest request, ServletResponse response,
    FilterChain chain) throws IOException, ServletException {

    boolean successfulRegistration = false;

    HttpServletRequest req = (HttpServletRequest) request;    
    Principal principal = req.getUserPrincipal();
    // Please note that we could have also used a cookie to 
    // retrieve the user name

    if (principal != null) {
      String username = principal.getName();
      successfulRegistration = registerUsername(username);
    } 

    try {
      chain.doFilter(request, response);
    } finally {
      if (successfulRegistration) {
        MDC.remove(USER_KEY);
      }
    }
  }

  public void init(FilterConfig arg0) throws ServletException {
  }
  

  /**
   * Register the user in the MDC under USER_KEY.
   * 
   * @param username
   * @return true id the user can be successfully registered
   */
  private boolean registerUsername(String username) {
    if (username != null && username.trim().length() > 0) {
      MDC.put(USER_KEY, username);
      return true;
    }
    return false;
  }
}
```

当过滤器的 `doFilter` 被调用时，它首先会在 request 中去寻找 `java.security.Principal` 对象。这个对象包含了当前认证用户的名字。如果找到了用户的信息，会将它注册到 `MDC` 中。

一个过滤器链执行完成，那么这个过滤器会从 `MDC` 中移除用户信息。

刚才我们描述的方法只是在请求期间，或者线程处理中才去设置 MDC 的值。其它的线程不会受到影响。而且，任何给定的线程在任何时候都会包含正确的 MDC 数据。

### MDC 与线程管理

MDC 的副本不能总是由工作线程从初始化线程继承。当使用 `java.util.concurrent.Executors` 进行线程管理时就是这种情况。例如，`newCachedThreadPool` 方法会创建一个 `ThreadPoolExecutor`，跟其它线程池代码一样，有着复杂的线程创建逻辑。

在这些情况下，推荐在提交任务去执行之前，在源线程上调用 `MDC.getCopyOfContextMap()`。当任务运行的时候，它的第一个动作，应该是调用 `MDC.setContextMapValues()` 将原始 MDC 值的存储副本与新的 `Executor` 管理线程关联起来。

### MDCInsertingServletFilter

在 web 应用中，获取给定 HTTP 请求的主机名，请求 URI 以及 user-agent 是十分有用的。[`MDCInsertingServletFilter`](https://logback.qos.ch/xref/ch/qos/logback/classic/helpers/MDCInsertingServletFilter.html) 通过如下的 key，将数据插入到 MDC 中。

| MDC key             | MDC value                                                    |
| ------------------- | ------------------------------------------------------------ |
| `req.remoteHost`    | 由 [getRemoteHost()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/ServletRequest.html#getRemoteHost%28%29) 返回 |
| `req.xForwardedFor` | 请求头 ["X-Forwarded-For"](http://en.wikipedia.org/wiki/X-Forwarded-For) 的值 |
| `req.method`        | 由 [getMethod()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getMethod%28%29) 返回 |
| `req.requestURI`    | 由 [getRequestURI()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getRequestURI%28%29) 返回 |
| `req.requestURL`    | 由 [getRequestURL()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getRequestURL%28%29) 返回 |
| `req.queryString`   | 由 [getQueryString()](http://java.sun.com/j2ee/sdk_1.3/techdocs/api/javax/servlet/http/HttpServletRequest.html#getQueryString%28%29) 返回 |
| `req.userAgent`     | 请求头 "User-Agent" 的值                                     |

想要使用 `MDCInsertingServletFilter`，需要在 *web.xml* 中添加以下行：

```xml
<filter>
  <filter-name>MDCInsertingServletFilter</filter-name>
  <filter-class>
    ch.qos.logback.classic.helpers.MDCInsertingServletFilter
  </filter-class>
</filter>
<filter-mapping>
  <filter-name>MDCInsertingServletFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping> 
```

**如果你的 web 应用有多个过滤器，请确保 `MDCInsertingServletFilter` 在其它过滤器之前声明 **。例如，假设 web 应用的主要处理是在过滤器 'F' 中完成，那么如果 `MDCInsertingServletFilter` 在 'F' 之后被调用，那么在 'F' 中被调用的代码将看不到 `MDCInsertingServletFilter` 设置的 MDC 的值。(这不是废话吗......)

一旦使用了该过滤器，通过 %X 可以输出相对应的 MDC 的值。例如，想要在一行上打印远程主机，后面跟着请求的 URI，令一行打印日志，后面跟着日志消息。那么你应该如下设置 `PatternLayout`：

```xml
%X{req.remoteHost} %X{req.requestURI}%n%d - %m%n
```



