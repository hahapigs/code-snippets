## 读取文件内容

 

项目结构

└── resources
    ├── META-INF
    ├── application.yml
    ├── banner.txt
    ├── i18n
    │   ├── messages.properties
    │   ├── messages_en_US.properties
    │   └── messages_zh_CN.properties
    ├── key
    │   └── xxx-key.pub
    ├── logback-spring.xml
    └── mapper
        └── Test-mapper.xml

以读取 resource 中的 xxx-key.pub 中的公钥为需求，或者读取磁盘绝对路径中的公钥

方式一：

```java
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.BufferedReader;

​```
InputStream is = Test.class.getClassLoader().getResourceAsStream("key/xxx.pub");
InputStreamReader reader = new InputStreamReader(is);
BufferedReader bf = new BufferedReader(reader);
```

方式二：

``` java
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.BufferedReader;

​```
InputStream is = Test.class.getResource("/key/xxx.pub").openStream();
InputStreamReader reader = new InputStreamReader(is);
BufferedReader bf = new BufferedReader(reader);
```

方式三：

```
import java.io.InputStreamReader;
import java.io.BufferedReader;
import java.io.FileReader;

​```
BufferedReader bf = new BufferedReader(new FileReader("/Users/canary/code-snippets-app/src/main/resources/key/xxx.pub"));
```

方式四：

``` java
import java.io.File;
import java.nio.file.Files;
import java.io.InputStream;
import java.io.FileInputStream;
import java.io.InputStreamReader;
import java.io.BufferedReader;

​```
String url = Test.class.getResource("/key/xxx.pub").getFile();
File file = new File(url);
InputStream is = Files.newInputStream(file.toPath());
InputStreamReader reader = new InputStreamReader(is);
BufferedReader bf = new BufferedReader(reader);
```

方式五：

```java
import org.springframework.core.io.InputStreamResource;
import java.io.File;
import java.io.InputStream;
import java.io.FileInputStream;
import java.io.InputStreamReader;
import java.io.BufferedReader;

​```
String url = Test.class.getResource("/key/xxx.pub").getFile();
File file = new File(url);
FileInputStream fis = new FileInputStream(file);
InputStream is = new InputStreamResource(fis).getInputStream();
InputStreamReader reader = new InputStreamReader(is);
BufferedReader bf = new BufferedReader(reader);
```

方式六：

``` java
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import java.io.InputStreamReader;
import java.io.BufferedReader;
 
​```
Resource resource = new ClassPathResource("key/dwt-key.pub");
InputStream is = resource.getInputStream();
InputStreamReader reader = new InputStreamReader(is);
BufferedReader bf = new BufferedReader(reader);
```



完整实现：

```java
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.BufferedReader;

​```
public class Test {  
  
		private static final String filePath = "key/dwt-key.pub";
  
    /**
     * 从文件中获取公钥匙字符串
     *
     * @param filePath 路径
     * @return 返回公钥字符串
     */
    public static String loadPublicKeyByFile(String filePath) {
        try {
            InputStream is = Test.class.getClassLoader().getResourceAsStream(filePath);
            InputStreamReader reader = new InputStreamReader(is);
            BufferedReader bf = new BufferedReader(reader);
            StringBuilder sb = new StringBuilder();
            String readLine = null;
            while ((readLine = bf.readLine()) != null) {
                if (readLine.charAt(0) == '-') {
                    continue;
                } else {
                    sb.append(readLine);
                    // sb.append('\r');
                }
            }
            bf.lines();
            return sb.toString();
        } catch (IOException e) {
            throw new RuntimeException("load public key by path is error!");
        }
    }
}  
 		
```

