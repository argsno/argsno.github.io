---
title: "Filter（过滤器）和Interceptor（拦截器）"
date: 2019-04-23T15:41:00+08:00
draft: false
---

## 引言
过滤器（Filter）和拦截器（Interceptor）由于功能差不多，都可以用来对请求进行预处理和对响应结果进行处理，可能比较容易搞混，这里从具体的功能和实现原理进行区别一下，做个笔记。

## Filter
Filter是Servlet标准的一部分，依赖于servlet容器，主要用于对用户请求进行预处理，也可以对HttpServletResponse进行后处理，是个典型的处理调用链。Filter也可以对用户请求生成响应，这一点与Servlet相同，但实际上很少会使用Filter向用户请求生成响应。使用Filter完整的流程是：Filter对用户请求进行预处理，接着将请求交给Servlet进行预处理并生成响应，最后Filter再对服务器响应进行后处理。

Filter有如下几个用处：

- 在HttpServletRequest到达Servlet之前，拦截客户的HttpServletRequest。
- 根据需要检查HttpServletRequest，也可以修改HttpServletRequest头和数据。
- 在HttpServletResponse到达客户端之前，拦截HttpServletResponse。
- 根据需要检查HttpServletResponse，也可以修改HttpServletResponse头和数据。

Filter有如下几个种类：

- 用户授权的Filter：Filter负责检查用户请求，根据请求过滤用户非法请求。
- 日志Filter：详细记录某些特殊的用户请求。
- 负责解码的Filter:包括对非标准编码的请求解码。
- 能改变XML内容的XSLT Filter等。
- Filter可以负责拦截多个请求或响应；一个请求或响应也可以被多个Filter拦截。

创建一个Filter只需两个步骤：

- 创建Filter处理类
- web.xml文件中配置Filter

创建Filter必须实现javax.servlet.Filter接口，在该接口中定义了如下三个方法：

- `void init(FilterConfig config)`: 用于完成Filter的初始化。
- `void destory()`: 用于Filter销毁前，完成某些资源的回收。
- `void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)`: 实现过滤功能，该方法就是对每个请求及响应增加的额外处理。该方法可以实现对用户请求进行预处理(ServletRequest request)，也可实现对服务器响应进行后处理(ServletResponse response) —- 它们的分界线为是否调用了chain.doFilter()，执行该方法之前，即对用户请求进行预处理；执行该方法之后，即对服务器响应进行后处理。

### 实现原理
```java
public interface Filter {
    // 容器创建的时候调用, 即启动tomcat的时候调用
    public void init(FilterConfig filterConfig) throws ServletException;
    
    // 由FilterChain调用, 并且传入Filter Chain本身
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;
    
    // 容器销毁的时候调用, 即关闭tomcat的时候调用
    public void destroy();
}

// FilterChain
public interface FilterChain {
    // 由Filter.doFilter()中的chain.doFilter调用
    public void doFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException;
}
```

在tomcat中Filter Chain的默认实现是ApplicationFilterChain, 在ApplicationFilterChain中最关键的方法就是internalDoFilter, 整个Filter流程的实现就是由这个方法完成.
```java
// internalDoFilter(只保留关键代码)
private void internalDoFilter(ServletRequest request, ServletResponse response)
    throws IOException, ServletException {

    // Call the next filter if there is one
    // pos: 当前的filter的索引, n: 调用链中所有的Filter的数量
    // 如果调用链中还有没有调用的Filter就继续调用, 否则跳过if语句
    if (pos < n) {
        ApplicationFilterConfig filterConfig = filters[pos++];
        try {
            // 获取Filter
            Filter filter = filterConfig.getFilter();
            if( Globals.IS_SECURITY_ENABLED ) {
                ...
                其他代码
                ...    
            } else {
                // 这句话是重点, 调用Filter的doFilter方法并把Filter Chain本身传进去(this)
                filter.doFilter(request, response, this);
            }
        } catch (IOException | ServletException | RuntimeException e) {
            ...
            异常处理代码
            ...    
        }
        return;
    }

    // We fell off the end of the chain -- call the servlet instance
    try {
        ...
        其他代码
        ...
        // Use potentially wrapped request from this point
        if ((request instanceof HttpServletRequest) &&
                (response instanceof HttpServletResponse) &&
                Globals.IS_SECURITY_ENABLED ) {
            ...
            其他代码
            ...
        } else {
                // 调用真正的Filter
            servlet.service(request, response);
        }
    } catch (IOException | ServletException | RuntimeException e) {
        ...
        异常处理代码
        ...
    } finally {
        ...
        始终要执行的代码
        ...
    }
}
```

## Interceptor
Interceptor 是Spring MVC实现的一部分，在AOP(Aspect-Oriented Programming)中用于在某个方法或字段被访问之前，进行拦截，然后在之前或之后加入某些操作。Spring MVC 中的Interceptor 拦截请求是通过HandlerInterceptor 来实现的。

在SpringMVC 中定义一个Interceptor 非常简单，主要有两种方式，第一种方式是要定义的Interceptor类要实现了Spring 的HandlerInterceptor 接口，或者是这个类继承实现了HandlerInterceptor 接口的类，比如Spring 已经提供的实现了HandlerInterceptor 接口的抽象类HandlerInterceptorAdapter ；第二种方式是实现Spring的WebRequestInterceptor接口，或者是继承实现了WebRequestInterceptor的类。

### 实现HandlerInterceptor接口
HandlerInterceptor 接口中定义了三个方法，我们就是通过这三个方法来对用户的请求进行拦截处理的。

- preHandle(HttpServletRequest request, HttpServletResponse response, Object handle) 方法，顾名思义，该方法将在请求处理之前进行调用。SpringMVC 中的Interceptor 是链式的调用的，在一个应用中或者说是在一个请求中可以同时存在多个Interceptor 。每个Interceptor 的调用会依据它的声明顺序依次执行，而且最先执行的都是Interceptor 中的preHandle 方法，所以可以在这个方法中进行一些前置初始化操作或者是对当前请求的一个预处理，也可以在这个方法中进行一些判断来决定请求是否要继续进行下去。该方法的返回值是布尔值Boolean 类型的，当它返回为false 时，表示请求结束，后续的Interceptor 和Controller 都不会再执行；当返回值为true 时就会继续调用下一个Interceptor 的preHandle 方法，如果已经是最后一个Interceptor 的时候就会是调用当前请求的Controller 方法。

- postHandle(HttpServletRequest request, HttpServletResponse response, Object handle, ModelAndView modelAndView) 方法，由preHandle方法的解释我们知道这个方法包括后面要说到的afterCompletion方法都只能是在当前所属的Interceptor的preHandle方法的返回值为true 时才能被调用。postHandle方法，顾名思义就是在当前请求进行处理之后，也就是Controller 方法调用之后执行，但是它会在DispatcherServlet进行视图返回渲染之前被调用，所以我们可以在这个方法中对Controller处理之后的ModelAndView对象进行操作。postHandle方法被调用的方向跟preHandle是相反的，也就是说先声明的Interceptor的postHandle方法反而会后执行。

- afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handle, Exception ex) 方法，该方法也是需要当前对应的Interceptor的preHandle 方法的返回值为true时才会执行。顾名思义，该方法将在整个请求结束之后，也就是在DispatcherServlet渲染了对应的视图之后执行。这个方法的主要作用是用于进行资源清理工作的。

### 实现
Interceptor功能的实现主要是在Spring Mvc的DispatcherServlet.doDispatch方法中, 让我们来看看源码

```java
// Interceptor的源码
public interface HandlerInterceptor {

    // 在调用真正的处理请求类之前调用
    boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception;

    // 在调用真正的处理请求类之后调用
    void postHandle(
            HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
            throws Exception;

 // 在完成渲染或者出错之后调用
    void afterCompletion(
            HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception;

}
```

```java
    // doDispatch源码(只保留关键代码)
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        try {
            ModelAndView mv = null;
            Exception dispatchException = null;

            try {
                ....
                其它的处理代码
                ....
                
                // 调用拦截器的前置处理方法
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }

                // Actually invoke the handler.
                // 调用真正的处理请求的方法
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                // 找到渲染模版
                applyDefaultViewName(processedRequest, mv);
                
                // 调用拦截器的后置处理方法
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            }
            catch (Exception ex) {
                ....
                异常处理代码
                ....
        }
        finally {
            ....
            始终要执行的代码
            ....
    }
```
Spring Mvc对整个请求的处理流程：调用拦截器的前置方法 -> 调用处理请求的方法 -> 渲染模版 -> 调用拦截器的后置处理方法 -> 调用拦截器的完成方法

接下来看一看Spring Mvc是如何实现依此调用这么多拦截器的前置方法, 后置方法, 完成方法的

进入到mapperHandler.applyPreHandle()方法中(调用拦截器的前置方法)
```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HandlerInterceptor[] interceptors = getInterceptors();
    // 如果拦截器数组不为空
    if (!ObjectUtils.isEmpty(interceptors)) {
       // 按顺序调用拦截器数组中的preHandle方法
        for (int i = 0; i < interceptors.length; i++) {
            HandlerInterceptor interceptor = interceptors[i];
            // 如果拦截器的preHandle方法返回false, 则调用当前拦截器的triggerAfterCompletion方法, 然后返回, 并且不再调用后续的拦截器
            if (!interceptor.preHandle(request, response, this.handler)) {
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
    }
    return true;
}
```

进入到mappedHandler.applyPostHandle()方法中(调用拦截器的后置方法)
```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) throws Exception {
    HandlerInterceptor[] interceptors = getInterceptors();
    // 如果拦截器数组不为空
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 倒序调用拦截器数组中拦截器的postHandle方法
        for (int i = interceptors.length - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            interceptor.postHandle(request, response, this.handler, mv);
        }
    }
}
```

不管是否出异常triggerAfterCompletion方法始终会被调用
```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex)
        throws Exception {
    HandlerInterceptor[] interceptors = getInterceptors();
    // 拦截器数组不为空
    if (!ObjectUtils.isEmpty(interceptors)) {
       // 从成功执行的最后一个拦截器开始逆序调用afterCompletion方法
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            }
            catch (Throwable ex2) {
                logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
            }
        }
    }
}
```

看过以上三个方法之后, Spring Mvc如何处理拦截器的前置, 后置, 完成方法就一目了然了. 其实Spring Mvc就是将拦截器统一放到了拦截器数组中, 然后在调用真正的处理请求方法之前和之后正序或者倒序遍历拦截器, 同时调用拦截器的相应的方法. 最后不管是否正常结束这个流程还是出异常都会从成功的最后一个拦截器开始逆序调用afterCompletion方法



## 总结
- 从以上分析可以看到过滤器和拦截器实现的方式的不同. Filter是利用了方法的调用(入栈出栈)完成整个流程, 而Interceptor是利用了for循环完成了整个流程.
- Filter是servlet规范中定义的java web组件, 在所有支持java web的容器中都可以使用。
- Filter组件更加的通用, 只要支持java servlet的容器都可以使用, 而Interceptor必须依赖于Spring
- Filter和Filter Chain是密不可分的, Filter可以实现依次调用正是因为有了Filter Chain。
- Filter的实现比较占用栈空间, 在Filter多的情况下可能会有栈溢出的风险存在.
- Interceptor不是servlet规范中的java web组件, 而是Spring提供的组件, 功能上和Filter差不多. 但是实现上和Filter不一样.
- Filter的优先级是高于Interceptor, 即请求是先到Filter再到Interceptor的, 因为Interceptor的实现主体还是一个servlet