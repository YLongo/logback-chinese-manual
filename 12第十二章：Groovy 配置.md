领域特定语言或者 DSL 更加普遍。logback 基于 XML 的配置可以看做 DSL 的实例。由于 XML 的本质，基于 XML 的配置文件变得非常的啰嗦以及臃肿。另外，logback 中的 Joran 有一个相对庞大的代码，用来专门处理基于 XML 的配置文件。Joran 支持一些非常好的特性，例如变量替换，条件处理，以及动态扩展。但是，不但 Joran 非常复杂，而且给用户的体验非常的不好，或者至少不直观。

本章叙述基于 Groovy 的 DSL 致力于一致性，直观性，以及非常强大。任何你可以使用 XML 配置的文件，你都可以用更加简短的符号使用 Groovy 来实现。为了帮助你迁移到 Groovy 风格的配置，我们开发了一个[工具](https://logback.qos.ch/translator/asGroovy.html)。

## 常规建议

一般来说，*logback.groovy* 文件是 Groovy 程序。因为 Groovy 是 Java 的超集，所以无论你在 Java 执行什么配置操作，你都可以在 *logback.groovy* 文件种做同样的事情。