# 判断 webapp 启动成功

## 背景

`web` 容器是 `jetty`，`mvc` 框架是 `springmvc`

`webapp` 启动比较耗时：连接数据库，加载数据到本地缓存；连接 `zookeeper` 获取配置，根据配置连接到各中间件，那么问题来了，如何知道 `webapp` 已经启动成功呢？

## 三种方案

### 实现 ApplicationListener&lt;ContextRefreshedEvent&gt;

这个算是网上介绍最多的方法了，若 `onApplicationEvent` 方法被调用，就意味着 `webapp` 启动成功，代码如下

```java
@Component
public class Welcome implements ApplicationListener<ContextRefreshedEvent> {

    private static final Logger log = LoggerFactory.getLogger(Welcome.class);
    
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() != null) {
            log.info("webapp started");
        }
    }
}
```

### 使用 Listener

一般使用到 `springmvc` 的 `webapp`，都会配置 `ContextLoaderListener`，这个监听器会加载配置文件，完成 `spring` 容器初始化的工作，可以在它后面配置一个监听器，这个监听器的监听事件被调用，就意味着 `webapp` 启动成功

`web.xml` 配置如下

```markup
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
	<listener>
		<listener-class>LastListener</listener-class>
	</listener>
```

代码如下

```java
    @Override
    public void contextInitialized(ServletContextEvent arg0) {
        log.info("webapp started");
    }
```

### 继承 DispatcherServlet

基于 `springmvc` 的 `webapp` 都会配置 `DispatcherServlet`，那么我们可以继承它，并重载其 `init` 方法，这个方法执行完成，就意味着 `webapp` 启动成功，代码如下

```java
    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        log.info("webapp started");
    }

```

## 区别

上述三种方案实际上是有区别的，真正的 `webapp` 启动成功应该是第三种方案 ，即继承 `DispatcherServlet`，要说清楚这个问题，我们可以从一个典型的 `web.xml` 文件开始

```markup
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://java.sun.com/xml/ns/javaee" 
	xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	xsi:schemaLocation="
	  http://java.sun.com/xml/ns/javaee 
	  http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	id="WebApp_ID" version="2.5">
	
	<display-name>mywebapp</display-name>

	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath:myconfig.xml</param-value>
	</context-param>


  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
	<listener>
		<listener-class>LastListener</listener-class>
	</listener>


	<servlet>
		<servlet-name>mywebapp</servlet-name>
		<servlet-class>MyDispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value></param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>mywebapp</servlet-name>
		<url-pattern>/*</url-pattern>
	</servlet-mapping>

</web-app>

```

为了说明问题，我们这个 `web.xml` 是极简的，它的执行顺序如下

1. 加载监听器，这个加载顺序是按 `web.xml` 里出现的顺序，并且是同步的
   1. `ContextLoaderListener`
   2. `LastListener`
2. 加载 `MyDispatcherServlet`

当 `ContextLoaderListener` 加载完成，触发第一次的 `onApplicationEvent`，这时候所有 `spring` 管理的 `bean` 都已经是就绪状态了，这意味着数据库已经连接上了，数据也已经加载到缓存；各个中间件诸如 `redis/mq/dfs` 等也是可以访问的状态，但是这个时候 `webapp` 还没有真正完成启动，因为这个时候 `DispatcherServlet` 还没有加载

接下来加载 `LastListener`，如果要求不那么严格的话，可以近似的认为此时 `webapp` 已启动成功了

接下来是 `MyDispatcherServlet`，在其 `init` 方法执行中，会触发第二次的 `onApplicationEvent`，只有完成 `init` 后，`webapp` 才算是真正的启动完成

