Joran 代表寒冷的西北风，常常猛烈的吹在日列瓦湖上。位于西欧中部的日列瓦湖，表面上看起来比其它许多欧洲的湖泊都要小。但是它的平均深度有 153 米，异常的深。并且，它是西欧最大的淡水湖。

正如前几章所示，logback 基于 Joran，一个成熟的，灵活的并且强大的配置框架。logback 提供的许多的功能，只能基于 Joran 来实现。这章将专注于 Joran 的基本设计以及一些明显的特征。

Joran 实际上是一个通用的配置系统，能够被独立用于日志记录。为了强调这一点，我们需要说明的是 logback-core 模块没有 logger 的概念。所以，本章的大多数示例与 logger，appender，layout 无关。

本章节中的示例可以在 *LOGBACK_HOME/logback-examples/src/main/java/chapters/onJoran/* 文件夹下被找到。

要安装 Joran，只需要[下载](https://logback.qos.ch/download.html)，然后将 *logback-core-1.3.0-alpha4.jar* 放到类路径下。

## 历史回顾

反射是 Java 语言一个强大的特性，使得声明式的配置软件系统变成可能。例如，EJB 许多重要的属性都被配置在 *ejb.xml* 文件中。尽管 EJB 是用 Java 编写的，但是它们的许多属性都是通过  *ejb.xml* 来指定的。类似的，logback 也可以通过指定的 XML 格式的配置文件来进行设置。JDK 1.5 中的注解在 EJB 3.0 被大量使用用来替换之前 XML 文件中的许多指令。Joran 也会充分利用注解，但是使用的范围少的多。由于 logback 配置的动态特性 (相比 EJB)，Joran 使用注解相当有限。

在 logback 它爹 log4j 中， `DOMConfigurator` 类是 log4j 1.2.x 以及以后的版本的一部分。也能够解析 XML 的配置文件。`DOMConfigurator` 的编写方式强迫开发人员在配置文件的结构每次发生改变时，需要重新调整代码。调整的代码需要重新编译并重新部署。同样重要的是，`DOMConfigurator` 的代码由循环组成，用于解析子元素，包含了许多 if/else 语句。这样的代码散发着冗余以及重复的味道。 [commons-digester](http://jakarta.apache.org/commons/digester/) 告诉我们可以通过模式匹配规则来解析 XML 文件。在解析的时候，解析器会应用匹配了指定模式的规则。规则类通常比较小，并且具有专业性。因此，理解与维护相对简单。

有了 `DOMConfigurator` 的经验，我们开始开发 `Joran`，一个在 logback 中使用的、强大的配置框架。Joran 受到了 commons-digester 项目很大的启发。但是，它使用了一个稍微不同术语。在 commons-digester 中，规则可以看座由模式组成，并且规则可以通过 `Digester.addRule(String pattern, Rule rule)` 方法组成。

