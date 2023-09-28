---
title: "Spring MVC - 请求处理流程"
date: 2019-04-24T19:34:21+08:00
draft: false
---

在Spring MVC框架中，用户发起请求，从“Request（请求）”开始，会依次进入“DispatcherServlet（核心分发器）” —> “HandlerMapping（处理器映射）” —> “Controller（控制器）” —> “ModelAndView（模型和视图）” —> “ViewResolver（视图解析器）” —> “View（视图）” —> “Response（响应）”结束，其中DispatcherServlet、HandlerMapping和ViewResolver只需要在XML文件中配置即可，从而大大提高了开发的效率，特别是对于 HandlerMapping 框架为其提供了默认的配置。涉及到了数据结构主要如下：
![](https://ws2.sinaimg.cn/large/006tNc79ly1g2dym7tnj6j30d707xdfo.jpg)

Spring MVC框架的流程图如下所示：
![](https://ws4.sinaimg.cn/large/006tNc79ly1g2dyg5cfvgj30k40b775q.jpg)

主要步骤如下：

1. 在请求离开浏览器时，会带有用户所请求内容的信息，至少会包含 请求的URL。但是还可能带有其他的信息，例如用户提交的表单信息。请求旅程的第一站是Spring的DispatcherServlet。前端控制器是常用的Web应用程序模 式，在这里一个单实例的Servlet将请求委托给应用程序的其他组件来 执行实际的处理。在Spring MVC中，DispatcherServlet就是前端控制器。
2. DispatcherServlet的任务是将请求发送给Spring MVC控制器 （controller）。控制器是一个用于处理请求的Spring组件。在典型的应用程序中可能会有多个控制器，DispatcherServlet需要知道 应该将请求发送给哪个控制器。所以DispatcherServlet以会查询一个或多个处理器映射（handler mapping） 来确定请求的下一站在哪里。处理器映射会根据请求所携带的URL信息来进行决策。
3. 一旦选择了合适的控制器，DispatcherServlet会将请求发送给选中的控制器。到了控制器，请求会卸下其负载（用户提交的信息）并耐心等待控制器处理这些信息。（实际上，设计良好的控制器 本身只处理很少甚至不处理工作，而是将业务逻辑委托给一个或多个服务对象进行处理。）
4. 控制器在完成逻辑处理后，通常会产生一些信息，这些信息需要返回给用户并在浏览器上显示。这些信息被称为模型（model）。不过，仅仅给用户返回原始的信息是不够的——这些信息需要以用户友好的方式进行格式化。所以，信息需要发送给一个视图 （view）。控制器所做的最后一件事就是将模型数据打包，并且标示出用于渲染输出的视图名。它接下来会将请求连同模型和视图名发送回 DispatcherServlet。
5. 传递给 DispatcherServlet的视图名仅仅是一个逻辑名称，这个名字将会用来查找产生结果的真正视图。DispatcherServlet将会使用视图解析器（view resolver） 来将逻辑视图名匹配为一个特定的视图实现。
6. 请求的最后一站是视图的实现，在这里它将交付模型数据。
7. 视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端。