---
title: spring-cloud-eureka
date: 2021-09-26 13:33:46
tags:	
 - 服务发现
categories:
 - 微服务

---

# Eureka server

`org.springframework.cloud:spring-cloud-starter-netflix-eureka-server`

> 如果您的项目已经使用 Thymeleaf 作为其模板引擎，则 Eureka 服务器的 Freemarker 模板可能无法正确加载。 在这种情况下，需要手动配置模板加载器：
>
> ```yaml
> spring:
>   freemarker:
>     template-loader-path: classpath:/templates/
>     prefer-file-system-access: false
> ```



开启

```java
@SpringBootApplication
@EnableEurekaServer
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

服务器有一个主页，以及/eureka/* 下用于 Eureka 功能的 HTTP API 端点。

Eureka 服务器没有后端存储，但注册中心中的服务实例都必须发送心跳以保持其注册最新（因此这可以在内存中完成）。 客户端还有一个 Eureka 注册的内存缓存（因此他们不必为每个服务请求都去注册中心）。

默认情况下，每个 Eureka 服务器也是 Eureka 客户端，并且需要（至少一个）服务 URL 来定位对等方。 如果您不提供它，该服务仍然可以运行，但它会在您的日志中填满大量关于无法向对等方注册的噪音，你可以通过以下配置关闭：

```yaml
eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
```

通过运行多个实例并要求它们相互注册，Eureka 可以变得更具弹性和可用性。 事实上，这是默认行为，因此您需要做的就是将有效的 serviceUrl 添加到对等点，如以下示例所示：

```yaml
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: https://peer2/eureka/

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: https://peer1/eureka/
```

一个 YAML 文件，两份配置（通过---分割）。 实际上，如果您在知道自己主机名的机器上运行，则不需要 eureka.instance.hostname（默认情况下，使用 java.net.InetAddress 查找它）。

您可以将多个对等点添加到系统中，并且只要它们通过至少一个边缘连接，就会在它们之间同步注册。 如果对等点在物理上是分开的（在一个数据中心内或在多个数据中心之间），那么系统原则上可以承受“裂脑”类型的故障。

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: https://peer1/eureka/,http://peer2/eureka/,http://peer3/eureka/

---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2

---
spring:
  profiles: peer3
eureka:
  instance:
    hostname: peer3
```

在某些情况下，Eureka 更喜欢通告服务的 IP 地址而不是主机名。 将 eureka.instance.preferIpAddress 设置为 true，当应用程序向 eureka 注册时，它使用其 IP 地址而不是其主机名。

> 如果 Java 无法确定主机名，则将 IP 地址发送到 Eureka。 设置主机名的唯一显式方法是设置 eureka.instance.hostname 属性。 您可以在运行时使用环境变量 — 设置主机名，例如 eureka.instance.hostname=${HOST_NAME}。



您只需通过 spring-boot-starter-security 将 Spring Security 添加到服务器的类路径，即可保护您的 Eureka 服务器。 默认情况下，当 Spring Security 在类路径上时，它将要求向应用程序发送每个请求时都发送一个有效的 CSRF 令牌。 Eureka 客户端通常不会拥有有效的跨站点请求伪造 (CSRF) 令牌，您需要为 /eureka/** 端点禁用此要求。 例如：

```java
@EnableWebSecurity
class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}
```

Eureka 服务器所依赖的 JAXB 模块已在 JDK 11 中删除。如果您打算在运行 Eureka 服务器时使用 JDK 11，则必须在 POM 或 Gradle 文件中包含这些依赖项。

```xml
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
</dependency>
```

# Eureka Clients

依赖：`org.springframework.cloud:spring-cloud-starter-netflix-eureka-client`

注册配置：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

当客户端向 Eureka 注册时，它会提供有关自身的元数据 ，例如主机、端口、健康指标 URL、主页和其他详细信息。 Eureka 服务端接每个实例的心跳消息。 如果心跳故障超过可配置的时间表，则该实例通常会从注册表中删除。

要禁用 Eureka Discovery Client，您可以将 eureka.client.enabled 设置为 false。 当 spring.cloud.discovery.enabled 设置为 false 时，Eureka Discovery Client 也将被禁用

Eureka 实例的状态页和健康指示器分别默认为 /info 和 /health。 如果您使用非默认上下文路径或 servlet 路径（例如 server.servletPath=/custom），即使对于 Actuator 应用程序，您也需要更改这些设置。 以下示例显示了两个设置的默认值：

```yaml
eureka:
  instance:
    statusPageUrlPath: ${server.servletPath}/info
    healthCheckUrlPath: ${server.servletPath}/health
```

如果您的应用程序希望通过 HTTPS 进行联系，您可以在 EurekaInstanceConfigBean 中设置两个标志：

- `eureka.instance.[nonSecurePortEnabled]=[false]`
- `eureka.instance.[securePortEnabled]=[true]`

由于 Eureka 内部工作的方式，它仍然为状态和主页发布一个非安全的 URL，除非你也明确地覆盖它们。 您可以使用占位符来配置 eureka 实例 URL，如以下示例所示：

```yaml
eureka:
  instance:
    statusPageUrl: https://${eureka.hostname}/info
    healthCheckUrl: https://${eureka.hostname}/health
    homePageUrl: https://${eureka.hostname}/
```

## 健康检查机制

默认情况下，Eureka 服务端使用客户端心跳来确定客户端是否已启动。 因此，在成功注册后，Eureka 宣布应用程序处于“UP”状态。 这种行为可以通过启用 Eureka 健康检查来改变，客户端实例将应用程序状态传播到 Eureka服务端。 因此，每个其他应用程序都不会向处于“UP”以外的状态的应用程序发送流量。 以下示例显示了如何为客户端启用健康检查：

```yaml
eureka:
  client:
    healthcheck:
      enabled: true
```

如果您需要对健康检查进行更多控制，请考虑实现您自己的 com.netflix.appinfo.HealthCheckHandler

Eureka 客户端将正在运行的实例的信息注册到 Eureka 服务器。  注册发生在第一次心跳时（30 秒后）。

Eureka 客户端需要通过每 30 秒发送一次心跳来更新租约。 更新通知 Eureka 服务器该实例还活着。 如果服务器在 90 秒内没有看到更新，它将从其注册表中删除该实例。 建议不要更改更新间隔，因为服务器使用该信息来确定客户端到服务器通信是否存在广泛传播的问题。

Eureka 客户端从服务器获取注册表信息并将其缓存在本地。 之后，客户端使用该信息来查找其他服务。 定期（每 30 秒）增量更新（上次更新到现在之间的变更信息）。 增量信息在服务器中保存的时间更长（大约 3 分钟），因此增量提取可能会再次返回相同的实例。 Eureka 客户端自动处理重复信息。

获取增量后，Eureka 客户端通过比较服务器返回的实例计数来与服务器协调信息，如果由于某种原因信息不匹配，则再次获取整个注册表信息。 Eureka 服务器缓存 deltas、整个注册表和每个应用程序的压缩负载，以及未压缩的信息。 有效负载还支持 JSON/XML 格式。 Eureka 客户端使用 jersey apache 客户端获取压缩 JSON 格式的信息。

Eureka 客户端在关闭时向 Eureka 服务器发送取消请求。 这会从服务器的实例注册表中删除实例，从而有效地使实例脱离流量。这是在 Eureka 客户端关闭时完成的，应用程序应确保在其关闭期间调用以下代码：

```java
     DiscoveryManager.getInstance().shutdownComponent()
```

来自 Eureka 客户端的所有操作可能需要一些时间才能反映在 Eureka 服务器中，随后会反映在其他 Eureka 客户端中。 这是因为 eureka 服务器上的负载缓存会定期刷新以反映新信息。 Eureka 客户端也会定期获取增量。 因此，将更改传播到所有 Eureka 客户端可能需要长达 2 分钟的时间。

如果 Eureka 服务器检测到比预期数量多的注册客户端异常终止了它们的连接，并且同时等待驱逐，它们将进入自我保护模式。 这样做是为了确保灾难性网络事件不会清除 eureka 注册表数据，并将其向下传播到所有客户端。

为了更好地理解自我保护，首先要了解eureka客户端如何“结束”他们的注册生命周期。 eureka 协议要求客户端在永久离开时执行明确的注销操作。 例如，在提供的 java 客户端中，这是在 shutdown() 方法中完成的。 任何连续 3 次心跳更新失败的客户端都被认为是不正常终止，并将被后台驱逐进程驱逐。 当当前注册表的 > 15% 处于此后期状态时，将启用自我保护。

在自我保护模式下，eureka服务器将停止驱逐所有实例，直到：它看到的心跳更新次数重新高于预期阈值，或自我保护被禁用（见下文）默认情况下启用自保。

```yaml
eureka:
  server:
    enable-self-preservation: false
    renewal-percent-threshold: 0.85
```



`leaseRenewalIntervalInSeconds`(默认30s)：指示 eureka 客户端需要多长时间（以秒为单位）向 eureka 服务器发送心跳以表明它仍然活着。 如果在`leaseExpirationDurationInSeconds` 中指定的时间段内未收到心跳，eureka 服务器将从其视图中删除该实例，从而禁止流向该实例的流量。请注意，如果实例实现 HealthCheckCallback 然后决定使其自身不可用，则该实例仍然无法获取流量

`leaseExpirationDurationInSeconds`（默认90s）：指示 eureka 服务器自收到最后一次心跳后等待的时间（以秒为单位），然后才能从其视图中删除此实例，并通过禁止流向此实例的流量。 将此值设置得太长可能意味着即使实例不活动，流量也可以路由到实例。 将此值设置得太小可能意味着，由于临时网络故障，实例可能会丢失流量。此值至少要设置为高于`leaseRenewalIntervalInSeconds` 中指定的值。

`registryFetchIntervalSeconds`（30s）：指示从 eureka 服务器获取注册表信息的频率（以秒为单位）

当服务下线时，调用该服务的客户端至少需要120s才能知道。





标准的元数据包括主机名、IP 地址、端口号、状态页面和健康检查等信息。 这些信息会注册到服务端。如果想要添加额外的信息，你可以配置`eureka.instance.metadataMap`，并且这些元数据可以在远程客户端中访问。



默认的示例Id: `${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}`,可以通过以下方式修改：

```yaml
eureka:
  instance:
    instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
```



使用原生的com.netflix.discovery.EurekaClient（注意不是DiscoveryClient）：

```java
@Autowired
private EurekaClient discoveryClient;

public String serviceUrl() {
    InstanceInfo instance = discoveryClient.getNextServerFromEureka("STORES", false);
    return instance.getHomePageUrl();
}
```

DiscoveryClient使用：

```java
@Autowired
private DiscoveryClient discoveryClient;

public String serviceUrl() {
    List<ServiceInstance> list = discoveryClient.getInstances("STORES");
    if (list != null && list.size() > 0 ) {
        return list.get(0).getUri();
    }
    return null;
}
```



默认情况下，EurekaClient 使用 Spring 的 RestTemplate 进行 HTTP 通信。 如果您希望改用 Jersey，则需要将 Jersey 依赖项添加到您的类路径中。 以下示例显示了您需要添加的依赖项：

```xml
<dependency>
    <groupId>com.sun.jersey</groupId>
    <artifactId>jersey-client</artifactId>
</dependency>
<dependency>
    <groupId>com.sun.jersey</groupId>
    <artifactId>jersey-core</artifactId>
</dependency>
<dependency>
    <groupId>com.sun.jersey.contribs</groupId>
    <artifactId>jersey-apache-client4</artifactId>
</dependency>
```



相同zone优先

```
eureka.instance.metadataMap.zone = zone1
eureka.client.preferSameZoneEureka = true
```



默认情况下，EurekaClient bean 是可刷新的，这意味着可以更改和刷新 Eureka 客户端属性。 当刷新发生时，客户端将从 Eureka 服务器注销，并且可能有一小段时间给定服务的所有实例都不可用。 避免这种情况发生的一种方法是禁用刷新 Eureka 客户端的功能。 为此，请设置 eureka.client.refresh.enable=false。



一旦服务器开始接收流量，在服务器上执行的所有操作都会复制到服务器知道的所有对等节点。 如果操作由于某种原因失败，则在下一次心跳时会协调信息，该心跳也会在服务器之间复制。

当 Eureka 服务器启动时，它会尝试从相邻节点获取所有实例注册表信息。 如果从节点获取信息出现问题，服务器会在放弃之前尝试所有对等点。 如果服务器能够成功获取所有实例，则它会根据该信息设置它应该接收的续订阈值。 如果任何时候续订率低于为该值配置的百分比（15 分钟内低于 85%），服务器将停止剔除实例以保护当前实例注册表信息。

在 Netflix 中，上述保护措施称为自保模式，主要用于一组客户端和 Eureka Server 之间存在网络分区的场景中的保护。 在这些情况下，服务器会尝试保护它已有的信息。 可能会出现大规模中断的情况，这可能会导致客户端获取不再存在的实例。 客户端必须确保他们对 eureka 服务器返回不存在或无响应的实例具有弹性。 在这些情况下最好的保护是快速超时并尝试其他服务器。

在服务器无法从相邻节点获取注册信息的情况下，它会等待几分钟（5 分钟），以便客户端可以注册他们的信息。 

Eureka 服务器使用 Eureka 客户端和服务器之间使用的相同机制相互通信。

在对等点之间网络中断的情况下，可能会发生以下情况

* 对等点之间的心跳复制可能会失败，服务器会检测到这种情况并进入自我保护模式以保护当前状态。
* 注册可能发生在孤立的服务器中，一些客户端可能会反映新的注册，而其他客户端可能不会。
* 在网络连接恢复到稳定状态后，这种情况会自动更正。 当对等方能够正常通信时，注册信息会自动传输到没有它们的服务器。
