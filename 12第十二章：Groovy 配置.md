领域特定语言或者 DSL 更加普遍。logback 基于 XML 的配置可以看做 DSL 的实例。由于 XML 的本质，基于 XML 的配置文件变得非常的啰嗦以及臃肿。另外，logback 中的 Joran 有一个相对庞大的代码，用来专门处理基于 XML 的配置文件。Joran 支持一些非常好的特性，例如变量替换，条件处理，以及动态扩展。但是，不但 Joran 非常复杂，而且给用户的体验非常的不好，或者至少不直观。

本章叙述基于 Groovy 的 DSL 致力于一致性，直观性，以及非常强大。任何你可以使用 XML 配置的文件，你都可以用更加简短的符号使用 Groovy 来实现。为了帮助你迁移到 Groovy 风格的配置，我们开发了一个[工具](https://logback.qos.ch/translator/asGroovy.html)。

## 常规建议

一般来说，*logback.groovy* 文件是 Groovy 程序。因为 Groovy 是 Java 的超集，所以无论你在 Java 执行什么配置操作，你都可以在 *logback.groovy* 文件中做同样的事情。但是，在 Java 中，使用变成的方式配置 logback 有点笨重，所以我们增加了一些 logback 特有的扩展来减轻你的负担。我们尝试限制 logback 特有的拓展符号尽量的少。如果你已经熟悉了 Groovy，那么你应该更加容易去读，去理解甚至去写你自己的 *logback.groovy* 文件。那么不熟悉 Groovy 的人依然会发现 *logback.groovy* 中的语法比 *logback.xml* 中的语法更加容易使用。

*logback.groovy* 文件是 Groovy 程序，具有最小的 logback 特定的拓展。所有常用的 groovy 结构，例如类的导入，变量定义，字符串 (GString) 中包含 \${..} 评估表达式，以及 if-else 语句在 *logback.grooby* 文件中都是可用的。

## 自动导入

**`1.0.10 以后`** 为了减少不必要的引用