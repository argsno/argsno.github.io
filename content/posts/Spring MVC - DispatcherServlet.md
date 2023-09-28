---
title: "Spring MVC - DispatcherServlet"
date: 2019-04-25T15:41:09+08:00
draft: true
---

DispatcherServlet作为Spring MVC的核心类，本文希望从源码角度对其核心原理进行熟悉。

核心数据结构
```java
    /** MultipartResolver used by this servlet. */
    @Nullable
    private MultipartResolver multipartResolver;

    /** LocaleResolver used by this servlet. */
    @Nullable
    private LocaleResolver localeResolver;

    /** ThemeResolver used by this servlet. */
    @Nullable
    private ThemeResolver themeResolver;

    /** List of HandlerMappings used by this servlet. */
    @Nullable
    private List<HandlerMapping> handlerMappings;

    /** List of HandlerAdapters used by this servlet. */
    @Nullable
    private List<HandlerAdapter> handlerAdapters;

    /** List of HandlerExceptionResolvers used by this servlet. */
    @Nullable
    private List<HandlerExceptionResolver> handlerExceptionResolvers;

    /** RequestToViewNameTranslator used by this servlet. */
    @Nullable
    private RequestToViewNameTranslator viewNameTranslator;

    /** FlashMapManager used by this servlet. */
    @Nullable
    private FlashMapManager flashMapManager;

    /** List of ViewResolvers used by this servlet. */
    @Nullable
    private List<ViewResolver> viewResolvers;
```

入口方法是实现自`FrameworkServlet`类的`void doService(HttpServletRequest request, HttpServletResponse response) throws Exception`方法：
```java
    /**
     * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
     * for the actual dispatching.
     */
    @Override
    protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // 进行简单的处理后，会交给doDispatch方法进行分发
        doDispatch(request, response);
    }
```

doDispatch方法：
```java
    /**
     * Process the actual dispatching to the handler.
     * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
     * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
     * to find the first that supports the handler class.
     * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
     * themselves to decide which methods are acceptable.
     * @param request current HTTP request
     * @param response current HTTP response
     * @throws Exception in case of any kind of processing failure
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // Determine handler for the current request.
        // 确定handler
        mappedHandler = getHandler(processedRequest);

        // Determine handler adapter for the current request.
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // 拦截器preHandle处理
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
        }

        // Actually invoke the handler.
        // 调用handler处理请求
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        applyDefaultViewName(processedRequest, mv);
        // 拦截器postHandle处理
        mappedHandler.applyPostHandle(processedRequest, response, mv);

        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

processDispatchResult方法：
```java
    /**
     * Handle the result of handler selection and handler invocation, which is
     * either a ModelAndView or an Exception to be resolved to a ModelAndView.
     */
    private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
            @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
            @Nullable Exception exception) throws Exception {
        // 发生异常时，获取异常的ModelAndView
        if (exception != null)
        render(mv, request, response);
    }
```

render方法：
```java
    /**
     * Render the given ModelAndView.
     * <p>This is the last stage in handling a request. It may involve resolving the view by name.
     * @param mv the ModelAndView to render
     * @param request current HTTP servlet request
     * @param response current HTTP servlet response
     * @throws ServletException if view is missing or cannot be resolved
     * @throws Exception if there's a problem rendering the view
     */
    protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
        // 获取View
        View view;
        if (viewName != null) {
            // We need to resolve the view name.
            // 解析view名称
            view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
        }
        else {
            // No need to lookup: the ModelAndView object contains the actual View object.
            view = mv.getView();
        }

        // Delegate to the View object for rendering.
        // 委托给View进行呈现
        try {
            if (mv.getStatus() != null) {
                response.setStatus(mv.getStatus().value());
            }
            view.render(mv.getModelInternal(), request, response);
        }
    }
```
