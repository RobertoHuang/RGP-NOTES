# 前言

`Logback`是现在比较流行的一个日志记录框架，它的配置比较简单学习成本相对较低，所以刚刚接触该框架的朋友不要畏惧，多花点耐心很快就能灵活应用了。本篇博文不会具体介绍`Logback`搭建过程，如果你是`Logback`初学者强烈建议阅读[Logback常用配置详解](http://aub.iteye.com/blog/1110008)，它对`Logback`的配置介绍的非常的详细，相信你看完这篇博客后会对`Logback`有一定的了解，然后再回头看下面的内容收获会更大

## YAML配置

```yaml
# 日志配置
logging:
  config: classpath:logback.xml
```

## 企业级应用常用Logback配置

```
<?xml version="1.0" encoding="UTF-8" ?>
<!--
    scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true
    scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位默认单位是毫秒，当scan为true时此属性生效，默认时间间隔为1分钟
    debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态，默认值为false
 -->
<configuration scan="true" scanPeriod="3 seconds" debug="false">
    <!-- 这一句的意思是打印所有进入的信息 -->
    <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />

    <!--
        appender是<configuration>的子节点，是负责写日志的组件
        两个必要属性name和class:name指定appender名称，class指定appender的全限定名
        定义控制台appender 作用:把日志输出到控制台 class="ch.qos.logback.core.ConsoleAppender"
    -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 对日志进行格式化 -->
        <encoder>
            <pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m"}\n
            </pattern>
        </encoder>
    </appender>

    <!--
        定义滚动记录文件appender 作用:滚动记录文件,先将日志记录到指定文件,当符合某个条件时,将日志记录到其他文件
        RollingFileAppender class="ch.qos.logback.core.rolling.RollingFileAppender"
        参数：
            <append>:如果是true日志被追加到文件结尾，如果是false清空现存文件,默认是true
            <file>:被写入的文件名，可以是相对目录也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值
            <rollingPolicy>:当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名
            <triggeringPolicy>:告知RollingFileAppender合适激活滚动
            <prudent>:当为true时不支持FixedWindowRollingPolicy支持TimeBasedRollingPolicy，但是有两个限制:1不支持也不允许文件压缩,2不能设置file属性必须留空
    -->
    <appender name="LOGBACK_ALL_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <append>true</append>
        <file>logs/logback-test-all.log</file>

        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天滚动一次的日志 -->
            <FileNamePattern>logs/logback-test.%d{yyyy-MM-dd}.log.zip</FileNamePattern>
            <!-- 每分钟滚动一次日志 -->
            <!-- <FileNamePattern>logs/logback-test.%d{yyyy-MM-dd_HH-mm}.log.zip</FileNamePattern> -->
            <!--  只保留30天内的日志文件 -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>

        <!-- 配置临界值过滤器 作用:过滤掉低于指定临界值的日志，当日志级别等于或高于临界值时过滤器返回NEUTRAL，当日志级别低于临界值时，日志会被拒绝 此处配置为INFO 即过滤掉日志级别小于INFO的日志信息 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>

        <encoder>
            <pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m%n", "caller":"%caller{1}"}
            </pattern>
        </encoder>
    </appender>

    <!--
        定义滚动记录文件appender 作用:滚动记录文件,先将日志记录到指定文件,当符合某个条件时,将日志记录到其他文件
        RollingFileAppender class="ch.qos.logback.core.rolling.RollingFileAppender"
        参数：
            <append>:如果是true日志被追加到文件结尾，如果是false清空现存文件,默认是true
            <file>:被写入的文件名，可以是相对目录也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值
            <rollingPolicy>:当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名
            <triggeringPolicy>:告知RollingFileAppender合适激活滚动
            <prudent>:当为true时不支持FixedWindowRollingPolicy支持TimeBasedRollingPolicy，但是有两个限制:1不支持也不允许文件压缩,2不能设置file属性必须留空
     -->
    <appender name="LOGBACK_ERROR_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <append>true</append>
        <file>logs/logback-test-error.log</file>
        <!--
            定义滚动策略 作用:根据固定窗口算法重命名文件的滚动策略 class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy
            <fileNamePattern>:表示当触发了回滚策略后，按这个文件命名规则生成归档文件，命名规则中的%i表示在maxIndex和minIndex之间的一个整数值
            <minIndex>:最小索引值
            <maxIndex>:最大索引值
         -->
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>logs/logback-test-error.log.%i</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>20</maxIndex>
        </rollingPolicy>

        <!-- 定义按文件大小触发滚动策略triggeringPolicy 作用:查看当前活动文件的大小，如果超过指定大小会告知RollingFileAppender触发当前活动文件滚动 只有一个参数 maxSize 这是活动文件的大小，默认值是10MB -->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>50MB</maxFileSize>
        </triggeringPolicy>

        <!--
            配置日志级别过滤器 作用:根据日志级别进行过滤，如果日志级别等于配置级别过滤器会根据onMath和onMismatch接收或拒绝日志
            参数:
                <level>:设置过滤级别
                <onMatch>:用于配置符合过滤条件的操作
                <onMismatch>:用于配置不符合过滤条件的操作
                此处配置为只接收ERROR日志级别信息
        -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>

        <encoder>
            <pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m%n", "caller":"%caller{1}"}
            </pattern>
        </encoder>
    </appender>

    <!--
        logger用来设置某一个包的日志打印级别,以及指定<appender>
        <loger> 仅有一个name属性,一个可选的level和一个可选的addtivity属性
                name:用来指定受此loger约束的某一个包或者具体的某一个类
                level:用来设置打印级别,大小写无关:TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF
                addtivity:是否向上级loger传递打印信息。默认是true,会将信息输入到root配置指定的地方,可以包含多个appender-ref，标识这个appender会添加到这个logger
    -->
    <logger name="com.roberto" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="LOGBACK_ALL_LOG" />
        <appender-ref ref="LOGBACK_ERROR_LOG" />
    </logger>

    <!-- 将root的打印级别设置为"INFO",指定了名字为"FILE","STDOUT"的appender -->
    <root>
        <level value="INFO" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="LOGBACK_ALL_LOG" />
        <appender-ref ref="LOGBACK_ERROR_LOG" />
    </root>
</configuration>
```

以上配置部分涵盖了大部分日志记录的需求，如控制台输出日志、文件记录日志、日志输出格式化、按文件大小滚动日志、按日期滚动日志、日志级别过滤等等，在实际应用中只需对配置进行适当的修改即可。以上配置注释部分可以帮助你解决百分90的疑惑，接下来我以上诉配置为基准，运行程序进行简单解释说明

## 企业级应用常用Logback配置讲解

- 在`com.roberto`包下测试

  ```java
  package com.roberto;
  
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  
  public class LogbackTest {
      private static Logger LOGGER = LoggerFactory.getLogger(LogbackTest.class);
  
      public static void main(String[] args) {
          LOGGER.debug("=======DEBUG=======");
          LOGGER.info("========INFO=======");
          LOGGER.warn("========WARN=======");
          LOGGER.error("=======ERROR=======");
      }
  }
  ```

  以上诉配置为基准新建`Logback`测试类 观察日志输出情况 切记包名为`com.roberto`

  - 控制台输出

    ```reStructuredText
    {"timestamp":"2018-08-17T10:42:11.089","thread":"main","line":"28","log_level":"DEBUG","class_name":"com.roberto.LogbackTest;","msg":"=======DEBUG======="}
    {"timestamp":"2018-08-17T10:42:11.094","thread":"main","line":"29","log_level":"INFO","class_name":"com.roberto.LogbackTest;","msg":"========INFO======="}
    {"timestamp":"2018-08-17T10:42:11.095","thread":"main","line":"30","log_level":"WARN","class_name":"com.roberto.LogbackTest;","msg":"========WARN======="}
    {"timestamp":"2018-08-17T10:42:11.095","thread":"main","line":"31","log_level":"ERROR","class_name":"com.roberto.LogbackTest;","msg":"=======ERROR======="}
    ```

    ```reStructuredText
    <logger name="com.roberto" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="LOGBACK_ALL_LOG" />
        <appender-ref ref="LOGBACK_ERROR_LOG" />
    </logger>
    ```

    控制台打印以上内容是因为这里配置了包名为`com.roberto`的`DEBUG`信息输出到控制台

  - `logback-test-all.log`内容

    ```reStructuredText
    {"timestamp":"2018-08-17T10:42:11.094","thread":"main","line":"29","log_level":"INFO","class_name":"com.roberto.LogbackTest;","msg":"========INFO=======", "caller":"Caller+0     at com.roberto.LogbackTest.main(LogbackTest.java:29)"}
    {"timestamp":"2018-08-17T10:42:11.095","thread":"main","line":"30","log_level":"WARN","class_name":"com.roberto.LogbackTest;","msg":"========WARN=======", "caller":"Caller+0     at com.roberto.LogbackTest.main(LogbackTest.java:30)"}
    {"timestamp":"2018-08-17T10:42:11.095","thread":"main","line":"31","log_level":"ERROR","class_name":"com.roberto.LogbackTest;","msg":"=======ERROR=======", "caller":"Caller+0     at com.roberto.LogbackTest.main(LogbackTest.java:31)"}
    ```

    ```reStructuredText
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>INFO</level>
    </filter>
    ```

    虽然配置`com.roberto`的`DEBUG`级别信息输出到`logback-test-all.log`，但是由于`LOGBACK_ALL_LOG`的`appender`中配置了过滤器，过滤掉了`INFO`级别以下的信息，所以`logback-test-all.log`信息如上

  - `logback-test-error.log`内容

    ```reStructuredText
    {"timestamp":"2018-08-17T10:42:11.095","thread":"main","line":"31","log_level":"ERROR","class_name":"com.roberto.LogbackTest;","msg":"=======ERROR=======", "caller":"Caller+0     at com.roberto.LogbackTest.main(LogbackTest.java:31)"}
    ```

    ```reStructuredText
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>ERROR</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
    </filter>
    ```

    虽然配置`com.roberto`的`DEBUG`级别信息输出到`logback-test-error.log`，但是由于`LOGBACK_ERROR_LOG`的`appender`中配置了过滤器，只输出`ERROR`级别信息，所以`logback-test-error.log`打印信息如上

- 在非`com.roberto`包下测试

  ```java
  package com.dreamt;
  
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  
  public class LogbackTest {
      private static Logger LOGGER = LoggerFactory.getLogger(LogbackTest.class);
  
      public static void main(String[] args) {
          LOGGER.debug("=======DEBUG=======");
          LOGGER.info("========INFO=======");
          LOGGER.warn("========WARN=======");
          LOGGER.error("=======ERROR=======");
      }
  }
  ```

  注意此处包名不为`com.roberto`，观察日志输出情况

  - 控制台输出

    ```reStructuredText
    {"timestamp":"2018-08-17T10:53:22.669","thread":"main","line":"29","log_level":"INFO","class_name":"com.dreamt.LogbackTest;","msg":"========INFO======="}
    {"timestamp":"2018-08-17T10:53:22.678","thread":"main","line":"30","log_level":"WARN","class_name":"com.dreamt.LogbackTest;","msg":"========WARN======="}
    {"timestamp":"2018-08-17T10:53:22.679","thread":"main","line":"31","log_level":"ERROR","class_name":"com.dreamt.LogbackTest;","msg":"=======ERROR======="}
    ```

  - `logback-test-all.log`内容

    ```reStructuredText
    {"timestamp":"2018-08-17T10:53:22.669","thread":"main","line":"29","log_level":"INFO","class_name":"com.dreamt.LogbackTest;","msg":"========INFO=======", "caller":"Caller+0     at com.dreamt.LogbackTest.main(LogbackTest.java:29)"}
    {"timestamp":"2018-08-17T10:53:22.678","thread":"main","line":"30","log_level":"WARN","class_name":"com.dreamt.LogbackTest;","msg":"========WARN=======", "caller":"Caller+0     at com.dreamt.LogbackTest.main(LogbackTest.java:30)"}
    {"timestamp":"2018-08-17T10:53:22.679","thread":"main","line":"31","log_level":"ERROR","class_name":"com.dreamt.LogbackTest;","msg":"=======ERROR=======", "caller":"Caller+0     at com.dreamt.LogbackTest.main(LogbackTest.java:31)"}
    ```

  - `logback-test-error.log`内容

    ```reStructuredText
    {"timestamp":"2018-08-17T10:53:22.679","thread":"main","line":"31","log_level":"ERROR","class_name":"com.dreamt.LogbackTest;","msg":"=======ERROR=======", "caller":"Caller+0     at com.dreamt.LogbackTest.main(LogbackTest.java:31)"}
    ```

  - 以上输出是由于配置文件如下

    ```reStructuredText
    <root>
        <level value="INFO" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="LOGBACK_ALL_LOG" />
        <appender-ref ref="LOGBACK_ERROR_LOG" />
    </root>
    ```

关于滚动日志这里就不做测试，你们可以自己写个循环输出日志进行测试，上面配置注释的非常清楚，只要你们肯动手肯定能实验成功的，以上部分纯属个人拙见，如有误解欢迎大家指正，谢谢~
