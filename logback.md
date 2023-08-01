## Logback

#### 环境配置

方式一：

``` tex
logback-spring-local.xml
logback-spring-dev.xml
logback-spring-test.xml
logback-spring-prod.xml
......
```

方式二：

在 logback-spring.xml 中使用  springProfile，同一个 springProfile 标签下支持多个环境，用｜隔开

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="10 seconds" debug="false">
  
		...... 
  
		<!-- 本地环境:打印控制台 -->
    <springProfile name="local">
        <!-- 可以输出项目中的debug日志，包括mybatis的sql日志 -->
        <!-- properties或yaml中如果配置 logging.level.com.example.canary = info 会覆盖此级别 -->
        <logger name="com.example.canary" level="DEBUG" additivity="false">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ALL_FILE" />
        </logger>
    </springProfile>
  
  	<!-- 开发和测试环境:打印控制台 -->
    <springProfile name="dev | test">
        <logger name="com.example.canary" level="INFO" additivity="false">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ALL_FILE" />
        </logger>
    </springProfile>
  	
  	<!-- 生产环境:输出到文件 -->
    <springProfile name="pro">
        <logger name="com.example.canary" level="INFO" additivity="false">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ALL_FILE" />
        </logger>
    </springProfile>
  
    <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，默认是DEBUG
        可以包含零个或多个appender元素。
    -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="WARN_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>
</configuration>
```

如果 application.properties 或 application.yaml 中如果配置 logging.level.com.example.canary = info , 它会自动覆盖 <logger>，同时可以省略 <springProfile>, 因为不同的环境的yaml，会自动装配。

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="10 seconds" debug="false">
  
		...... 
  
		<!-- properties或yaml中如果配置 logging.level.com.example.canary = info 会覆盖此级别 -->
    <logger name="com.example.canary" level="DEBUG" additivity="false">
    		<appender-ref ref="CONSOLE" />
    		<appender-ref ref="ALL_FILE" />
  	</logger>
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="WARN_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>
</configuration>
```



#### 上下文名称

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="10 seconds" debug="false">
  	<!-- 上下文名称, 可以使 %contextName 使用上下文名称-->
    <contextName>canary</contextName>
</configuration>
```

#### 变量

``` xml
<configuration scan="true" scanPeriod="10 seconds" debug="false">
		<!-- 变量, 可以使 ${SERVER_NAME} 使用上下文变量, scope 默认为 local
         scope: local, 本地范围内的属性存在配置文件的加载过程中。配置文件每加载一次，变量就会被重新定义一次
                context, 上下文范围内的属性会一直存在上下文被清除
                system, 系统范围内的属性，会插入到 JVM 的系统属性中，跟随 JVM 一同消亡
    -->
    <springProperty scope="context" name="SERVER_NAME" source="spring.application.name" defaultValue="canary"/>
  	<springProperty scope="context" name="LOG_PATH" source="logging.file.path" defaultValue="/data/canary/log"/>
    <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义变量后，可以使“${LOG_PATH}”来使用变量。-->
    <property scope="local" name="LOG_PATH" value="/data/canary/log" />
    <!-- LOG_PATH 可以在 properties 或 yaml 中 使用 logging.file.path = /data/canary/log 进行自动装配，另外还可以通过 java -DLOG_PATH="/data/canary/log" 方式指定日志存放目录 -->
</configuration>  
```

#### 完整示例

application-local.yaml

``` yaml
logging:
  config: classpath:logback-spring.xml
  file:
    path: /data/canary/log
  level:
    com.example.canary: debug
```



logback-spring.xml

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。 -->
<configuration scan="true" scanPeriod="10 seconds" debug="false">

    <!-- 禁用启动时候状态信息的打印, 还可以通过设置 java -Dlogback.statusListenerClass=ch.qos.logback.core.status.NopStatusListener 的值来禁用 -->
    <statusListener class="ch.qos.logback.core.status.NopStatusListener" />

    <!-- <include resource="org/springframework/boot/logging/logback/base.xml" /> -->

    <!-- 上下文名称, 可以使 %contextName 使用上下文名称-->
    <contextName>canary</contextName>
    <!-- 变量, 可以使 ${SERVER_NAME} 使用上下文变量, scope 默认为 local
         scope: local, 本地范围内的属性存在配置文件的加载过程中。配置文件每加载一次，变量就会被重新定义一次
                context, 上下文范围内的属性会一直存在上下文被清除
                system, 系统范围内的属性，会插入到 JVM 的系统属性中，跟随 JVM 一同消亡
    -->
    <springProperty scope="context" name="SERVER_NAME" source="spring.application.name" defaultValue="canary"/>
    <!-- <springProperty scope="context" name="LOG_PATH" source="logging.file.path" defaultValue="/data/canary/log"/> -->
    <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义变量后，可以使“${LOG_PATH}”来使用变量。-->
    <!-- <property scope="local" name="LOG_PATH" value="/data/canary/log" />-->
    <!-- 另外，还可以通过 java -DLOG_PATH="/data/canary/log" 方式指定日志存放目录 -->

    <!-- 彩色日志 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr"
                    converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>

    <!-- 配置格式变量：CONSOLE_LOG_PATTERN 彩色日志格式 -->
    <!-- 日志输出格式：
        %d表示日期时间，
        %thread表示线程名，
        %-5level：级别从左显示5个字符宽度
        %logger{50} 表示logger名字最长50个字符，否则按照句点分割。
        %msg：日志消息，
        %n是换行符
    -->
    <property name="CONSOLE_LOG_PATTERN"
              value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}" />

    <!-- 输出到控制台 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息 -->
        <!-- 例如：如果此处配置了INFO级别，则后面其他位置即使配置了DEBUG级别的日志，也不会被输出 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>DEBUG</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>


    <!-- 输出到文件 -->

    <!-- 时间滚动输出 level 为 >= DEBUG 日志 -->
    <appender name="ALL_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${LOG_PATH}/${SERVER_NAME}.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd}/${SERVER_NAME}.%i.log</fileNamePattern>
            <!-- <fileNamePattern>${LOG_PATH}/info/${SERVER_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern> -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!-- 日志文件保留天数 -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <!-- 输出的日志级别是大于或等于DEBUG级别的日志信息 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>DEBUG</level>
        </filter>
    </appender>

    <!-- 时间滚动输出 level 为 WARN 日志 -->
    <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${LOG_PATH}/${SERVER_NAME}_warn.log</file>
        <!-- 日志文件输出格式 -->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd}/${SERVER_NAME}_warn.%i.log</fileNamePattern>
            <!-- <fileNamePattern>${LOG_PATH}/warn/${SERVER_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern> -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!-- 日志文件保留天数 -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录warn级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>WARN</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>


    <!-- 时间滚动输出 level 为 ERROR 日志 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${LOG_PATH}/${SERVER_NAME}_error.log</file>
        <!-- 日志文件输出格式 -->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd}/${SERVER_NAME}_error.%i.log</fileNamePattern>
            <!-- <fileNamePattern>${LOG_PATH}/error/${SERVER_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern> -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!-- 日志文件保留天数 -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!--
        <logger>用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。
        <logger>仅有一个name属性，
        一个可选的level和一个可选的addtivity属性。
        name:用来指定受此logger约束的某一个包或者具体的某一个类。
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
              如果未设置此属性，那么当前logger将会继承上级的级别。
    -->
    <!--
        使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看sql语句的话，有以下两种操作：
        第一种把<root level="INFO">改成<root level="DEBUG">这样就会打印sql，不过这样日志那边会出现很多其他消息
        第二种就是单独给mapper下目录配置DEBUG模式，代码如下，这样配置sql语句会打印，其他还是正常DEBUG级别：
     -->

    <!-- 本地环境:打印控制台 -->
    <springProfile name="local">
        <!-- 可以输出项目中的debug日志，包括mybatis的sql日志 -->
        <!-- properties或yaml中如果配置 logging.level.com.example.canary = info 会覆盖此级别 -->
        <logger name="com.example.canary" level="DEBUG" additivity="false">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ALL_FILE" />
        </logger>
    </springProfile>


    <!-- 开发和测试环境:打印控制台 -->
    <springProfile name="dev | test">
        <logger name="com.example.canary" level="INFO" additivity="false">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ALL_FILE" />
        </logger>
    </springProfile>


    <!-- 生产环境:输出到文件 -->
    <springProfile name="pro">
        <logger name="com.example.canary" level="INFO" additivity="false">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ALL_FILE" />
        </logger>
    </springProfile>

    <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，默认是DEBUG
        可以包含零个或多个appender元素。
    -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="WARN_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>

</configuration>
```

