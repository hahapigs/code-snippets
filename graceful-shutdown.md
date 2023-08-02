## 优雅停服

#### 方式一：通过 Actuator 的 Endpoint 机制关闭服务

pom.xml

``` xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

application.yaml

``` yaml
management:
  endpoints:
    web:
      exposure:
        # 开启 shutdown 端点访问
        include: shutdown
  endpoint:
    shutdown:
      # 开启 shutdown 实现优雅停服
      enabled: true
```

命令

``` powershell
# httpie
$ http post http://localhost:8080/actuator/shutdown
# curl
$ curl -X post http://localhost:8080/actuator/shutdown
```

 #### 方式二：监听服务 pid，通过kill方式关闭服务（拓展配置）

resources/META-INFO/spring.factories

``` tex
org.springframework.context.ApplicationListener=\
org.springframework.boot.context.ApplicationPidFileWriter,\
org.springframework.boot.web.context.WebServerPortFileWriter
```

application.yaml

``` yaml
spring:
	pid:
		# pid 文件生成目录及名称，如果不配置，默认为项目根目录，名称为 application.pid
		file: ./canary.pid
```

命令

```powershell
$ cat ./canary.pid | xargs kill
```

#### 方式三：监听服务 pid，通过 kill 方式关闭服务（代码配置）

CanaryApplication.java

``` java
@SpringBootApplication
public class CanaryApplication {
	public static void main(String[] args) {
		SpringApplication application = new SpringApplication(CanaryApplication.class);
    // 如果 ApplicationPidFileWriter 不指定文件路径和名称（"./canary.pid"），默认为项目根目录，名称为 application.pid
    // 也可以通过 application.yaml 或 application.properties 指定
		application.addListeners(new ApplicationPidFileWriter("./canary.pid"));
		application.run(args);
	}
}
```

application.yaml

``` yaml
spring:
	pid:
		# pid 文件生成目录及名称，如果不配置，默认为项目根目录，名称为 application.pid
		file: ./canary.pid
```

命令

```powershell
$ cat ./canary.pid | xargs kill
```

#### 方式四：通过 ApplicationContext.close() 关闭服务

创建 SpringContext.java 实现 ApplicationContextAware.java

``` java
import jakarta.annotation.Nullable;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

/**
 * SpringBean 上下文
 *
 * @since 1.0
 * @author zhaohongliang
 */
@Component
public class SpringContext implements ApplicationContextAware {

    /**
     * application context
     */
    private static ApplicationContext context;

    /**
     * 设置 application context
     *
     * @param context application上下文
     */
    private static void setContext(ApplicationContext context) {
        SpringContext.context = context;
    }

    @Override
    public void setApplicationContext(@Nullable ApplicationContext applicationContext) throws BeansException {
        SpringContext.setContext(applicationContext);
    }


    public static <T> T getBean(Class<T> clazz) {
        return context.getBean(clazz);
    }

    public static <T> T getBean(String beanName, Class<T> clazz) {
        return context.getBean(beanName, clazz);
    }

    public static Object getBean(String beanName) {
        return context.getBean(beanName);
    }
}

```

CanaryApplication.java

``` java
@SpringBootApplication
public class CanaryApplication {
	public static void main(String[] args) {
		SpringApplication.run(CanaryApplication.class, args);
	}
  
  /**
	 * 停止服务
	 */
  @PostMapping("/shutdown")
  public void shutdown() {
    ((ConfigurableApplicationContex) SpringContext.getConext()).close();
  }
}
```

