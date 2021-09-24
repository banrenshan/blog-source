`spring Cloud`构建在`Spring Boot`之上，提供了两个基础库:`Spring Cloud Context`和`Spring Cloud Commons`。

* `Spring Cloud Context`为`Spring Cloud`应用程序的ApplicationContext提供了实用程序和特殊服务，例如：引导上下文、加密、刷新范围和环境端点
* `Spring Cloud Commons`为不同的`Spring Cloud`实现提供抽象规范 ，例如`Spring Cloud Netflix`和`Spring Cloud Consul`

# Spring Cloud Context

`spring boot` 在spring基础上构建应用程序，例如约定俗成的文件配置规则、通用的端点管理和任务监控。`spring cloud`在此基础上新添加了一些基础组件。

## Bootstrap Application Context

spring cloud 启动时先创建bootstrap 上下文，然后创建 application 上下文。bootstrap会加载例如git等外部数据源上的配置信息，这些配置优先级比较高，无法被application配置的属性覆盖，但是bootstrap文件中配置的属性优先级比较低，会被application文件中同属性覆盖。

> * 一般情况下，我们会在bootstrap文件中配置`spring.application.name`和`spring.profiles.active`
> * 跟日志配置相关的属性，最好也放在这个文件中

下面是**bootstrap.yml**的示例文件：

```yaml
spring:
  application:
    name: foo
  cloud:
    config:
      uri: ${SPRING_CONFIG_URI:http://localhost:8888}
```

如果你使用`SpringApplication` 或`SpringApplicationBuilder`构建 app context，Bootstrap 会是该context的父级context。子级context继承父级context的属性。

你可以使用`spring.cloud.bootstrap.name`修改bootstrap文件的名称，使用`spring.cloud.bootstrap.location`或`spring.cloud.bootstrap.additional-location`指定bootstrap文件的位置。

通过bootstrap加载的远程属性默认具有高优先级，本地无法覆盖他们，设置`spring.cloud.config.allowOverride=true`可以修改此行为，注意该属性必须是远程的，本地无效。设定该属性后，还有配套的两个属性可以更精细化的控制覆盖行为：

- spring.cloud.config.overrideNone=true:从任何本地属性源覆盖。
- spring.cloud.config.overrideSystemProperties=false:只有系统属性，命令行参数和环境变量才能覆盖远程设置。



bootstrap默认的外部属性源是Spring Cloud Config Server，但是您可以通过将类型为PropertySourceLocator的bean添加到bootstrap上下文(通过spring.factories)来添加其他源。例如，您可以从其他服务器或数据库读取属性。

```java
@Configuration
public class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        return new MapPropertySource("customProperty",
                Collections.<String, Object>singletonMap("property.from.sample.custom.source", "worked as intended"));
    }

}
```

environment是将要创建的ApplicationContext的环境，可以通过他来获取bootstrap配置文件中的属性等。

此外，还需要在`META-INF/spring.factories`文件中配置如下内容：

```sh
org.springframework.cloud.bootstrap.BootstrapConfiguration=sample.custom.CustomPropertySourceLocator
```

## 刷新配置

当远程属性源发生变化时，会发送`EnvironmentChangeEvent`事件，你可以监听这些事件来：

* 重新绑定@ConfigurationProperties bean
* 修改logger级别

请注意，默认情况下，Spring Cloud Config Client 不会轮询环境中的更改。 通常，我们不建议使用这种方法来检测更改（尽管您可以使用 @Scheduled 注释进行设置）。 如果您有横向扩展的客户端应用程序，最好将 EnvironmentChangeEvent 广播到所有实例，而不是让它们轮询更改（例如，通过使用 Spring Cloud Bus）。

当配置发生变化时，标记为@RefreshScope 的 Spring @Bean 会得到刷新处理。 需要注意的是@RefreshScope 是懒加载的？？？

/refresh端点可以用来更改logger级别和刷新@ConfigurationProperties bean

## 扩展端点

* POST 到 /actuator/env 以更新环境并重新绑定 @ConfigurationProperties 和日志级别。 要启用此端点，您必须设置 management.endpoint.env.post.enabled=true。
* /actuator/refresh 重新加载引导程序上下文并刷新@RefreshScope bean。
* /actuator/restart 关闭 ApplicationContext 并重新启动它（默认情况下禁用）。
* /actuator/pause 和 /actuator/resume 用于调用生命周期方法（ApplicationContext 上的 stop() 和 start()）。

# spring cloud common

服务发现、负载平衡和 断路器 抽象。

## 服务发现

Spring Cloud Commons 提供了 `@EnableDiscoveryClient` 注解。 这会在`META-INF/spring.factories`（文件内容：`org.springframework.cloud.client.discovery.EnableDiscoveryClient=xxx` ）寻找 `DiscoveryClient` 和 `ReactiveDiscoveryClient` 接口的实现。 DiscoveryClient 实现的示例包括 Spring Cloud Netflix Eureka、Spring Cloud Consul Discovery 和 Spring Cloud Zookeeper Discovery。

启用的相关属性如下：

```properties
spring.cloud.discovery.blocking.enabled=false
spring.cloud.discovery.reactive.enabled=false
spring.cloud.discovery.enabled=false
```

默认情况下， DiscoveryClient 的实现会自动向远程发现服务器注册本地 Spring Boot 服务器。 可以通过在 @EnableDiscoveryClient 中设置 autoRegister=false 来禁用此行为。

> @EnableDiscoveryClient不需要显示声明，只要类路径上存在服务发现的类库，就会自动声明

如果类路径中没有基于注册中心的DiscoveryClient，将使用 SimpleDiscoveryClient 实例，它使用属性来获取有关服务和实例的信息。配置信息如下：

```prop
pring.cloud.discovery.client.simple.instances.service1[0].uri=http://s11:8080
```



spring cloud Commons 现在提供了一个 ServiceRegistry 接口，该接口提供 register(Registration) 和 deregister(Registration) 等方法，让您可以提供自定义注册服务。 Registration是一个标记接口。

使用举例：

```java
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguration {
    private ServiceRegistry registry;

    public MyConfiguration(ServiceRegistry registry) {
        this.registry = registry;
    }

    // called through some external process, such as an event or a custom actuator endpoint
    public void register() {
        Registration registration = constructRegistration();
        this.registry.register(registration);
    }
}
```

每个 ServiceRegistry 实现都有自己的 Registry 实现:

- `ZookeeperRegistration` used with `ZookeeperServiceRegistry`
- `EurekaRegistration` used with `EurekaServiceRegistry`
- `ConsulRegistration` used with `ConsulServiceRegistry`

> 服务自动注册时将触发两个事件。 第一个事件称为 InstancePreRegisteredEvent，在注册服务之前触发。 第二个事件称为 InstanceRegisteredEvent，在注册服务后触发。 您可以注册一个 ApplicationListener(s) 来监听和响应这些事件。
>
> 如果 spring.cloud.service-registry.auto-registration.enabled 属性设置为 false，则不会触发这些事件。

actuator 端点`/service-registry` 可以查看服务注册的状态以及注册服务。

如果你的服务器存在多个ip的话，在注册服务的时候，可能想指定某些ip不能向注册中心注册：

```yaml
spring:
  cloud:
    inetutils:
      ignoredInterfaces:
        - docker0
        - veth.*
```

偏向使用某些ip：

```yaml
spring:
  cloud:
    inetutils:
      preferredNetworks:
        - 192.168
        - 10.0
```

使用回环地址：

```yaml
spring:
  cloud:
    inetutils:
      useOnlySiteLocalInterfaces: true
```

## 负载均衡

ReactiveLoadBalancer接口定义负载均衡器，spring cloud实现了两种：轮询和随机。默认实现是RoundRobinLoadBalancer。

ServiceInstanceListSupplier 接口定义获取服务实例信息，基于服务发现的`DiscoveryClientServiceInstanceListSupplier`，下面接受的其他均衡器都是委托DiscoveryClientServiceInstanceListSupplier来获取服务列表，然后做进一步处理。



CachingServiceInstanceListSupplier：基于缓存的服务实例列表

```prop
spring.cloud.loadbalancer.cache.ttl=35
spring.cloud.loadbalancer.cache.capacity=256
spring.cloud.loadbalancer.cache.enabled=false
```

> 尽管基本的非缓存实现对于原型设计和测试很有用，但它的效率远低于缓存版本，因此我们建议始终在生产中使用缓存版本。 如果缓存已由 DiscoveryClient 实现完成，例如 EurekaDiscoveryClient，则应禁用负载平衡器缓存以防止双重缓存。



ZonePreferenceServiceInstanceListSupplier:检测相同zone的服务实例列表

在应用中设置`eureka.instance.metadata-map.zone=xx`注册实例的zone信息，A调用B时，就会选择和A的zone相同的B,可以通过设置`spring.cloud.loadbalancer.zone`来调用特定zone的B。

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withZonePreference()
                    .withCaching()
                    .build(context);
    }
}
```



HealthCheckServiceInstanceListSupplier : 定期检查DiscoveryClientServiceInstanceListSupplier返回的实例是否健康。

> 这种机制在使用 SimpleDiscoveryClient 时特别有用。 对于由实际 Service Registry 支持的客户端，没有必要使用，因为我们在查询外部 ServiceDiscovery 后已经获得了健康的实例。

```properties
spring.cloud.loadbalancer.health-check.initialDelay=
spring.cloud.loadbalancer.health-check.interval=
spring.cloud.loadbalancer.health-check.path.default= /actuator/health(默认)
spring.cloud.loadbalancer.health-check.path.[SERVICE_ID]=
```

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withHealthChecks()
                    .build(context);
        }
    }
```



SameInstancePreferenceServiceInstanceListSupplier:  直接选择先前的实例

```java
public class CustomLoadBalancerConfiguration {
    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withSameInstancePreference()
                    .build(context);
        }
    }
```



RequestBasedStickySessionServiceInstanceListSupplier: 基于会话粘性，在cookie中存储了实例id，选择的时候，只要匹配实例id就是先前的实例。

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                   .withDiscoveryClient()
                   .withRequestBasedStickySession()
                   .build(context);
        }
    }
```



HintBasedServiceInstanceListSupplier：选择命中的实例，用户可以在请求中指定选择那个实例

`eureka.instance.metadata-map.hit=xxx`设置实例的hit值，然后请求头`X-SC-LB-Hint`（spring.cloud.loadbalancer.hint-header-name修改此值）传递hit值。HintBasedServiceInstanceListSupplier会比较这两个值，选择具体的实例

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                   .withDiscoveryClient()
                   .withHints()
                   .withCaching()
                   .build(context);
    }
}
```

