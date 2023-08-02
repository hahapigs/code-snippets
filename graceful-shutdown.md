## 优雅停服

#### 方式一：

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

 #### 方式二：

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

#### 方式三：

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

