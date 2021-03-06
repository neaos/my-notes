# Spring Boot WebMVC的异常处理

默认情况下，Spring Boot Web环境下，如果有异常发生，异常会以如下两种方法返回给调用者。

#### 使用浏览器时

此时浏览器会显示一个Whitelabel Error Page页面，这个页面会显示异常相关的简单信息。如下所示，可以看到500的响应码，请求的path，请求的时间，以及异常信息。![](/assets/default-exception-white-page.png)

### 使用App访问

常见的App和服务器交互一般使用json格式，如下所示：异常json中也包含了请求的path，请求时间，异常信息。![](/assets/default-exception-json.png)

### 原理

以Spring Boot2、Tomcat为例

#### 配置Tomcat特性

spring.boot.autoconfigure.jar的spring.factories中有如下配置：

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=... ...
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,... ...
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,... ...
```

即当 Spring Boot应用包含@Autconfiguration注解时自动引入EmbeddedWebServerFactoryCustomizerAutoConfiguration配置类。

EmbeddedWebServerFactoryCustomizerAutoConfiguration中配置了一个TomcatWebServerFactoryCustomizer类型的Bean。

![](/assets/EmbeddedWebServerFactoryCustomizerAutoConfiguration.png)

TomcatWebServerFactoryCustomizer是SpringBoot中专门定制Tomcat特性的工具类，customize方法中执行了customizeErrorReportValve\(properties.getError\(\), factory\)方法。properties.getError\(\)返回一个ErrorProperties实例，该实例中包含了属性path，默认值为/error，可以通过server.error.path修改默认值。

通过customizeErrorReportValve方法，Spring Boot默认将所有的异常都映射到了路径/error。

如果熟悉tomcat的配置，可以认为上面的操作对tomcat进行了如下配置：

```xml
<error-page>
   <exception-type>java.lang.Throwable</exception-type>
   <location>/error</location>
 </error-page>
```

#### 处理/error请求

ErrorMvcAutoConfiguration自动化配置类中包含如下配置：![](/assets/ErrorMvcAutoConfiguration.png)即用户没有自定义ErrorController时，构造一个BasicErrorController。

打开BasicErrorController，可以发现其中定义了/error映射，并且分别为html和json做了适配，如下：

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
 ... ...
    @RequestMapping(produces = "text/html")
    public ModelAndView errorHtml(HttpServletRequest request,
            HttpServletResponse response) {
        HttpStatus status = getStatus(request);
        Map<String, Object> model = Collections.unmodifiableMap(getErrorAttributes(
                request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
        response.setStatus(status.value());
        ModelAndView modelAndView = resolveErrorView(request, response, status, model);
        return (modelAndView != null ? modelAndView : new ModelAndView("error", model));
    }

    @RequestMapping
    @ResponseBody
    public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
        Map<String, Object> body = getErrorAttributes(request,
                isIncludeStackTrace(request, MediaType.ALL));
        HttpStatus status = getStatus(request);
        return new ResponseEntity<>(body, status);
    }
... ...
```

这样就实现了上面的异常处理效果。

## 自定义异常处理方式 

默认的异常处理方式的缺点显而易见，简陋的White Error Page，无法自定义的JSON消息格式。Spring Boot提供了多种方式对异常处理进行定制化。

### 自定义ErrorController

根据ErrorMvcAutoConfiguration中条件注解，只要有其他ErrorController实例存在，Spring Boot自定义的BasicErrorController就不会生效。最简单的方式是直接继承AbstractErrorController。如下：

```java
@Controller
@Profile(value = {"myErrorController"})
public class ExceptionHandleController extends AbstractErrorController {


    public ExceptionHandleController(ErrorAttributes errorAttributes) {
        super(errorAttributes);
    }

    @RequestMapping(value = "/error", produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseBody
    public Map<String, Object> handleError(HttpServletRequest request) {
        Map<String, Object> errorAttributes = super.getErrorAttributes(request, true);
        return errorAttributes;
    }

    @Override
    public String getErrorPath() {
        return "/error";
    }

}
```

启动参数添加-Dspring.profiles.active=myErrorController

执行结果：这里只是演示，实际不应暴露异常堆栈

```
{
    "timestamp" : "2018-05-21T15:43:04.918+0000",
    "status" : 500,
    "error" : "Internal Server Error",
    "message" : "This is Hello Exception",
    "trace" : "java.lang.RuntimeException: This is Hello Exception
    at com.ljm.springbootweberrorhandle.ExceptionController.hello(ExceptionController.java:15)
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.base/java.lang.reflect.Method.invoke(Method.java:564)
    at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:209)
    at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:136)
    at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:102)
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:877)
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:783)
    at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:991)
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:925)
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:974)
    at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:866)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:635)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:851)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:742)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:109)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:81)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:198)
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:496)
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140)
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81)
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:342)
    at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:803)
    at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
    at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:790)
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1468)
    at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
    at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
    at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
    at java.base/java.lang.Thread.run(Thread.java:844)
",
    "path" : "/hello"
}
```

### 自定义HttpCode页面文件

Spring Boot包含了一个名为DefaultErrorViewResolver的类，这个类提供了将Http状态码映射到页面文件的功能。ErrorMvcAutoConfiguration自动化配置类引入了DefaultErrorViewResolver。DefaultErrorViewResolver处理异常的优先级高于ErrorController，如果异常DefaultErrorViewResolver无法处理才会由ErrorController处理。

如果返回了特定状态码的异常，这个类会按照如下顺序查找页面，以500为例：

/&lt;templates&gt;/error/500.&lt;ext&gt;

/&lt;static&gt;/error/500.html

/&lt;templates&gt;/error/5xx.&lt;ext&gt;

/&lt;static&gt;/error/5xx.html

即优先使用模板处理，然后使用静态页面处理。因为模板可以获取具体的异常信息，而静态页面无法获取到这些信息。

注意：这种配置只能处理方式有很大的局限， 只能处理html的请求的异常，无法处理json请求的异常。

这里使用静态页面处理401响应为例。创建error目录，并添加静态页。

![](/assets/error-401.png)

创建异常类，并为其添加@ResponseStatus注解，@ResponseStatus注解的value属性指定了异常发生时的HTTP响应码。

```java
@ResponseStatus(HttpStatus.UNAUTHORIZED)
public class Http401Exception extends Exception {
    public Http401Exception(String s) {
        super(s);
    }
}
```

添加抛出401异常的方法：

```java
    @RequestMapping(value = "/401", produces = {MediaType.TEXT_HTML_VALUE})
    public String http401() throws Http401Exception {
        throw new Http401Exception("This HTTP 401 Exception");
    }
```

测试如下：

![](/assets/401jpage.png)

### 组合Spring MVC原有的异常处理方式

Spring MVC原有常用的异常方式通常为@ControllerAdvice注解，此注解优先级高于DefaultErrorViewResolver和ErrorController通常用于处理特定的异常。组合Spring Boot提供的异常处理机制，可以实现如下分工：

1. @ControllerAdvice处理特定异常。

2. DefaultErrorViewResolver处理特定的状态码。

3. ErrorController处理剩下的其他异常，如容器级别的异常。

在上面例子基础上再添加@ControllerAdvice：专门处理IllegAccessError

```java
@ControllerAdvice
public class MyExceptionAdvice {

    @ExceptionHandler(IllegalAccessError.class)
    @ResponseBody
    ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
        HttpStatus status = getStatus(request);
        ResponseEntity responseEntity = new ResponseEntity(ex, status);
        return responseEntity;
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
        if (statusCode == null) {
            return HttpStatus.INTERNAL_SERVER_ERROR;
        }
        return HttpStatus.valueOf(statusCode);
    }
}
```

```java
    @RequestMapping(value = "/access")
    public String accessError() {
        throw new IllegalAccessError("Illegal Access");
    }
```

效果：

![](/assets/illegAccessErrorResponseEntity.png)

注意：@Controlleradvice无法处理Filter中抛出的异常，这类异常只能被ErrorController处理。

```java
@Component
public class FilterWithException implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        if (((HttpServletRequest) request).getRequestURI().indexOf("filterError") != -1) {
            throw new IllegalAccessError("IllegalAccessError in Filter");
        }
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

Spring Boot能识别Filter，在Filter上添加@Component注解即可。此处抛出的IllegalAccessError，@ControllerAdivce是无法处理。

@ControllerAdivce只能处理@Controller、@RestController和HandlerInterceptor preHandle中抛出的异常\(postHandle

 中抛出的异常无法通过上面的任何方式处理，因为Response已经返回给调用者了\)。

效果如下：可以看到是由BasicErrorController处理的。

![](/assets/filterError.png)

完整代码：https://github.com/pkpk1234/springboot-web-errorhandle

