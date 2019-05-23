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
|SessionStatus + 类级别的 @SessionAttributes|用来标记form表单处理完成，触发在类级别@SessionAttributes注解定义的session属性清除。参见@SessionAttributes|
|UriComponentsBuilder|用来准备相对于当前请求host，port，schema，上下文路径以及servlet映射的字面部分的URL。参见URI Links|
|@SessionAttribute|用来访问任何session属性，与模型属性不同，模型属性存储在session中，使用类级别的@SessionAttributes申明。参见@SessionAttributes|
|@RequestAttribute|用来访问request属性，参见@RequestAttribute|
|其它参数|如果方法参数没有匹配到上面提到值，并且它是基本类型（由[BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.7.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)定义），它会被处理为@RequestParam，否则会被处理为@ModelAttribute|

### 返回值

下面表格描述了支持的Controller方法返回值，所有返回值都支持reactive类型

| Controller方法返回值 | 描述 |
| --- | --- |
|@ResponseBody|返回值通过HttpMessageConverter来转换，并写入response，参见@ResponseBody.|
|HttpEntity<B>, ResponseEntity<B>|返回值指定了完整的response（包括http头和体），通过HttpMessageConverter来转换，并写入response，参见@ResponseBody.|
|HttpHeaders|用来返回response的header，没有body|
|String|view名称，用ViewResolver来解析，和隐式model同时使用，model通过命令对象和@ModelAttribute方法来决定。处理方法也可以通过程序申明model参数来加强model（参见Explicit Registrations）|
|View|要使用的，用来和隐式模型一起渲染的View实例。隐式模型通过命令对象和@ModelAttribute方法来决定。处理方法也可以通过程序申明model参数来加强model（参见Explicit Registrations|
|java.util.Map, org.springframework.ui.Model|添加到隐式模型上的属性，view名称通过RequestToViewNameTranslator隐式决定|
|@ModelAttribute|添加到模型上的一个属性，模型名称通过RequestToViewNameTranslator隐式决定。 注意 @ModelAttribute是可选的。参见其它任何返回值|
|ModelAndView对象|要使用的视图和模型属性，以及响应状态（可选择）|
|void|方法的返回值为void类型（或者null类型）被认为已经完全处理的响应，如果该方法有ServletResponse和OutputStream参数或者@ResponseStatus注解。如果controller已经做了ETag或者lastModified时间戳检查也是正确的（参见Controllers）。如果上述都不是，void返回类型也在REST controller中表示没有响应对象或html controller的默认视图名称选择|
|DeferredResult<V>|异步从任一线程中生成前面的任何返回值——例如，作为事件或回掉的结果，参见Asynchronous Requests和DeferredResult|
|Callable<V>|异步在spring-mvc管理的线程中生成上面的返回值。参见Asynchronous Requests和Callable|
|ListenableFuture<V>, java.util.concurrent.CompletionStage<V>, java.util.concurrent.CompletableFuture<V>|作为便利的可选择的DeferredResult（例如，当前服务返回它们中的一个时）|
|ResponseBodyEmitter, SseEmitter|异步生成对象流，使用HttpMessageConverter的实现写到response中，也支持作为ResponseEntity的body。参见 Asynchronous Requests和HTTP Streaming|
|StreamingResponseBody|异步写到response的OutputStream中，也支持作为ResponseEntity的body。参见 Asynchronous Requests和HTTP Streaming|
|响应类型— Reactor, RxJava, 或者其它通过ReactiveAdapterRegistry注册的类型|可选择的DeferredResult（例如Flux, Observable），多个值的流都被收集到List中。对于流式场景（例如text/event-stream, application/json+stream），SseEmitter和 ResponseBodyEmitter被替换使用,ServletOutputStream阻塞I/O在spring mvc管理的线程上执行，背压应用于每一次写完成。参见Asynchronous Requests和Reactive Types.|
|其它返回值|其它不在该表格中列出的返回值，如果是String或者void，被认为是视图名（默认视图通过RequestToViewNameTranslator来决定），假如不是基础类型（由BeanUtils#isSimpleProperty决定）。基础类型的值不被处理|

#### 类型转换

某些注解的controller方法参数代表了基于String类型的请求输入（例如@RequestParam, @RequestHeader, @PathVariable, @MatrixVariable, and @CookieValue)，如果参数定义成其它非String类型的，能够进行类型转换。
对于这种情况，类型转换基于配置的converter自动应用。默认的，简单类型（int, long, Date以及其它）都支持。你可以通过WebDataBinder定制转换（参见DataBinder）或者通过使用FormattingConversionService注册的Formatters，参见Spring字段格式化.

#### Matrix变量

RFC 3986讨论了在路径部分的名值对。在Spring MVC中，我们提到Matrix变量是基于Tim Berners-Lee的“old post”，但它们也可以被指定为URI路径参数。

Matrix变量能在任意路径部分中出现，每一个变量由分号分隔，多个值由逗号分隔（如：/cars;color=red,green;year=2012）。多个值也可以通过重复的变量名来指定（如：color=red;color=green;color=blue）

如果URL要包含Matrix变量，对于一个contrller方法的请求映射必须使用URI变量去遮住那个变量内容，确保请求成功匹配，而不依赖于Matrix变量的顺序和出现。以下的例子展示了Matrix变量的使用：

```java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {
    // petId == 42
    // q == 11
}
```

被给定的路径部分可能包含Matrix变量时，你有时需要区分那个路径变量需要填入matrix变量中。如下的例子展示了如何做的。

```java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
    @MatrixVariable(name="q", pathVar="ownerId") int q1,
    @MatrixVariable(name="q", pathVar="petId") int q2) {
    // q1 == 11
    // q2 == 22
}
```

Matrix变量可以定义为可选以及有默认值，如下的例子所展示的

```java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {
    // q == 1
}
```

如果需要获取到所有的Matrix变量，你可以使用MultiValueMap

```java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```

需要注意的是，你需要让matrix变量可用。在MVC java配置中，你需要通过Path Matching设置UrlPathHelper以及removeSemicolonContent=false。在mvc xml配置中，你可以设置

```xml
<mvc:annotation-driven enable-matrix-variables="true"/>
```

#### @RequestParam

你可以使用@RequestParam注解绑定Servlet请求参数（也就是查询参数或者表单数据）到contrller的方法参数。

如下例子展示如何使用

```java
@Controller
@RequestMapping("/pets")
public class EditPetForm {
    // ...
    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { 
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }
    // ...
}
```

默认情况下，使用该注解的方法参数是必须的，但你可以通过设置@RequestParam注解的required标志位为false来指定方法参数是可选，或者使用java.util.Optional包裹方法参数

如果目标方法参数类型不是String，那么类型转换会自动应用。参见类型转换

申明方法参数是数组或List允许同样的参数名有多个参数值。

当@RequestParam注解申明为Map<String, String> or MultiValueMap<String, String>且没有参数名指定时，map会自动填充为参数名和参数值

注意，使用@RequestParam是可选的（如设置它的属性）。默认情况下，任意简单类型参数（由BeanUtils#isSimpleProperty决定）且没有被任意其它参数解析器所解析的，都会被认为加了@RequestParam注解。

#### @RequestHeader

可以使用@RequestHeader注解绑定请求头到controller的方法参数。

考虑如下请求头

```
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
```

如下的例子获取来Accept-Encoding和Keep-Alive头

```java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
```

如果目标方法参数类型不是String，会自动进行类型转换。参见类型转换

当@RequestHeader注解使用在Map<String, String>, MultiValueMap<String, String>,或者HttpHeaders参数上时，map会填充所有header值。

> 内建支持将逗号分隔的字符串转换为数组或字符串集合或者其它类型转换系统知道的类型。例如：注解为@RequestHeader("Accept")的方法参数类型可以是String，也可以是String[]或者List\<String>

