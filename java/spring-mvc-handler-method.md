# Spring MVC处理方法

这两天发现对spring mvc处理方法注解还是一知半解，所以今天抽空写一下这方面的文章。其实就是对[Spring MVC Handler Method](https://docs.spring.io/spring/docs/5.1.7.RELEASE/spring-framework-reference/web.html#mvc-ann-methods)及相关章节的大致翻译。

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
|@PathVariable|用来访问模板变量，参见[URI模式](#URI模式)|
|@MatrixVariable|用来访问URI路径部分的键值对，参见[Matrix变量](#matrix变量)|
|@RequestParam|用来访问Servlet request参数，包括multipart files。请求值被转成申明的方法参数类型，参见[@RequestParam](#requestparam)和[Multipart](#multipart).|
|@RequestHeader|用来访问request头。值被转换为申明的方法参数类型。参见[@RequestHeader](#requestheader).|
|@CookieValue|用来访问Cookie。Cookie值被转换为申明的方法参数类型。参见[@CookieValue](#cookievalue)|
|@RequestBody|用来访问HTTP request body。Body内容通过HttpMessageConverter的实现类被转换为申明的方法参数类型，参见[@RequestBody](#requestbody)|
|HttpEntity<B>|用来访问request headers和body。body通过HttpMessageConverter来转换。参见[HttpEntity](#httpentity)|
|@RequestPart|用来访问multipart/form-data的part，使用HttpMessageConverter来转换part的body。参见[Multipart](#multipart).|
|java.util.Map, org.springframework.ui.Model, org.springframework.ui.ModelMap|用来访问模型。该模型在html控制器中使用，且作为视图渲染的一部分暴露到模板中|
|RedirectAttributes|指定了在redirect情况下使用的属性（被添加到查询字符串后）以及flash属性，会临时保存到redirect后的request。参见[Redirect Attributes](#redirectattributes)和[Flash Attributes](#flashattributes)|
|@ModelAttribute|用来访问在model中已经存在的属性（不存在则实例化）且已经应用过data binding和validation。参见[@ModelAttribute](#modelattribute)以及[Model](#model)和[DataBinder](#databinder).|
|Errors, BindingResult|用来访问命令对象（@ModelAttribute修饰的参数）在validation和data binding中的错误，或者@RequestBody或@RequestPart修饰的参数的validation错误。你必须在需要验证的方法参数后立即申明Errors或者BindingResult|
|SessionStatus + 类级别的 @SessionAttributes|用来标记form表单处理完成，触发在类级别@SessionAttributes注解定义的session属性清除。参见[@SessionAttributes](#sessionattributes)|
|UriComponentsBuilder|用来准备相对于当前请求host，port，schema，上下文路径以及servlet映射的字面部分的URL。参见[URI Links](#urilinks)|
|@SessionAttribute|用来访问任何session属性，与模型属性不同，模型属性存储在session中，使用类级别的@SessionAttributes申明。参见[@SessionAttributes](#sessionattributes)|
|@RequestAttribute|用来访问request属性，参见[@RequestAttribute](#requestattribute)|
|其它参数|如果方法参数没有匹配到上面提到值，并且它是基本类型（由[BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.7.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)定义），它会被处理为@RequestParam，否则会被处理为@ModelAttribute|

### 返回值

下面表格描述了支持的Controller方法返回值，所有返回值都支持reactive类型

| Controller方法返回值 | 描述 |
| --- | --- |
|@ResponseBody|返回值通过HttpMessageConverter来转换，并写入response，参见@ResponseBody.|
|HttpEntity<B>, ResponseEntity<B>|返回值指定了完整的response（包括http头和体），通过HttpMessageConverter来转换，并写入response，参见[@ResponseBody](#responsebody).|
|HttpHeaders|用来返回response的header，没有body|
|String|view名称，用ViewResolver来解析，和隐式model同时使用，model通过命令对象和@ModelAttribute方法来决定。处理方法也可以通过程序申明model参数来加强model（参见[Explicit Registrations](#explicitregistrations)）|
|View|要使用的，用来和隐式模型一起渲染的View实例。隐式模型通过命令对象和@ModelAttribute方法来决定。处理方法也可以通过程序申明model参数来加强model（参见[Explicit Registrations](#explicitregistrations)|
|java.util.Map, org.springframework.ui.Model|添加到隐式模型上的属性，view名称通过RequestToViewNameTranslator隐式决定|
|@ModelAttribute|添加到模型上的一个属性，模型名称通过RequestToViewNameTranslator隐式决定。 注意 @ModelAttribute是可选的。参见其它任何返回值|
|ModelAndView对象|要使用的视图和模型属性，以及响应状态（可选择）|
|void|方法的返回值为void类型（或者null类型）被认为已经完全处理的响应，如果该方法有ServletResponse和OutputStream参数或者@ResponseStatus注解。如果controller已经做了ETag或者lastModified时间戳检查也是正确的（参见[Controllers](#controllers)）。如果上述都不是，void返回类型也在REST controller中表示没有响应对象或html controller的默认视图名称选择|
|DeferredResult<V>|异步从任一线程中生成前面的任何返回值——例如，作为事件或回掉的结果，参见[异步请求](#异步请求)和[DeferredResult](#deferredresult)|
|Callable<V>|异步在spring-mvc管理的线程中生成上面的返回值。参见[异步请求](#异步请求)和[Callable](#callable)|
|ListenableFuture<V>, java.util.concurrent.CompletionStage<V>, java.util.concurrent.CompletableFuture<V>|作为便利的可选择的DeferredResult（例如，当前服务返回它们中的一个时）|
|ResponseBodyEmitter, SseEmitter|异步生成对象流，使用HttpMessageConverter的实现写到response中，也支持作为ResponseEntity的body。参见 [异步请求](#异步请求)和[HTTP流](#HTTP流)|
|StreamingResponseBody|异步写到response的OutputStream中，也支持作为ResponseEntity的body。参见[异步请求](#异步请求)和[HTTP流](#HTTP流)|
|响应类型— Reactor, RxJava, 或者其它通过ReactiveAdapterRegistry注册的类型|可选择的DeferredResult（例如Flux, Observable），多个值的流都被收集到List中。对于流式场景（例如text/event-stream, application/json+stream），SseEmitter和 ResponseBodyEmitter被替换使用,ServletOutputStream阻塞I/O在spring mvc管理的线程上执行，背压应用于每一次写完成。参见[异步请求](#异步请求)和[响应类型](#响应类型).|
|其它返回值|其它不在该表格中列出的返回值，如果是String或者void，被认为是视图名（默认视图通过RequestToViewNameTranslator来决定），假如不是基础类型（由BeanUtils#isSimpleProperty决定）。基础类型的值不被处理|

### 类型转换

某些注解的controller方法参数代表了基于String类型的请求输入（例如@RequestParam, @RequestHeader, @PathVariable, @MatrixVariable, and @CookieValue)，如果参数定义成其它非String类型的，能够进行类型转换。
对于这种情况，类型转换基于配置的converter自动应用。默认的，简单类型（int, long, Date以及其它）都支持。你可以通过WebDataBinder定制转换（参见DataBinder）或者通过使用FormattingConversionService注册的Formatters，参见[Spring字段格式化](#spring字段格式化).

### Matrix变量

RFC 3986讨论了在路径部分的名值对。在Spring MVC中，我们提到Matrix变量是基于Tim Berners-Lee的“old post”，但它们也可以被指定为URI路径参数。

Matrix变量能在任意路径部分中出现，每一个变量由分号分隔，多个值由逗号分隔（如：/cars;color=red,green;year=2012）。多个值也可以通过重复的变量名来指定（如：color=red;color=green;color=blue）

如果URL要包含Matrix变量，对于一个contrller方法，请求映射必须通过URI变量来包含变量内容，确保请求成功匹配，而不依赖于Matrix变量的顺序和出现。以下的例子展示了Matrix变量的使用：

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

需要注意的是，你需要让matrix变量可用。在MVC java配置中，你需要通过[路径匹配](#路径匹配)设置UrlPathHelper以及removeSemicolonContent=false。在mvc xml配置中，你可以设置

```xml
<mvc:annotation-driven enable-matrix-variables="true"/>
```

### @RequestParam

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

如果目标方法参数类型不是String，那么类型转换会自动应用。参见[类型转换](#类型转换)

申明方法参数是数组或List允许同样的参数名有多个参数值。

当@RequestParam注解申明为Map<String, String> or MultiValueMap<String, String>且没有参数名指定时，map会自动填充为参数名和参数值

注意，使用@RequestParam是可选的（如设置它的属性）。默认情况下，任意简单类型参数（由BeanUtils#isSimpleProperty决定）且没有被任意其它参数解析器所解析的，都会被认为加了@RequestParam注解。

### @RequestHeader

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

如果目标方法参数类型不是String，会自动进行类型转换。参见[类型转换](#类型转换)

当@RequestHeader注解使用在Map<String, String>, MultiValueMap<String, String>,或者HttpHeaders参数上时，map会填充所有header值。

> 内建支持将逗号分隔的字符串转换为数组或字符串集合或者其它类型转换系统知道的类型。例如：注解为@RequestHeader("Accept")的方法参数类型可以是String，也可以是String[]或者List\<String>

### @CookieValue

使用@CookieValue注解可以绑定http cookie到controller的方法参数。

考虑有如下cookie的请求

```
SESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
```

下面的例子展示如何获取到cookie值

```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
```

如果目标方法参数不是string，会自动使用类型转换。参见[类型转换](#类型转换)

### @ModelAttribute

方法参数上使用@ModelAttribute来访问model的属性，或者如果不存在，则初始化。model的属性也会覆盖http servlet的请求参数中名称和字段名称匹配的值。这个可以参考数据绑定，它可以帮助你处理解析和转换单独的参数和form字段。下面的例子展示了如何操作。

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { }
```

Pet实例按照如下步骤解析
* 如果已经通过Model添加了，则使用[model](#model)解析
* 通过使用[@SessionAttributes](#sessionattributes)从Http session中解析
* 通过Converter从URI路径变量中解析（参见下面的例子）
* 通过调用默认构造器解析
* 通过调用参数与Servlet请求参数匹配的主构造器解析。参数名称通过@ConstructorProperties的javabean获取或者通过二进制码中运行时保留的参数解析

通常是使用[model](#model)来填充属性，另外一种可选择的方法是依赖Converter<String, T>和URI路径变量转换器的组合。下面的例子中，模型属性名称account与URI路径变量account匹配，通过使用注册的Converter<String, Account>解析字符串account数字，Account类被自动加载。

```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```

在model属性实例加载完后，会使用数据绑定。WebDataBinder类将Servlet请求参数名称（请求参数和表单字段）和目标对象的字段名称匹配。匹配的字段在类型转换后被填充。数据绑定（以及验证）的更多信息，参见Validation。自定义数据绑定的更多信息，参见[DataBinder](#databinder)

数据绑定可能会出错。默认情况下，BindException异常被抛出。然而如果想在controller中检查这类错误，你可以在@ModelAttribute后添加一个BindingResult参数，如同下面例子展示的一样

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

在某些情况下，你想访问没有绑定的模型属性。此时你可以向controller中注入Model，直接访问它，或者，设置@ModelAttribute(binding=false)，如同下面的例子一样

```java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { 
    // ...
}
```

你可以在数据绑定完成后自动使用验证，通过添加javax.validation.Valid注解或者Spring的@Validated注解（bean验证和Spring验证）。下面的例子展示如何使用

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

注意，使用@ModelAttribute是可选的（例如，设置它的属性）。默认情况下，任何不是简单类型的参数（由BeanUtils#isSimpleProperty决定）且不会被其它参数解析器解析的参数会和加上@ModelAttribute效果一样。

### @SessionAttributes

@SessionAttributes用来存储两个请求间的http servlet中的session模型属性。它是一个类型级别的注解，定义了特别的controller的session属性。通常列出模型属性的名称或者模型属性的类型，它们应该被透明地存储到session中来被后续request访问

如下例子使用@SessionAttributes注解

```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {
    // ...
}
```

第一个请求中，名称为pet的模型属性被添加到模型中时，它会自动地保存到http servlet session中。它会一直保存直到其它的controller方法使用SessionStatus方法参数清空了存储，就像下面的例子中展示的

```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors) {
            // ...
        }
            status.setComplete(); 
            // ...
        }
    }
}
```

### @SessionAttribute

如果你需要访问之前存在的全局的session属性（在controller之外，例如通过filter），它可能存在，也可能不存在，你可以在方法参数上使用@SessionAttribute注解，像下面的例子一样

```java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { 
    // ...
}
```

对于需要增加或移除session变量的情况，考虑注入org.springframework.web.context.request.WebRequest或javax.servlet.http.HttpSession到controller方法。

对于将模型属性临时存储到session中作为controller流程的一部分情况，考虑使用@SessionAttributes

### @RequestAttribute

类似于@SessionAttribute，你可以使用@RequestAttribute注解访问已经存在的请求的属性（例如，通过filter或者HandlerInterceptor创建的）

```java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { 
    // ...
}
```
    
### 重定向属性

默认情况下，在重定向URL里，所有模型属性都应该做为URI模板变量被暴露。其它基本属性或基础类型的集合或数组都作为查询参数自动添加。

如果模型需要重定向，那么作为参数添加的基本类型属性是期望看到的。然而，在注解的controller中，模型可以包含额外用于渲染目的的属性（例如：下拉字段属性）。为了避免在URL中包含这种属性，@RequestMapping方法可以申明RedirectAttributes类型的参数，用它来指定对RedirectView可用的属性。如果方法重定向，RedirectAttributes内容被用到。否则，模型内容被用到。

RequestMappingHandlerAdapter提供了一个叫做ignoreDefaultModelOnRedirect的标志位，可以使用它来指示当controller方法重定向时，默认模型的内容不被使用。相反地，controller方法也可以声明RedirectAttributes类型的属性，如果没有声明，没有属性会被传递给RedirectView。MVC命名空间和MVC Java配置默认设置这个标志位为false，以保持向后兼容。然而，对于新的应用，我们建议设置为true

需要注意的是当扩展重定向URL时，从当前请求发出的URI模板变量自动可用，你不需要显示地通过Model或RedirectAttributes添加它们。下面的例子展示了如何定义重定向。

```java
@PostMapping("/files/{path}")
public String upload(...) {
    // ...
    return "redirect:files/{path}";
}
```

另外一种到重定向目标的传递数据的方法是使用flash属性。不像其它重定向属性，flash属性保存到HTTP Session中（因此不在URL中出现）。参见[flash属性](#flash属性)。

### flash属性

flash属性提供了一种方法用来将一个请求的属性保存给另一个使用。这个是在重定向中最需要的。例如：Post-Redirect-Get模式。flash属性在重定向前会临时保存（通常是在session中）供重定向后的请求使用，并且会立即删除。

Spring MVC有两种flash属性的抽象支持。FlashMap用来保存flash属性，FlashMapManager用来保存，获取以及管理FlashMap实例。

flash属性不需要声明即可使用。然而，如果没有使用，它不会引起http session的创建。对于每个请求，有一个输入的FlashMap，用来保存上一个请求的属性（如果有），还有一个输出的FlashMap，用来保存下一个请求需要的属性。两个FlashMap实例在Spring MVC中任何位置都可以通过RequestContextUtils访问。

注解的controller通常不需要直接使用FlashMap。相反，@RequestMapping方法可以接受RedirectAttributes类型的属性，在重定向场景中可以使用它添加flash属性。通过RedirectAttributes添加的flash属性自动传播到输出的FlashMap。相似地，重定向后，输入的FlashMap的属性会自动添加到controller的Model中，供目标URL地址使用。

>  ***请求和flash属性的匹配***
> flash属性的概念在其他web框架中都存在，已经证明某些情况下会出现同步问题。这是因为，根据定义，flash属性在下一次请求时都会被存储。然而最近的下一次请求可能不是想要的而是其它异步请求（例如，轮询或者资源请求），这种情况下，flash属性会被过早移除。
> 
> 为了降低这种问题的出现，RedirectView自动使用路径和目标请求URL的查询参数记录》FlashMap实例。结果是，当请求查找输入的FlashMap时，默认的FlashMapManager就会匹配到。
> 
> 在已经包含信息的重定向URL中，这样做并不会完全消除同步引起的问题，但是会大大降低。因此，我们建议在重定向到场景中主要使用flash属性

### Multipart

当MultipartResolver启用时，使用multipart/form-data的POST请求内容会被解析，可以作为正常request请求参数被访问。下面的例子访问一个正常的form域和一个上传文件

```java
@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

将参数类型定义为List<MultipartFile>，则允许解析相同参数名称的多个文件。

当@RequestParam注解作为Map<String, MultipartFile>或者MultiValueMap<String, MultipartFile>定义，且没有在注解中指定参数名时，map被每一个给定参数名称的multipart文件填充。

> 在Servlet3.0 multipart解析时，你也可以将方法参数或者集合类型定义为javax.servlet.http.Part而不是Spring的Multipart

你也可以使用multipart内容当成数据的一部分绑定到命令对象中。例如，下面例子中的form字段和文件可以是表单对象的字段

```java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...
}

@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        if (!form.getFile().isEmpty()) {
            byte[] bytes = form.getFile().getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

Multipart请求也可以从非浏览器客户端用RESTful服务风格提交。下面的例子展示使用JSON格式的文件

```
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
    "name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...
```

你可以使用@RequestParam作为String格式访问meta-data，但你可能想要从JSON中解析（类似于@RequestBody）。在HttpMessageConverter转换之后可以使用@RequestPart注解访问multipart

```java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {
    // ...
}
```

你可以使用@RequestPart以及javax.validation.Valid或者Spring的@Validated注解，它们两个都可以启用标准bean验证。默认情况下，错误验证引起MethodArgumentNotValidException，转换为400（BAD_REQUEST）响应。你也可以使用Errors或者BindingResult在controller中处理验证错误。

```java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") MetaData metadata,
        BindingResult result) {
    // ...
}
```
 
### @RequestBody

你可以使用@RequestBody注解将request体读出并通过[HttpMessageConverter](#httpmessageconverter)序列化为对象。

```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```

可以使用MVC Config的[消息转换器](#消息转换器)来配置或自定义消息转换。

可以使用@RequestBody以及javax.validation.Valid或者Spring的@Validated注解，它们两个都可以启用标准bean验证。默认情况下，错误验证引起MethodArgumentNotValidException，转换为400（BAD_REQUEST）响应。你也可以使用Errors或者BindingResult在controller中处理验证错误。

```java
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Account account, BindingResult result) {
    // ...
}
```

### HttpEntity

HttpEntity和使用@RequestBody差不多，只是它是基于容器对象将request头和体暴露出来。

```java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
```

### @ResponseBody

你可以在方法上使用@ResponseBody注解将返回值通过HttpMessageConverter序列化为response体

```java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
```

@ResponseBody也可以使用在类级别，这种情况下，它被所有controller方法继承。这个就是@RestController的效果，而后者也仅仅是一个标识了@Controller和@ResponseBody的原注解。

可以在响应类型上使用@ResponseBody，参见[异步请求](#异步请求)和[响应类型](#响应类型)

可以使用MVC Config的消息转换器来配置或自定义消息转换。

可以使用@ResponseBody和JSON序列化视图，参见[Jackson JSON](#jackson_json)

### ResponseEntity

ResponseEntity类似于@ResponseBody，但包含状态和消息头。

```java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).build(body);
}
```

Spring MVC支持单个响应类型值来异步返回ResponseEntity，且/或使用单个和多个值的响应类型作为消息体。

### Jackson JSON

Spring提供对Jackson JSON库的支持。

#### JSON视图

Spring MVC提供内建的[Jackson序列化视图](#jackson序列化视图)支持，它允许渲染对象中所有字段的一个子集。为了使用@ResponseBody或者ResponseEntity controller方法，你可以使用Jackson的@JsonView注解来激活序列化的视图类

```java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```
 
> @JsonView支持视图类数组，但你可以在每个controller方法中指定一个。如果你需要激活多个视图，你可以使用组合接口

对于依赖于视图解析的controller来说，你可以添加序列化视图类到模型上。

```java
@Controller
public class UserController extends AbstractController {

    @GetMapping("/user")
    public String getUser(Model model) {
        model.addAttribute("user", new User("eric", "7!jd#h23"));
        model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
        return "userView";
    }
}
```

### 1.3.4 模型

你可以在如下情况下使用@ModelAttribute注解

* 在@RequestMapping方法的[方法参数](#modelattribute)上，用来创建或访问模型的对象，以及使用WebDataBinder来绑定到请求上。
* 在@Controller或者@ControllerAdvice类上作为方法级注解，用来在@RequestMapping方法调用前就初始化模型
* 在@RequestMapping方法上，用来标记方法返回值是模型属性

Controller可以有多个@ModelAttribute方法。在同一个controller中，所有这些方法都是在@RequestMapping方法调用前就被调用。@ModelAttribute方法可以通过@ControllerAdvice在不同controller间共用。参见[Controller Advice](#controller_advice)

@ModelAttribute方法有灵活的方法签名。它们支持和@RequestMapping一样的参数，除了@ModelAttribute以及任何与请求体相关的参数之外。

如下例子展示了@ModelAttribute方法

```java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
```

如下例子只添加一个属性

```java
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
```

> 当没有显示指定名称时，会基于Object类型选择一个默认名称，如同在[Conventions的java文档中](#conventions的java文档中)解释的一样。

@ModelAttribute也可以作为方法级注解在@RequestMapping方法上使用，这种情况下，@RequestMapping方法返回值作为模型属性返回。通常没必要这样做，因为这在html控制器中是默认行为，除非返回值是String，因为String会被解析为视图名称。@ModelAttribute也可以自定义模型属性名称，类似下面的例子

```java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```

### 数据绑定

@Controller或@ControllerAdvice可以包含@InitBinder方法，用来初始化WebDataBinder的实例，它们可以

* 绑定请求参数（form或查询数据）到模型对象
* 将基于String的请求值（请求参数、路径变量、header、cookie以及其它）转换为controller方法参数的目标类型
* 当渲染html表单时将模型对象格式化为String值

@InitBinder方法能注册特定controller的java.bean.PropertyEditor或者Spring Converter和Formatter组件。另外，你可以使用MVC配置来注册Converter和Formatter类型到FormattingConversionService中。

@InitBinder方法支持和@RequestMapping一样的参数，除了@ModelAttribute之外。通常，它们定义为WebDataBinder参数以及void返回值。下面是一个例子

```java
@Controller
public class FormController {

    @InitBinder 
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}
```

另外，当你通过FormattingConversionService注册Formatter时，你也可以使用相同的方法注册特定controller的Formatter实现，像下面的例子一样

```java
@Controller
public class FormController {

    @InitBinder 
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }

    // ...
}
```



