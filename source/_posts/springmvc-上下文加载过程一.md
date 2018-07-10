---
title: springmvc 上下文加载过程一
date: 2018-07-06 10:58:48
tags: [spring, springmvc, context]
---

#### 上下文
spring 的根本是 IOC 和 AOP, 这两个点这里不展开讨论, 只需要知道 spring 是依靠容器来管理 bean, 从而实现 IOC 和 AOP 功能, 而这里的 spring 容器则是指上下文 Context. Context 实现了工厂接口, 所以可以通过 Context 直接获取被 spring 实例化的 bean.<!-- more --> 一般来说 spring 实例化 bean 有两种方式, 一种是用注解比如 `@Controller`, `@Service`, `@Repository` 等, 设置 spring 的扫描范围内有这些注解的类就会被实例化, 加入到上下文 Context 中; 另一种是在配置文件用 `<bean/>` 标签配置, 效果是一样的.

#### 两个配置文件
springmvc 的传统做法是有两个配置文件, 一个一般命名为 applicationContext.xml, 另一个一般命名为 spring-mvc.xml. 这两个配置文件其实是差不多的, 都是用来初始化 spring 上下文的. 其中 applicationContext.xml 是对应父上下文, spring-mvc.xml 是对应子上下文, 这里父子关系并不是继承关系, 而是在子上下文里有个引用保存父上下文. 因此如果子上下文找不到对应的 bean, 那么它会自动去父上下文找; 反过来则不行, 因为父上下文没有保存子上下文的引用, 所以父上下文找不到 bean 是不能去子上下文找的.
一般来讲, applicationContext.xml 是配置服务层, 持久层的扫描, 数据库的配置以及事务配置, 当然像线程池和定时任务这些都可以放到这里配置; 而 spring-mvc.xml 是配置控制层扫描, 视图管理等等这些, 简单来讲就是涉及对外出口的那部分配置, 而非对外出口的配置都可以放到 applicationContext.xml 里.
这里需要注意的是我们要知道哪些 bean 是在父上下文实例化, 哪些 bean 是在子上下文实例化, 这样才避免不必要错误. 比如在 applicationContext.xml 配置了事务功能, 但是却对控制层使用事务注解, 那样当然是没有效果的, 因为控制层的实例化是在子上下文, 而 spring-mvc.xml 没有配置事务功能.
其实这样看来是没有必要分两个配置文件的, 一个配置文件就可以搞定所有, 但是网上说分两个配置文件是为了方便扩展, 而且大多数都是分两个文件, 所以就不需要纠结这个了, 只要知道两个配置文件的区别就可以了.

#### web.xml
springmvc 上下文的初始化入口是在 web.xml 配置的, tomcat 这些 web 容器会读取 web.xml, 然后初始化里面配置的 listener, filter, servlet. 一般是用 ContextLoaderListener 这个类去初始化父上下文, DispatcherServlet 初始化子上下文. 这两个类都是 spring 提供的, 它们会读指定的配置文件也就是 applicationContext.xml 和 spring-mvc.xml, 然后初始化对应上下文.

---

#### ContextLoaderListener 初始化过程
这个类实现了 ServletContextListener 接口, 这个接口是 javax.servlet-api 提供的, 属于 web 标准的一部分, 所以 tomcat 这些 web 容器才能自动初始化 listener, 也就是自动调用 contextInitialized(ServletContexgtEvent sce) 来完成初始化. 这里我们主要是讨论如何去创建上下文的实例, 至于里面涉及到的 bean 的生成之类就不展开讨论了. 在 web.xml 的相关配置一般如下:
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <!-- 注意: 冒号后面是不能有空格的 -->
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener
    </listener-class>
</listener>

```

```java
public class ContextLoaderListener extends ContextLoader implements 
	ServletContextListener {
    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }

    /**
     * 实际是调用父类 ContextLoader 完成初始化
     */
    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
    }

    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}
```

#### ContextLoader 静态代码块
ContextLoader 类首先会执行静态代码块, 这里是加载 ContextLoader 这个类所在目录下的 ContextLoader.properties 文件, 这个文件只有一个配置: 
```
org.springframework.web.context.WebApplicationContext = 
	org.springframework.web.context.support.XmlWebApplicationContext
```
这个配置是用来表示默认的上下文的类型, 当需要实例化上下文时, 就会读取这个配置把 `XmlWebApplicationContext` 这个类实例化.
```java
static {
    try {
        ClassPathResource resource = 
            new ClassPathResource("ContextLoader.properties", ContextLoader.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    } catch (IOException var1) {
        throw new IllegalStateException("Could not load 'ContextLoader.properties': " +
            var1.getMessage());
    }

    currentContextPerThread = new ConcurrentHashMap(1);
}
```

#### initWebApplicationContext
这个方法是 ContextLoaderListener 的父类 ContextLoader 的方法, 接收参数是 ServletContext. ServletContext 是 web 容器启动时为每个应用创建的, 它表示当前 web 应用.
```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 最后生成的上下文都是保存到 servletContext 中
    // key 值就是 WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE
    // 所以如果一开始 servletContext 就有上下文就抛出异常
    if (servletContext.getAttribute(
        WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        // 抛异常
    } else {
        // 省略日志打印
        try {
            // 这里就是创建父上下文, 放到下面说
            if (this.context == null) {
                this.context = this.createWebApplicationContext(servletContext);
            }

            // 读取配置文件, 放到下面说
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = 
                    (ConfigurableWebApplicationContext)this.context;
                if (!cwac.isActive()) {
                    if (cwac.getParent() == null) {
                        ApplicationContext parent = 
                            this.loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }

                    this.configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }

            // 把父上下文保存到 servletContext 中
            // 然后在子上下文创建时, 就可以通过这个来获取父上下文了
            servletContext.setAttribute(
            WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
            
            // 这里看的不是很明白, 暂时不讨论
            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            } else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }

            // 省略日志打印
            return this.context;
        } catch (RuntimeException var8) {
            logger.error("Context initialization failed", var8);
            servletContext.setAttribute(
                WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, var8);
            throw var8;
        } catch (Error var9) {
            logger.error("Context initialization failed", var9);
            servletContext.setAttribute(
                WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, var9);
            throw var9;
        }
    }
}
```

#### createWebApplicationContext
这里主要来讨论如何找到对应上下文的类来初始化
```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // 根据下面分析, 这里是返回 XmlWebApplicationContext 的 Class 对象
    // 也就是说默认上下文是用 XmlWebApplicationContext 表示
    Class<?> contextClass = this.determineContextClass(sc);
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        // 抛异常
    } else {
        // 这里就是通过反射方式将上下文实例化
        return 
            (ConfigurableWebApplicationContext)BeanUtils.instantiateClass(contextClass);
    }
}

protected Class<?> determineContextClass(ServletContext servletContext) {
    // 这里会读取 web.xml 用 <context-param> 配置的 contextClass 参数
    // 如果有值, 就用这个值表示的类进行上下文实例创建
    // 一般我们都不会配置这个参数, 所以是返回空值
    String contextClassName = servletContext.getInitParameter("contextClass");
    if (contextClassName != null) {
        try {
            return ClassUtils.forName(
                contextClassName, ClassUtils.getDefaultClassLoader());
        } catch (ClassNotFoundException var4) {
            // 抛异常
        }
    } else {
        // 所以一般都是用默认的上下文类型初始化
        // 这里是读取在上面说的静态代码块中的配置文件的属性
        // WebApplicationContext 这个类的名字跟配置文件的 key 其实是一样的
        // 因此这里返回的就是 XmlWebApplicationContext 这个类的名字了
        // 然后通过这个类名去加载对应的 class 文件
        contextClassName = 
            defaultStrategies.getProperty(WebApplicationContext.class.getName());

        try {
            return ClassUtils.forName(
                contextClassName, ContextLoader.class.getClassLoader());
        } catch (ClassNotFoundException var5) {
            // 抛异常
        }
    }
}
```

#### configureAndRefreshWebApplicationContext
这个方法是读取配置文件, 然后创建相关 bean. 这里就不仔细讨论 bean 的创建过程, 比较繁琐.
```java
protected void configureAndRefreshWebApplicationContext(
	    ConfigurableWebApplicationContext wac, ServletContext sc) {
    String configLocationParam;
    // 给父上下文设置 id
    // 如果在 web.xml 有设置参数 contextId, 那么就用这个作为父上下文 id
    // 否则就默认生成一个 id
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        configLocationParam = sc.getInitParameter("contextId");
        if (configLocationParam != null) {
            wac.setId(configLocationParam);
        } else {
            wac.setId(
                ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX + 
                ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }

    // 父上下文保存 ServletConfig 的引用
    wac.setServletContext(sc);

    // 设置 web.xml 配置的 contextConfigLocation 参数, 也就是配置文件路径
    configLocationParam = sc.getInitParameter("contextConfigLocation");
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment)env).initPropertySources(sc, (ServletConfig)null);
    }
    // 这步看不懂, 跳过
    this.customizeContext(sc, wac);
    // 这里开始 bean 的创建, 不详细讨论了
    wac.refresh();
}
```

到这里, 我们知道了 spring 是如何创建父上下文的实例了, 关于子上下文的创建留到下一 part 再讲.

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*