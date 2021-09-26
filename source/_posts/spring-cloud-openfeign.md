Feign是一个声明性的Web服务客户端。它使编写Web服务客户端变得更容易。要使用Feign，请创建一个接口并对其进行注释。它具有可插入的注释支持，包括Feign注释和JAX-RS注释。 Feign还支持可插拔编码器和解码器。Spring Cloud增加了对Spring MVC注释的支持，并使用了Spring Web中默认使用的相同HttpMessageConverters。 Spring Cloud集成了Ribbon和Eureka，在使用Feign时提供负载均衡的http客户端。

## 引入依赖

1. group:org.springframework.cloud
2. artifact:spring-cloud-starter-openfeign

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

客户端调用

```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

@FeignClient的值是服务的名称,主要用来负载均衡.当然也可以用url属性来指定具体的地址.该接口在上下文中注册的bean名称是完全限定名称,你可以使用qualifier属性来指定别名.

上面的Ribbon客户端将要发现“strore”服务的物理地址。如果您的应用程序是Eureka客户端，那么它将解析Eureka服务注册表中的服务。如果您不想使用Eureka，则只需在外部配置中配置服务器列表即可.

## 修改默认配置

feign的核心概念是命名的客户端,每个客户端是一组链接远程服务的组件,开发人员通过@FeignClient注解指定这组组件的名称.

Spring Cloud使用FeignClientsConfiguration按需为每个命名客户端创建一个新的集合作为ApplicationContext.这包含（除其他外）feign.Decoder，feign.Encoder和feign.Contract。 可以使用@FeignClient批注的contextId属性覆盖该集合的名称。

Spring Cloud允许您通过使用@FeignClient声明其他配置（在FeignClientsConfiguration之上）来完全控制feign客户端。 例：

```java
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

在这种情况下，客户端由FeignClientsConfiguration中已有的组件以及FooConfiguration中的默认组件组成（后者将覆盖前者）。

> FooConfiguration类不需要被@Configuration标注.注意不要配置在@ComponentScan指定的包范围,否则会成为全局默认配置.

serviceId属性已经被name属性取代

> 除了更改ApplicationContext集合的名称之外，使用@FeignClient批注的contextId属性，它还会覆盖客户端名称的别名，并将其用作为该客户端创建的配置bean名称的一部分。

> 以前，使用url属性不需要name属性。 现在需要使用name。

name和url属性支持占位符:

```java
@FeignClient(name = "${feign.name}", url = "${feign.url}")
public interface StoreClient {    //.. }
```

Spring Cloud Netflix默认为feign（BeanType beanName：ClassName）提供以下bean：

* `Decoder` feignDecoder: `ResponseEntityDecoder` (which wraps a `SpringDecoder`)
* `Encoder` feignEncoder: `SpringEncoder`
* `Logger` feignLogger: `Slf4jLogger`
* `MicrometerCapability` micrometerCapability: If `feign-micrometer` is on the classpath and `MeterRegistry` is available
* `Contract` feignContract: `SpringMvcContract`
* `Feign.Builder` feignBuilder: `FeignCircuitBreaker.Builder`
* `Client` feignClient:如果 Spring Cloud LoadBalancer 在类路径上，则使用 FeignBlockingLoadBalancerClient。 如果它们都不在类路径上，则使用默认的 feign 客户端。

> spring-cloud-starter-openfeign 支持 spring-cloud-starter-loadbalancer。 但是，作为一个可选的依赖项，如果您想使用它，您需要确保将其添加到您的项目中。

可以通过将feign.okhttp.enabled或feign.httpclient.enabled分别设置为true并将依赖放在类路径上来使用OkHttpClient和ApacheHttpClient假装客户端。 您可以在使用OK时使用OkHttpClient或者Apache是使用ClosableHttpClient的bean来自定义所使用的HTTP客户端。

Spring Cloud OpenFeign 默认情况下不为feign提供以下bean，但仍然从应用程序上下文中查找这些类型的bean以创建feign客户端：

- `Logger.Level`
- `Retryer`
- `ErrorDecoder`
- `Request.Options`
- `Collection<RequestInterceptor>`
- `SetterFactory`
- `QueryMapEncoder`
- `Capability` (`MicrometerCapability` is provided by default)

默认情况下会创建 Retryer.NEVER_RETRY 类型为 Retryer 的 bean，这将禁用重试。 请注意，这种重试行为与 Feign 默认行为不同，它会自动重试 IOExceptions，将它们视为与网络相关的瞬态异常，以及从 ErrorDecoder 抛出的任何 RetryableException。

创建其中一种类型的 bean 并将其放置在 @FeignClient 配置中（例如上面的 FooConfiguration）允许您覆盖所描述的每个 bean。 例子：

```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

这将 SpringMvcContract 替换为 feign.Contract.Default 并将 RequestInterceptor 添加到 RequestInterceptor 的集合中。

@FeignClient 也可以使用配置属性进行配置:

```yaml
feign:
    client:
        config:
            feignName:
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: full
                errorDecoder: com.example.SimpleErrorDecoder
                retryer: com.example.SimpleRetryer
                defaultQueryParameters:
                    query: queryValue
                defaultRequestHeaders:
                    header: headerValue
                requestInterceptors:
                    - com.example.FooRequestInterceptor
                    - com.example.BarRequestInterceptor
                decode404: false
                encoder: com.example.SimpleEncoder
                decoder: com.example.SimpleDecoder
                contract: com.example.SimpleContract
                capabilities:
                    - com.example.FooCapability
                    - com.example.BarCapability
                metrics.enabled: false
```

可以以与上述类似的方式在 @EnableFeignClients 属性 defaultConfiguration 中指定默认配置。 不同之处在于此配置将适用于所有 feign 客户端。

如果您更喜欢使用配置属性来配置所有 @FeignClient，您可以使用 default 名称创建配置属性。

```yaml
feign:
    client:
        config:
            default:
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: basic
```

您可以使用 feign.client.config.feignName.defaultQueryParameters 和 feign.client.config.feignName.defaultRequestHeaders 来发送默认请求参数和请求头。

如果我们同时创建@Configuration bean 和配置属性，配置属性将获胜。 它将覆盖@Configuration 值。 但是如果你想把优先级改成@Configuration，你可以把feign.client.default-to-properties改成false。

如果我们想创建多个具有相同名称或 url 的 feign 客户端，以便它们指向同一服务器但每个具有不同的自定义配置，那么我们必须使用 @FeignClient 的 contextId 属性以避免这些配置的bean名称冲突。

```java
@FeignClient(contextId = "fooClient", name = "stores", configuration = FooConfiguration.class)
public interface FooClient {
    //..
}
```

```java
@FeignClient(contextId = "barClient", name = "stores", configuration = BarConfiguration.class)
public interface BarClient {
    //..
}
```

也可以将 FeignClient 配置为不从父上下文继承 bean。 您可以通过覆盖 FeignClientConfigurer bean 中的 inheritParentConfiguration() 以返回 false 来实现此目的：

```java
@Configuration
public class CustomConfiguration{

@Bean
public FeignClientConfigurer feignClientConfigurer() {
            return new FeignClientConfigurer() {

                @Override
                public boolean inheritParentConfiguration() {
                    return false;
                }
            };

        }
}
```

在我们提供的 SpringEncoder 中，我们为二进制内容类型设置空字符集，为所有其他类型设置 UTF-8。

您可以修改此行为以通过将 feign.encoder.charset-from-content-type 的值设置为 true 来从 Content-Type 标头字符集派生字符集。

## 超时设置

我们可以在默认客户端和具名客户端上配置超时。 OpenFeign 使用两个超时参数：

* connectTimeout 可防止由于服务器处理时间过长而阻塞调用者。
* readTimeout 从连接建立时开始应用，并在返回响应时间过长时触发。

## Feign Builder API 创建客户端

下面是一个示例，它创建了两个具有相同接口的 Feign 客户端，但为每个客户端配置了一个单独的请求拦截器。

```java
@Import(FeignClientsConfiguration.class)
class FooController {

    private FooClient fooClient;

    private FooClient adminClient;

    @Autowired
    public FooController(Client client, Encoder encoder, Decoder decoder, Contract contract, MicrometerCapability micrometerCapability) {
        this.fooClient = Feign.builder().client(client)
                .encoder(encoder)
                .decoder(decoder)
                .contract(contract)
                .addCapability(micrometerCapability)
                .requestInterceptor(new BasicAuthRequestInterceptor("user", "user"))
                .target(FooClient.class, "https://PROD-SVC");

        this.adminClient = Feign.builder().client(client)
                .encoder(encoder)
                .decoder(decoder)
                .contract(contract)
                .addCapability(micrometerCapability)
                .requestInterceptor(new BasicAuthRequestInterceptor("admin", "admin"))
                .target(FooClient.class, "https://PROD-SVC");
    }
}
```

上面例子中的 FeignClientsConfiguration.class 是 Spring Cloud OpenFeign 提供的默认配置。

Feign Contract 对象定义了接口上哪些注解和值是有效的。 自动装配的 Contract bean 提供对 SpringMVC 注释的支持，而不是默认的 Feign 原生注释。

您还可以使用 Builder` 将 FeignClient 配置为不从父上下文继承 bean。 您可以通过在 Builder 上覆盖调用 `inheritParentContext(false) 来做到这一点。

## 断路器支持

如果 Spring Cloud CircuitBreaker 在类路径上并且 feign.circuitbreaker.enabled=true，Feign 将使用断路器包装所有方法。

要在每个客户端的基础上禁用 Spring Cloud CircuitBreaker 支持，请创建一个具有“prototype”范围的 vanilla Feign.Builder，例如：

```java
@Configuration
public class FooConfiguration {
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
}
```

断路器名称遵循此模式 `<feignClientClassName>#<CalledMethod>(<parameterTypes>)`。 例如调用上面的FooClient的bar方法，断路器名称将是 FooClient#bar()。

提供 CircuitBreakerNameResolver 的 bean，您可以更改断路器名称的模式。

```java
@Configuration
public class FooConfiguration {
    @Bean
    public CircuitBreakerNameResolver circuitBreakerNameResolver() {
        return (String feignClientName, Target<?> target, Method method) -> feignClientName + "_" + method.getName();
    }
}
```

要启用 Spring Cloud CircuitBreaker 组，请将 `feign.circuitbreaker.group.enabled` 属性设置为 true（默认为 false）。

## 失败回退

Spring Cloud CircuitBreaker 支持回退的概念：当电路打开或出现错误时执行默认代码路径。 要为给定的 @FeignClient 启用回退，请将`fallback`属性设置为实现回退的类名。 您还需要将您的实现声明为 Spring bean。

```java
@FeignClient(name = "test", url = "http://localhost:${server.port}/", fallback = Fallback.class)
protected interface TestClient {

        @RequestMapping(method = RequestMethod.GET, value = "/hello")
        Hello getHello();

        @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
        String getException();

}

@Component
static class Fallback implements TestClient {

        @Override
        public Hello getHello() {
            throw new NoFallbackAvailableException("Boom!", new RuntimeException());
        }

        @Override
        public String getException() {
            return "Fixed response";
        }

}
```

如果需要访问触发回退的原因，可以使用 @FeignClient 中的 fallbackFactory 属性。

```java
@FeignClient(name = "testClientWithFactory", url = "http://localhost:${server.port}/",
            fallbackFactory = TestFallbackFactory.class)
    protected interface TestClientWithFactory {

        @RequestMapping(method = RequestMethod.GET, value = "/hello")
        Hello getHello();

        @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
        String getException();

    }

    @Component
    static class TestFallbackFactory implements FallbackFactory<FallbackWithFactory> {

        @Override
        public FallbackWithFactory create(Throwable cause) {
            return new FallbackWithFactory();
        }

    }

    static class FallbackWithFactory implements TestClientWithFactory {

        @Override
        public Hello getHello() {
            throw new NoFallbackAvailableException("Boom!", new RuntimeException());
        }

        @Override
        public String getException() {
            return "Fixed response";
        }

    }
```



将 Feign 与 Spring Cloud CircuitBreaker 回退一起使用时，ApplicationContext 中有多个相同类型的 bean。 这将导致 @Autowired 无法工作，因为没有一个 bean，或者一个标记为primary的 bean。 为了解决这个问题，Spring Cloud OpenFeign 将所有 Feign 实例标记为 @Primary，因此 Spring Framework 知道要注入哪个 bean。 在某些情况下，这可能是不可取的。 要关闭此行为，请将 @FeignClient 的`primary`属性设置为 false。

```java
@FeignClient(name = "hello", primary = false)
public interface HelloClient {
    // methods here
}
```

## Feign 继承支持

Feign 通过单继承接口支持样板 API。 这允许将常见操作分组到方便的基本接口中。

```java
public interface UserService {
    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```

```java
@RestController
public class UserResource implements UserService {

}
```

```java
package project.user;

@FeignClient("users")
public interface UserClient extends UserService {

}
```

> 通常不建议在服务器和客户端之间共享一个接口。 它引入了紧耦合，也不是所有维护的 Spring MVC 版本都支持（某些版本没有继承方法参数映射）。

## 请求和响应压缩

您可以考虑为您的 Feign 请求启用请求或响应 GZIP 压缩。 您可以通过启用以下属性之一来执行此操作：

```java
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```

Feign 请求压缩为您提供类似于您为 Web 服务器设置的设置：

```java
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

这些属性允许您选择压缩媒体类型和最小请求阈值长度。

对于除了 OkHttpClient 之外的 http 客户端，可以启用默认的 gzip 解码器来解码 UTF-8 编码的 gzip 响应：

```java
feign.compression.response.enabled=true
feign.compression.response.useGzipDecoder=true
```

## 日志

为每个创建的 Feign 客户端创建一个logger。 默认情况下，logger的名称是用于创建 Feign 客户端的接口的完整类名。 Feign logging 只响应 DEBUG 级别。

```yaml
logging.level.project.user.UserClient: DEBUG
```

* NONE: 默认级别
* BASIC：仅记录请求方法和 URL 以及响应状态代码和执行时间。
* HEADERS：记录基本信息以及请求和响应标头。
* FULL：记录请求和响应的标头、正文和元数据。

例如，以下内容会将 Logger.Level 设置为 FULL：

```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

## @QueryMap支持

OpenFeign @QueryMap 注释支持将 POJO 用作 GET 参数映射。不幸的是，默认的 OpenFeign QueryMap 注释与 Spring 不兼容，因为它缺少 value 属性。Spring Cloud OpenFeign 提供了一个等效的 @SpringQueryMap 注解，用于将 POJO 或 Map 参数注解为查询参数映射。

```java
// Params.java
public class Params {
    private String param1;
    private String param2;

    // [Getters and setters omitted for brevity]
}
```

```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/demo")
    String demoEndpoint(@SpringQueryMap Params params);
}
```

如果您需要对生成的查询参数映射进行更多控制，则可以实现自定义 QueryMapEncoder bean。

## CollectionFormat 支持

我们通过提供@CollectionFormat 注释来支持 feign.CollectionFormat。您可以通过传递所需的 feign.CollectionFormat 作为注释值来注释 Feign 客户端方法。

在以下示例中，使用 CSV 格式而不是默认的 EXPLODED 来处理方法。

```java
@FeignClient(name = "demo")
    protected interface PageableFeignClient {

        @CollectionFormat(feign.CollectionFormat.CSV)
        @GetMapping(path = "/page")
        ResponseEntity performRequest(Pageable page);

    }
```

在发送 Pageable 作为查询参数时设置 CSV 格式，以便正确编码。？？？

## spring data支持

您可以考虑启用 Jackson Modules 以支持 org.springframework.data.domain.Page 和 org.springframework.data.domain.Sort 解码。

```java
feign.autoconfiguration.jackson.enabled=true
```

## RefreshScope支持？？？

如果启用了 Feign 客户端刷新，则使用 feign.Request.Options 作为刷新范围的 bean 创建每个 feign 客户端。 这意味着可以通过 POST /actuator/refresh 针对任何 Feign 客户端实例刷新 connectTimeout 和 readTimeout 等属性。

默认情况下，Feign 客户端中的刷新行为是禁用的。 使用以下属性启用刷新行为：

```java
feign.client.refresh-enabled=true
```

## MatrixVariable 支持

Spring Cloud OpenFeign 提供对 Spring @MatrixVariable 注解的支持。

如果map作为方法参数传递，@MatrixVariable 路径段是通过使用 = 连接map中的键值对来创建的。

??? 没有翻译完

## capabilities 功能

Feign 功能公开了核心 Feign 组件，以便可以修改这些组件。 例如，功能可以获取客户端，对其进行装饰，并将装饰后的实例返回给 Feign。 对指标库的支持是一个很好的现实例子。 请参阅 Feign 指标。

创建一个或多个 Capability bean 并将它们放置在 @FeignClient 配置中，让您可以注册它们并修改相关客户端的行为。

```java
@Configuration
public class FooConfiguration {
    @Bean
    Capability customCapability() {
        return new CustomCapability();
    }
}
```

## Feign 指标

如果以下所有条件都为真，则会创建并注册 MicrometerCapability bean，以便您的 Feign 客户端将指标发布到 Micrometer：

* feign-micrometer 在类路径上

* MeterRegistry bean 可用

* feign 指标属性设置为 true（默认情况下）

  * `feign.metrics.enabled=true` (for all clients)
  * `feign.client.config.feignName.metrics.enabled=true` (for a single client)

  

您还可以通过注册自己的 bean 来自定义 MicrometerCapability：

```java
@Configuration
public class FooConfiguration {
    @Bean
    public MicrometerCapability micrometerCapability(MeterRegistry meterRegistry) {
        return new MicrometerCapability(meterRegistry);
    }
}
```

