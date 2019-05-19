# Spring MVC处理方法

这两天发现对spring mvc处理方法注解还是一知半解，所以今天抽空写一下这方面的文章。其实就是对[Spring MVC Handler Method](https://docs.spring.io/spring/docs/5.1.7.RELEASE/spring-framework-reference/web.html#mvc-ann-methods)一章的大致翻译。

> 注：使用的是Spring 5.1.7版本

## 1.3.3 处理方法
@RequestMapping处理方法可以有各种签名，可以在支持的方法参数和返回值中选择

### 方法参数
JDK 8的java.util.Optional也支持作为方法参数与注解联合使用，这些注解需要有required属性（例如，@RequestParam, @RequestHeader以及其它），等效于required=false


| Controller方法参数 | 描述 |
| --- | --- |
| WebRequest, NativeWebRequest | 通用访问requesst参数和request和session属性，而不需要直接依赖于Servlet API |
| javax.servlet.ServletRequest, javax.servlet.ServletResponse | 可以选择任何指定的request和response类型，如ServletRequest, HttpServletRequest，或者Spring的MultipartRequest, MultipartHttpServletRequest |
|javax.servlet.http.HttpSession|强制Session存在。结果是：这个参数不可能为null。需要注意的是session访问不是线程安全的。如果多个请求被允许同时访问session，可以考虑设置RequestMappingHandlerAdapter实例的synchronizeOnSession为true|
|javax.servlet.http.PushBuilder|Servlet 4.0 的push builder API，用来编程方式操作HTTP/2 资源推送。需要注意的是：Servlet规范中，如果客户端不支持HTTP/2特性，则注入的PushBuilder实例可能为null|
|java.security.Principal|当前鉴权的用户。可能是一个Principal的特定实现|
|HttpMethod|请求的http方法|
|java.util.Locale|当前request的locale，由特定的LocaleResolver决定（事实上是配置的LocaleResolver或LocaleContextResolver）|
|java.util.TimeZone + java.time.ZoneId|当前的request关联的时区，由LocaleContextResolver决定|
|java.io.InputStream, java.io.Reader|用来访问原始的request body，其由Servlet API暴露|
|java.io.OutputStream, java.io.Writer|用来访问原始的response body，其由Servlet API暴露|
|@PathVariable|用来访问模板变量，参见URI模式|
|@MatrixVariable|用来访问URI路径部分的键值对，参见Matrix变量|
|@RequestParam|用来访问Servlet request参数，包括multipart files。请求值被转成申明的方法参数类型，参见@RequestParam和Multipart.|
|@RequestHeader|用来访问request头。值被转换为申明的方法参数类型。参见@RequestHeader.|
|@CookieValue|用来访问Cookie。Cookie值被转换为申明的方法参数类型。参见@CookieValue|
|@RequestBody|用来访问HTTP request body。Body内容通过HttpMessageConverter的实现类被转换为申明的方法参数类型，参见@RequestBody|
|HttpEntity<B>|用来访问request headers和body。body通过HttpMessageConverter来转换。参见HttpEntity|
|@RequestPart|用来访问multipart/form-data的part，使用HttpMessageConverter来转换part的body。参见Multipart.|
|java.util.Map, org.springframework.ui.Model, org.springframework.ui.ModelMap|用来访问模型。该模型在html控制器中使用，且作为视图渲染的一部分暴露到模板中|
|RedirectAttributes|指定了在redirect情况下使用的属性（被添加到查询字符串后）以及flash属性，会临时保存到redirect后的request。参见Redirect Attributes和Flash Attributes|
|@ModelAttribute|用来访问在model中已经存在的属性（不存在则实例化）且已经应用过data binding和validation。参见@ModelAttribute和Model和DataBinder.|
|Errors, BindingResult|用来访问命令对象（@ModelAttribute修饰的参数）在validation和data binding中的错误，或者@RequestBody或@RequestPart修饰的参数的validation错误。你必须在需要验证的方法参数后立即申明Errors或者BindingResult|
|SessionStatus + 类级别的 @SessionAttributes||
    



