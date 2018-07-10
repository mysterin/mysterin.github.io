---
title: springmvc 上下文加载过程二
date: 2018-07-09 16:13:30
tags: [spring, springmvc, context]
---

#### DispatcherServlet
HttpServlet 是 web 容器标准之一, 容器会把匹配成功的 URL 请求转发给 HttpServlet 处理, 而 springmvc 中提供了 DispatcherServlet, 它继承 HttpServlet, 它的初始化也是由 web 容器调用 init(ServletConfig config) 完成<!-- more -->, 代码如下:
```java
/**
 * 可以看出实际是调用无参 init() 方法完成初始化
 */
public void init(ServletConfig config) throws ServletException {
    this.config = config;
    this.init();
}
```
一般在 web.xml 配置 Servlet 是这样的:
```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

```

#### init
这个 init 方法是 DispatcherServlet 的父类 HttpServletBean 提供
```java
public final void init() throws ServletException {
    // 省略日志打印
    try {
    	// 这里使用 HttpServletBean 的静态内部类构造一个键-值的对象
    	// 它会遍历在 web.xml 为 DispatcherServlet 配置的参数
    	// 然后保存对应的键-值
        PropertyValues pvs = new HttpServletBean.ServletConfigPropertyValues(
            this.getServletConfig(), this.requiredProperties);

        // BeanWrapper 是一个对 bean 包装的类
        // 使用 BeanWrapper 设置键-值时, 会对包装的 bean 设置对应的属性值
        // 比如这里包装的 bean 是 DispatcherServlet, 拥有属性: contextConfigLocation
        // 然后参考上面的 web.xml 配置了 contextConfigLocation 的值是配置文件路径
        // 当 BeanWrapper 调用 setPropertyValues 方法时, 它就会自然地把
        // contextConfigLocation 的值设置到 DispatcherServlet 的属性值
        // 所以通过这一系列操作就已经设置了 spring-mvc.xml 这个配置文件了
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
        ResourceLoader resourceLoader = new ServletContextResourceLoader(
            this.getServletContext());
        bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader,
            this.getEnvironment()));
        // 这个方法是空的, 应该是为了扩展用
        this.initBeanWrapper(bw);
        bw.setPropertyValues(pvs, true);
    } catch (BeansException var4) {
        // 省略日志打印
        throw var4;
    }

    // 主要是这里完成初始化
    this.initServletBean();
    // 省略日志打印
}
```

#### initServletBean
这个方法是 DispatcherServlet 的父类 FrameworkServlet 提供
```java
protected final void initServletBean() throws ServletException {
    // 省略日志打印

    long startTime = System.currentTimeMillis();

    try {
    	// 这里完成子上下文初始化
        this.webApplicationContext = this.initWebApplicationContext();
        // 还是空方法, 方便扩展
        this.initFrameworkServlet();
    } catch (ServletException var5) {
        this.logger.error("Context initialization failed", var5);
        throw var5;
    } catch (RuntimeException var6) {
        this.logger.error("Context initialization failed", var6);
        throw var6;
    }

    // 省略日志打印
}
```

#### initWebApplicationContext
这个方法是 DispatcherServlet 的父类 FrameworkServlet 提供
```java
protected WebApplicationContext initWebApplicationContext() {
    // 根据 ServletConfig 获取 ServletContext
    // 再根据 ServletContext 获取父上下文
    WebApplicationContext rootContext = 
        WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
    WebApplicationContext wac = null;

    // 没有专门设置, webApplicationContext 一般都是空, 所以不会走这一步
    if (this.webApplicationContext != null) {
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = 
                (ConfigurableWebApplicationContext)wac;
            if (!cwac.isActive()) {
                if (cwac.getParent() == null) {
                    cwac.setParent(rootContext);
                }

                this.configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }

    // 如何能在 ServletContext 找到上下文, 就用这个上下文作为子上下文
    // 默认都是没有找到的
    if (wac == null) {
        wac = this.findWebApplicationContext();
    }

    // 这里才是默认创建子上下文的步骤
    // 注意这里接受了父上下文作为参数
    if (wac == null) {
        wac = this.createWebApplicationContext(rootContext);
    }

    if (!this.refreshEventReceived) {
        this.onRefresh(wac);
    }

    if (this.publishContext) {
        String attrName = this.getServletContextAttributeName();
        this.getServletContext().setAttribute(attrName, wac);
        // 省略日志打印
    }

    return wac;
}

/**
 * 根据 ServletConfig 获取 ServletContext
 */
public final ServletContext getServletContext() {
    return this.getServletConfig() != null ? 
        this.getServletConfig().getServletContext() : null;
}

/**
 * 根据 ServletContext 获取父上下文
 * ServletContext 如何保存父上下文可以参考上一 part
 */
public static WebApplicationContext getWebApplicationContext(ServletContext sc) {
    return getWebApplicationContext(sc, 
        WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
}

public static WebApplicationContext 
        getWebApplicationContext(ServletContext sc, String attrName) {
    Assert.notNull(sc, "ServletContext must not be null");
    Object attr = sc.getAttribute(attrName);
    if (attr == null) {
        return null;
    } else if (attr instanceof RuntimeException) {
        throw (RuntimeException)attr;
    } else if (attr instanceof Error) {
        throw (Error)attr;
    } else if (attr instanceof Exception) {
        throw new IllegalStateException((Exception)attr);
    } else if (!(attr instanceof WebApplicationContext)) {
        throw new IllegalStateException(
            "Context attribute is not of type WebApplicationContext: " + attr);
    } else {
        return (WebApplicationContext)attr;
    }
}
```

#### createWebApplicationContext
同样还是 FrameworkServlet 提供的方法
```java
protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
    // 这里获取子上下文的默认类型
    // 可以追溯到子上下文的默认类型也是 XmlWebApplicationContext, 和父上下文一样
    Class<?> contextClass = this.getContextClass();
    // 省略日志打印

    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        // 省略抛异常
    } else {
    	// 通过反射实例化 XmlWebApplicationContext
        ConfigurableWebApplicationContext wac = 
          (ConfigurableWebApplicationContext)BeanUtils.instantiateClass(contextClass);
        wac.setEnvironment(this.getEnvironment());
        // 设置父上下文, 也就是说子上下文会保存父上下文的引用
        // 所以子上下文找不到的 bean 可以去父上下文找
        wac.setParent(parent);
        // 设置配置文件路径
        wac.setConfigLocation(this.getContextConfigLocation());

        // 配置子上下文以及创建相关 bean
        this.configureAndRefreshWebApplicationContext(wac);
        return wac;
    }
}
```

#### configureAndRefreshWebApplicationContext
可以看出在上一 part 也有一个相同名字的方法说明, 其实两个方法的作用是一致的, 都是配置上下文然后执行刷新操作, 也就是创建相关 bean 的操作. 只不过上一 part 的方法是属于 ContextLoader 类, 而这里的方法是属于 FrameworkServlet 类. 他们的区别主要在于刷新操作上, 但是
```java
protected void 
    configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    // 同样为上下文设置 id
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        if (this.contextId != null) {
            wac.setId(this.contextId);
        } else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX + 
            ObjectUtils.getDisplayString(this.getServletContext().getContextPath()) + 
            "/" + this.getServletName());
        }
    }

    // 为上下文设置 ServletContext 和 ServletConfig
    wac.setServletContext(this.getServletContext());
    wac.setServletConfig(this.getServletConfig());
    // 设置命名空间, 默认是 [Servlet名字]-servlet
    wac.setNamespace(this.getNamespace());
    wac.addApplicationListener(new SourceFilteringListener(wac, 
        new FrameworkServlet.ContextRefreshListener(null)));
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment)env)
            .initPropertySources(this.getServletContext(), this.getServletConfig());
    }
    // 空方法, 扩展用
    this.postProcessWebApplicationContext(wac);
    // 这里应该是根据对配置的 class 实例化, 一般应该用不上
    this.applyInitializers(wac);
    // 这里具体刷新操作不详细讨论
    wac.refresh();
}
```

#### 默认配置文件路径
有个点是需要注意的, 就算在 web.xml 没有设置配置文件的路径, spring 也提供了默认的配置文件路径. 参考这两 part 的内容, 我们已经可以确定上下文实例是使用 XmlWebApplicationContext 类创建的, 而在这个类里重写了默认配置文件的方法:
```java
/**
 * 如果没有配置 contextConfigLocation, 那么默认返回这个方法的值
 * 这里还会判断有没有设置命名空间, 而父上下文是没有设置命名空间的, 子上下文才会设置
 * 所以对于父上下文, 默认配置路径是: /WEB-INF/applicationContext.xml
 * 对于子上下文, 默认配置路径是: /WEB-INF/[Servlet名字]-servlet.xml
 */
protected String[] getDefaultConfigLocations() {
    return this.getNamespace() != null ? 
        new String[]{"/WEB-INF/" + this.getNamespace() + ".xml"} : 
        new String[]{"/WEB-INF/applicationContext.xml"};
}
```

总的来说, 这两 part 也只是简单说了 web 容器是如何启动 spring 容器的, 主要分成两步, 一是创建 `XmlWebApplicationContext` 实例; 二是为实例设置配置文件路径. 至于更加复杂的 bean 的创建管理是在上下文的刷新方法 refresh() 中进行的, 以后有机会再来详细探讨.

---
>*如果有疑问欢迎来 [Issues](https://github.com/mysterin/mysterin.github.io/issues) 探讨*