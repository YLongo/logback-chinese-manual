领域特定语言或者 DSL 更加普遍。logback 基于 XML 的配置可以看做 DSL 的实例。由于 XML 的本质，基于 XML 的配置文件变得非常的啰嗦以及臃肿。另外，logback 中的 Joran 有一个相对庞大的代码，用来专门处理基于 XML 的配置文件。Joran 支持一些非常好的特性，例如变量替换，条件处理，以及动态扩展。但是，不但 Joran 非常复杂，而且给用户的体验非常的不好，或者至少不直观。

本章叙述基于 Groovy 的 DSL 致力于一致性，直观性，以及非常强大。任何你可以使用 XML 配置的文件，你都可以用更加简短的符号使用 Groovy 来实现。为了帮助你迁移到 Groovy 风格的配置，我们开发了一个[工具](https://logback.qos.ch/translator/asGroovy.html)。

## 常规建议

一般来说，*logback.groovy* 文件是 Groovy 程序。因为 Groovy 是 Java 的超集，所以无论你在 Java 执行什么配置操作，你都可以在 *logback.groovy* 文件中做同样的事情。但是，在 Java 中，使用变成的方式配置 logback 有点笨重，所以我们增加了一些 logback 特有的扩展来减轻你的负担。我们尝试限制 logback 特有的拓展符号尽量的少。如果你已经熟悉了 Groovy，那么你应该更加容易去读，去理解甚至去写你自己的 *logback.groovy* 文件。那么不熟悉 Groovy 的人依然会发现 *logback.groovy* 中的语法比 *logback.xml* 中的语法更加容易使用。

*logback.groovy* 文件是 Groovy 程序，具有最小的 logback 特定的拓展。所有常用的 groovy 结构，例如类的导入，变量定义，字符串 (GString) 中包含 \${..} 评估表达式，以及 if-else 语句在 *logback.grooby* 文件中都是可用的。

## 自动导入

**`1.0.10 版本以后`** 为了减少不必要的引用，一些共同的类以及包会被自动导入。因此，只要你只是配置了内置的 appender，layout 等等，你不需要在你的脚本中添加相对应的导入语句。当然，对于默认导入不会涉及到类，你需要自己导入。

下面是默认导入的列表：

-   import ch.qos.logback.core.*;
-   import ch.qos.logback.core.encoder.*;
-   import ch.qos.logback.core.read.*;
-   import ch.qos.logback.core.rolling.*;
-   import ch.qos.logback.core.status.*;
-   import ch.qos.logback.classic.net.*;
-   import ch.qos.logback.classic.encoder.PatternLayoutEncoder;

另外，`ch.qos.logback.classic.Level` 中的所有常量 (大写) 都会被静态导入，以及小写的别名。也就是说在你的脚本中可以引用 *INFO* 以及 *info*，而不需要使用静态导入语句。

## 不再支持 SiftingAppender

**`1.0.12 版本以后`** 在 groovy 配置文件中不再支持 `SiftingAppender`。但是，如果有需要，可以重新引进。

## *logback.groovy* 特定的拓展

**本质上，*logback.groovy* 语法包含以下所说的六个方法；按照它们习惯上相反的顺序出现。**严格来说，这些方法的调用顺序并**不**重要，但是有一个例外：appender 附加到 logger 之前**必须**被定义。

- ### root(Level level, List\<String\> appenderNames = [])

`root` 方法可以用来设置 root logger 的日志级别。第二个可选参数的类型为 `List<String>`，可以用来添加之前定义的 appender 的名字。如果你不想指定 appenderNames，那么就是一个空 (empty) 的列表。在 Groovy 中，用 `[]` 表示一个空的列表。

设置 root logger 的级别为 WARN，你可以这样写：

```groovy
root(WARN)
```

设置 root logger 的级别为 INFO，并且将名为 "CONSOLE" 与 "FILE" 的 appender 附加到 root 上，你可以这样写：

```groovy
root(INFO, ["CONSOLE", "FILE"])
```

在前面的例子中，假设名为 "CONSOLE" 与 "FILE" 的 appender 已经被定义好了。很快将会讨论有关 appender 的定义。

- ### logger(String name, Level level, List\<String> appenderNames = [], Boolean additivity = null)

`logger()` 方法接收四个参数，最后两个是可选的。第一个参数表示配置 logger 的名字。第二参数表示指定 logger 的级别。设置 logger 的级别为 `null` 将强制它从它最近的祖先那里[继承](https://github.com/Volong/logback-chinese-manual/blob/master/02%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E6%9E%B6%E6%9E%84.md#%E6%9C%89%E6%95%88%E7%AD%89%E7%BA%A7%E5%8F%88%E7%A7%B0%E4%B8%BA%E7%AD%89%E7%BA%A7%E7%BB%A7%E6%89%BF)级别。第三个参数的类型为 `List<String>`，是可选的，默认为空列表。列表中 appender 会被附加到指定的 logger 上去。第四个参数的类型为 `Boolean`，也是可选的，用来控制[叠加性](https://github.com/Volong/logback-chinese-manual/blob/master/02%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E6%9E%B6%E6%9E%84.md#appender-%E4%B8%8E-layout)。如果忽略，默认值为 `null`。

例如，下面这个脚本设置 "com.foo" 这个 logger 的级别为 INFO：

```groovy
logger("com.foo", INFO)
```

下个脚本设置 "com.foo" 这个 logger 的级别为 DEBUG，并且将名为 "CONSOLE" 的 appender 附加到其上：

```groovy
logger("com.foo", DEBUG, ["CONSOLE"])
```

下个脚本跟上一个类似，只是这个还设置了 "com.foo" 这个 logger 的叠加性为 false：

```groovy
logger("com.foo", DEBUG, ["CONSOLE"]，false)
```

- ### appender(String name, Class clazz, Closure closure = null)

appender 方法的第一个参数接收 appender 的名字进行配置。第二个参数是强制的，表示 appender 实例化的类。第三个参数包含更多的配置信息。如果忽略，默认为 null。

大部分 appender 都需要设置属性，并且注入子组件才能正常工作。属性通过 '=' 进行设置。子组件的注入通过调用以属性命名的方法，并且将实例化的类作为参数传递给该方法。这个约定可以被递归的应用到配置的属性以及任何 appender 子组件的子组件中。这个方法是 *logback.groovy* 的核心，可能是唯一需要去学习的约定。

例如，接下来的脚本实例化一个 `FileAppender` 命名为 "FILE"，设置它的 `file` 属性为 "testFile.log"，以及它的 `append` 属性设置为 false。类型为 `PatternLayoutEncoder` 的 encoder 被注入到这个 appender 中。encoder 的模式属性设置为 "%level %logger - %msg%n"。然后将这个 appender 附加到 root logger 上。

```groovy
appender("FILE", FileAppender) {
    file = "testFile.log"
    append = false
    encoder(PatternLayoutEncoder) {
        pattern = "%level %logger - %msg%n"
    }
}

root(DEBUG, ["FILE"])
```

- ### timestamp(String datePattern, long timeReference = -1)

`timestamp()` 方法根据 `datePattern` 将 `timeReference` 参数格式化，返回一个对应的字符串。`datePattern` 参数应该尊村 [SimpleDateFormat](https://docs.oracle.com/javase/8/docs/api/java/text/SimpleDateFormat.html) 中定义的约定。如果 `timeReference` 没有指定，那么默认为 -1。在这种情况下，当解析配置文件时，当前时间作为 `timeReference` 参数的值。

在下个例子中，`bySecond` 变量表示被 "yyyyMMdd'T'HHmmss" 格式化之后的当前时间。之后，"bySecond" 变量被用于 `file` 属性的定义中。

```groovy
def bySecond = timestamp("yyyyMMdd'T'HHmmss")

appender("FILE", FileAppender) {
    file = "log-${bySecond}.txt"
    encoder(PatternLayoutEncoder) {
        pattern = "%logger{35} - %msg%n"
    }
}

root(DEBUG, ["FILE"])
```

- ### conversionRule(String conversionWord, Class converterClass)

在创建了你自己的[转换说明符](https://github.com/Volong/logback-chinese-manual/blob/master/06%E7%AC%AC%E5%85%AD%E7%AB%A0%EF%BC%9ALayouts.md#%E8%87%AA%E5%AE%9A%E4%B9%89%E8%BD%AC%E6%8D%A2%E8%AF%B4%E6%98%8E%E7%AC%A6)之后，你需要通知 logback 它的存在。下面这个简单的 logback.groovy 文件告诉 logback 在遇到 `%sample` 转换字符时使用 MySampleConverter。

```groovy
import chapters.layouts.MySampleConverter

conversionRule("sample", MySampleConverter)
appender("STDOUT", ConsoleAppender) {
    encoder(PatternLayoutEncoder) {
        pattern = "%-4relative [%thread] %sample - %msg%n"
    }
}

root(DEBUG, ["STDOUT"])
```

- ### scan(String scanPeriod = null)

调用 scan() 方法告诉 logback 周期性的扫描 logback.groovy 文件的变化。当检测到变化时，*logback.groovy* 文件会被重新加载。

```groovy
scan()
```

默认情况下，一分钟扫描一次配置文件。你可以通过 "scanPeriod" 来指定一个不同的扫描周期。它的值可以被指定以 milliseconds, seconds, minutes 或者 hours 位单位。例如：

```groovy
scan("30 seconds")
```

如果没有指定时间单位，那么默认的时间单位为 milliseconds，但是通常来说是不合适的 (既然不合适，为什么默认还是毫秒，费解🤔)。如果你更改了默认的扫描周期，记得要指定时间单位。更多关于扫描工作的细节，请查看[自动加载](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%BD%93%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E6%9B%B4%E6%94%B9%E6%97%B6%E8%87%AA%E5%8A%A8%E5%8A%A0%E8%BD%BD)部分。

-   ### statusListener(Class listenerClass)

你可以通过调用 `statusListener` 方法，并给该方法传递一个监听器类，来添加一个状态监听器。例：

```groovy
import chapters.layouts.MySampleConverter

// 强烈建议在最后一个导入语句之后，其它所有语句之前添加状态监听器
statusListener(OnConsoleStatusListener)
```

关于[状态监听器](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E7%9B%91%E5%90%AC%E7%8A%B6%E6%80%81%E4%BF%A1%E6%81%AF)请查看之前的章节。

-   ### jmxConfigurator(String name)

你可以通过该方法注册一个 [`JMXConfigurator`](https://logback.qos.ch/manual/jmxConfig.html) MBean。无参调用将会使用 logback 默认的对象名 (`ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator`) 去注册 MBean。

```groovy
jmxConfigurator()
```

要改变 `Name` 键的值，而不是 "default"，仅仅只需要给 `jmxConfigurator` 方法传递一个不同的名字参数就可以了。

```groovy
jmxConfigurator('MyName')
```

如果你想要完整的定义对象名，可以使用同样的语法，但是需要传递一个有效的对象名字符串作为参数：

```groovy
jmxConfigurator('myApp:type=LoggerManager')
```

该方法首先会去尝试将该参数作为对象名，如果它不表示一个有效的对象名，则会把它当作 "Name" 键的值。

## 内置 DSL

*logback.groovy* 是一个内置 DSL 的意思是，它的内容可以作为 Groovy 脚本执行。因此，所有常用的 Groovy 指令，例如类的导入，GString，变量的定义，包含字符串 (GString) 的 \${..} 评估表达式，if-else 语句这些在 logback.groovy 文件中都是可用的。在接下来的讨论中，我们将会展示 Groovy 指令在 *logback.groovy* 文件中的典型用法。

### 变量定义与 GString

你可以在 *logback.groovy* 文件中的任何地方定义变量，然后在 GString 中使用该变量。例如：

```groovy
def USER_HOME = System.getProperty("user.home")

appender("FILE", FileAppender) {
    // 使用 USER_HOME 变量
    file = "${USER_HOME}/myApp.log"
    encoder(PatternLayoutEncoder) {
        pattern = "%msg%n"
    }
}
root(DEBUG, ["FILE"])
```

### 在控制台打印

通过调用 Groovy 的 `println` 方法在控制台进行打印。例如：

```groovy
def USER_HOME = System.getProperty("user.home");
println "USER_HOME=${USER_HOME}"

appender("FILE", FileAppender) {
    println "Setting [file] property to [${USER_HOME}/myApp.log]"
    file = "${USER_HOME}/myApp.log"
    encoder(PatternLayoutEncoder) {
        pattern = "%msg%n"
    }
}
root(DEBUG, ["FILE"])
```

### 自动输出字段

#### 'hostname' 变量

'hostname' 变量包含当前 host 的名字。但是由于作用域规则，所以作者不能完全解释清楚 (😓)。'hostname' 变量只在最上层的作用域中有效，但是在内部的作用域中无效。下面的例子应该可以解释这一点：

```groovy
// 如果当前 host 的名字为 x，那么将会输出 "hostname is x"
println "hostname is ${hostname}"

appender("STDOUT", ConsoleAppender) {
    // 将会输出 "hostname is null"
    println "hostname is ${hostname}"
}
```

如果你想要在所有的作用域中使用 hostname 变量。那么你需要定义一个变量，并将 'hostname' 的值赋给它。如下：

```groovy
// 将 hostname 的值赋给 HOSTNAME
def HOSTNAME = hostname

// 如果当前 host 的名字为 x，那么将会输出 "hostname is x"
println "hostname is ${HOSTNAME}"

appender("STDOUT", ConsoleAppender) {
    // 如果当前 host 的名字为 x，那么将会输出 "hostname is x"
    println "hostname is ${HOSTNAME}"
}
```

### 任何对于当前上下文的引用都是上下文感知的

*logback.groovy* 脚本是在 [ContextAware](https://logback.qos.ch/xref/ch/qos/logback/core/spi/ContextAware.html) 对象的范围内执行完成的。因此，在当前上下文的范围内，你可以使用 '`context`'，并且可以通过 `addInfo()`、`addWarn()`、与 `addError()` 方法将状态信息发送给上下文的 `StatusManager`。

```groovy
// 添加一个控制台转态监听器总是没错的
statusListener(OnConsoleStatusListener)

// 设置上下文的名字为 wombat
context.name = 'wombat'

// 添加一个关于上下文名字的状态信息
addInfo("Context name has been set to ${context_name}")

def USER_HOME = System.getProperty("user.home")

// 添加关于 USRE_HOME 的状态信息
addInfo("USER_HOME=${USER_HOME}")

appender("FILE", FileAppender) {
    addInfo("Setting [file] property to [${USER_NAME}/myApp.log]")
    file = "${USER_HOME}/myApp.log"
    encoder(PatternLayoutEncoder) {
        pattern = "%msg%n"
    }
}
root(DEBUG, ["FILE"])
```

### 条件配置

由于 Groovy 是一种完全成熟的编程语言，条件语句允许单一的 *logback.groovy* 文件用来适用不同的环境，例如开发，测试以及生产。

在下个脚本中，console appender 根据 host 来激活，而不是我们的生产环境 pixie 或 orion。rolling file appender 的输出目录也是根据 host 来确定。

```groovy
statusListener(OnConsoleStatusListener)

def appenderList = ["ROLLING"]
def WEBAPP_DIR = "."
def consoleAppender = true;

// hostname 是否匹配 pixie 或 orion
if (hostname =~ /pixie|orion/) {
    WEBAPP_DIR = "/opt/myapp"
    consoleAppender = false
} else {
    appenderList.add("CONSOLE")
}

if (consoleAppender) {
    appender("CONSOLE", ConsoleAppender) {
        encoder(PatternLayoutEncoder) {
            pattern = "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
        }
    }
}

appender("ROLLING", RollingFileAppender) {
    encoder(PatternLayoutEncoder) {
        Pattern = "%d %level %thread %mdc %logger - %m%n"
    }
    rollingPolicy(TimeBasedRollingPolicy) {
        FileNamePattern = "${WEBAPP_DIR}/log/translator-%d{yyyy-MM}.zip"
    }
}

root(INFO, appenderList)
```

