---
title: "Spring MVC - Controller vs RestController"
date: 2019-04-25T15:22:14+08:00
draft: false
---

Spring基于注解的MVC框架简化了RESTful Web服务的开发流程。传统的Spring MVC控制器和RESTful服务控制器的主要区别是HTTP的ResponseBody创建方式的不同。传统MVC控制器ReponseBody创建是基于视图（View）技术，而RESTful服务仅仅返回一个对象，将对象进行序列化为JSON/XML/String并作为HTTP Response返回。传统MVC控制器使用`@Controller`注解而RESTful服务使用`@RestController`注解。

看下这两个注解的实现：
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
    @AliasFor(annotation = Component.class)
	String value() default "";
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
	String value() default "";
}

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ResponseBody {

}
```
可以看到`@RestController`注解等于`@Controller`加上`@ResponseBody`。

在实际使用中，`@ResponseBody`也可以直接加在特定的方法上，或者返回值类型的位置。