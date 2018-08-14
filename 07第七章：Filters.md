在之前的章节中介绍的[方法打印以及基本选择规则](https://github.com/Volong/logback-chinese-manual/blob/master/02%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E6%9E%B6%E6%9E%84.md#%E6%96%B9%E6%B3%95%E6%89%93%E5%8D%B0%E4%BB%A5%E5%8F%8A%E5%9F%BA%E6%9C%AC%E9%80%89%E6%8B%A9%E8%A7%84%E5%88%99)是 logback-classic 的核心。在这章中，将介绍其它的过滤方法。

logback 过滤器基于三元逻辑，允许它们组装或者链接在一起组成一个任意复杂的过滤策略。它们在很大程度上受到 Linux iptables 的启发。

## 在 logback-classic 中

在 logback-classic 中，有两种类型的过滤器，regular 过滤器以及 turbo 过滤器。

### Regular 过滤器

reqular 过滤器继承自 [`Filter`](https://logback.qos.ch/xref/ch/qos/logback/core/filter/Filter.html) 这个抽象类。本质上它由一个单一的 `decide()` 方法组成，接收一个 `ILoggingEvent` 实例作为参数。

