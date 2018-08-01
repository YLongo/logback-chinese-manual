## 什么是 encoder

encoder 将日志事件转换为字节数组，同时将字节数组写入到一个 `OutputStream` 中。encoder 在 logback 0.9.19 版本引进。在之前的版本中，大多数的 appender 依赖 layout 将日志事件转换为 string，然后再通过 `java.io.Writer` 写出。在之前的版本中，用户需要在 `FileAppender` 中内置一个 `PatternLayout`。在 0.9.19 之后的版本中，`FileAppender` 以及子类[需要一个 encoder 而不是 layout](https://logback.qos.ch/codes.html#layoutInsteadOfEncoder)。

为什么会有这个改变？

layout 将会在下一章节讨论，它只能将日志事件转换为成 string。而且，考虑到 layout 在日志事件写出时不能控制日志事件，不能将日志事件批量聚合。与之相反的是，encoder 不但可以完全控制字节写出时的格式，而且还可以控制这些字节什么时候被写出。



