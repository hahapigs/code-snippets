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

