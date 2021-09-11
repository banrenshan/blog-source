---
title: spring data redis
tags: 
 - redis
 - spring-data
categories:
 - NOSQL
---

# redis支持

## 连接redis

关于连接相关的api都存储在org.springframework.data.redis.connection包下。RedisConnection负责与redis通信，它还自动将底层连接库异常转换为 Spring 一致的 DAO 异常层次结构，以便您可以在不更改任何代码的情况下切换底层连接库。

> 对于需要原生库 API 的极端情况，RedisConnection 提供了一个专用方法 (getNativeConnection)，该方法返回用于通信的原始底层对象。

RedisConnectionFactory 创建 RedisConnection 对象。 此外，工厂充当 PersistenceExceptionTranslator 对象，这意味着一旦声明，它们就可以让您进行透明的异常转换。 例如，您可以通过使用@Repository 注释和AOP 进行异常转换。 有关更多信息，请参阅 Spring Framework 文档中的专用部分？？？。

有些底层连接库可能不支持redis的某些特性，如果我们在这样的连接上执行了不支持的功能，会抛出UnsupportedOperationException异常。

### Lettuce连接器

Lettuce 基于 Netty 的开源连接器。

```xml
  <dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.1.4.RELEASE</version>
  </dependency>
```

```java
@Configuration
class AppConfig {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    return new LettuceConnectionFactory(new RedisStandaloneConfiguration("server", 6379));
  }
}
```

默认情况下，由 LettuceConnectionFactory 创建的所有 LettuceConnection 实例,为所有非阻塞和非事务操作,共享相同的线程安全连接。 要每次使用专用连接，请将 shareNativeConnection 设置为 false。

如果 shareNativeConnection 设置为 false，LettuceConnectionFactory 也可以配置为使用 LettucePool 来池化阻塞或事务连接。

Lettuce 与 Netty 的native 传输集成，让您可以使用 Unix 域套接字与 Redis 进行通信。  以下示例显示了如何在 /var/run/redis.sock 为 Unix 域套接字创建连接工厂：

```java
@Configuration
class AppConfig {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    return new LettuceConnectionFactory(new RedisSocketConfiguration("/var/run/redis.sock"));
  }
}
```

### Jedis 连接器

```xml
  <dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.6.3</version>
  </dependency>
```

```java
@Configuration
class AppConfig {

  @Bean
  public JedisConnectionFactory redisConnectionFactory() {
    return new JedisConnectionFactory();
  }
}
```

### 读写分离

```java
@Configuration
class WriteToMasterReadFromReplicaConfiguration {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
      .readFrom(REPLICA_PREFERRED)
      .build();

    RedisStandaloneConfiguration serverConfig = new RedisStandaloneConfiguration("server", 6379);

    return new LettuceConnectionFactory(serverConfig, clientConfig);
  }
}
```

## Sentinel 

```java
/**
 * Jedis
 */
@Bean
public RedisConnectionFactory jedisConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new JedisConnectionFactory(sentinelConfig);
}

/**
 * Lettuce
 */
@Bean
public RedisConnectionFactory lettuceConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new LettuceConnectionFactory(sentinelConfig);
}
```

配置文件：

- `spring.redis.sentinel.master`:  master的名称.
- `spring.redis.sentinel.nodes`: 逗号分隔的主机：端口.
- `spring.redis.sentinel.password`: redis的认证密码

> 有时，需要与其中一个哨兵直接交互。 使用 RedisConnectionFactory.getSentinelConnection() 或 RedisConnection.getSentinelCommands() 可让您访问配置的第一个active  Sentinel。

## RedisTemplate

RedisTemplate默认使用jdk序列化方式，你可以更改此行为。您还可以将序列化设置为 null，并通过将 enableDefaultSerializer 属性设置为 false 来将 RedisTemplate 与原始字节数组一起使用。

由于存储在 Redis 中的键和值通常是 java.lang.String，因此 Redis 模块为 RedisConnection 和 RedisTemplate 提供了两个扩展，分别是 StringRedisConnection（DefaultStringRedisConnection 实现）和 StringRedisTemplate 。 除了绑定到 String 键之外，还使用 StringRedisSerializer，这意味着存储的键和值是人类可读的。 以下清单显示了一个示例：

```java
public class Example {

  @Autowired
  private StringRedisTemplate redisTemplate;

  public void addLink(String userId, URL url) {
    redisTemplate.opsForList().leftPush(userId, url.toExternalForm());
  }
}
```

与其他 Spring 模板一样，RedisTemplate 和 StringRedisTemplate 允许您通过 RedisCallback 接口直接与 Redis 对话。 此功能可让您完全控制，因为它直接与 RedisConnection 对话。 请注意，当使用 StringRedisTemplate 时，回调会收到 StringRedisConnection 的实例。 以下示例显示了如何使用 RedisCallback 接口：

```java
public void useCallback() {

  redisTemplate.execute(new RedisCallback<Object>() {
    public Object doInRedis(RedisConnection connection) throws DataAccessException {
      Long size = connection.dbSize();
      // Can cast to StringRedisConnection if using a StringRedisTemplate
      ((StringRedisConnection)connection).set("key", "value");
    }
   });
}
```

## 序列化

从框架的角度来看，Redis 中存储的数据只有字节。 虽然 Redis 本身支持各种类型，但在大多数情况下，这些类型指的是数据的存储方式，而不是它所代表的内容。 由用户决定是否将信息转换为字符串或任何其他对象。

在 Spring Data 中，用户（自定义）类型和原始数据之间的转换由 org.springframework.data.redis.serializer 包中的类处理。这个包包含两种类型的序列化器，顾名思义，它们负责序列化过程：

* 基于 RedisSerializer 的双向序列化器。
* 使用 RedisElementReader 和 RedisElementWriter 的元素读取器和写入器。

主要区别在于，RedisSerializer 主要序列化为 byte[]，而读取器和写入器使用 ByteBuffer。

有多种实现可用（包括本文档中已经提到的两种）：

* JdkSerializationRedisSerializer，默认用于RedisCache和RedisTemplate。
* StringRedisSerializer
* OxmSerializer
* GenericJackson2JsonRedisSerializer 、Jackson2JsonRedisSerializer

请注意，存储格式不仅限于值。它可以用于键、值或散列，没有任何限制。

## Hash 映射

可以使用Redis 中的各种数据结构来存储数据。 Jackson2JsonRedisSerializer 可以转换 JSON 格式的对象。 理想情况下，可以使用普通键将 JSON 存储为值。 您可以通过使用 Redis 哈希来实现更复杂的结构化对象映射。 Spring Data Redis 提供了各种将数据映射到哈希的策略（取决于用例）：

* 直接映射，通过使用 HashOperations 和序列化
* 使用 Redis 存储库
* 使用 HashMapper 和 HashOperations

###  Hash Mappers

映射 Redis 哈希 到 Map<K, V> 。多种实现可用：

* BeanUtilsHashMapper 使用 Spring 的 BeanUtils。

* ObjectHashMapper 使用对象到哈希映射。

* Jackson2HashMapper 使用 FasterXML Jackson。

```java
public class Person {
  String firstname;
  String lastname;

  // …
}

public class HashMapping {

  @Autowired
  HashOperations<String, byte[], byte[]> hashOperations;

  HashMapper<Object, byte[], byte[]> mapper = new ObjectHashMapper();

  public void writeHash(String key, Person person) {

    Map<byte[], byte[]> mappedHash = mapper.toHash(person);
    hashOperations.putAll(key, mappedHash);
  }

  public Person loadHash(String key) {

    Map<byte[], byte[]> loadedHash = hashOperations.entries("key");
    return (Person) mapper.fromHash(loadedHash);
  }
}
```

### Jackson2HashMapper

Jackson2HashMapper 使用 FasterXML Jackson 为域对象提供 Redis Hash 映射。 Jackson2HashMapper 可以将顶级属性映射为哈希字段名称，并且可以选择将结构展平。 简单类型映射到简单值。 复杂类型（嵌套对象、集合、映射等）表示为嵌套 JSON。

展平为所有嵌套属性创建单独的哈希条目，并尽可能将复杂类型解析为简单类型。

```java
public class Person {
  String firstname;
  String lastname;
  Address address;
  Date date;
  LocalDateTime localDateTime;
}

public class Address {
  String city;
  String country;
}
```

普通映射结果：

| Hash Field    | Value                                                  |
| :------------ | :----------------------------------------------------- |
| firstname     | `Jon`                                                  |
| lastname      | `Snow`                                                 |
| address       | `{ "city" : "Castle Black", "country" : "The North" }` |
| date          | `1561543964015`                                        |
| localDateTime | `2018-01-02T12:13:14`                                  |

flat映射结果：

| Hash Field      | Value                 |
| :-------------- | :-------------------- |
| firstname       | `Jon`                 |
| lastname        | `Snow`                |
| address.city    | `Castle Black`        |
| address.country | `The North`           |
| date            | `1561543964015`       |
| localDateTime   | `2018-01-02T12:13:14` |

> java.util.Date 和 java.util.Calendar 以毫秒表示。如果 jackson-datatype-jsr310 在类路径上，则 JSR-310 日期/时间类型将序列化为其 toString 形式。

## Redis Messaging (Pub/Sub)

Redis 消息传递大致可以分为两个方面的功能：

* 消息的发布或生产

* 消息的订阅或消费

这通常被称为发布/订阅（简称 Pub/Sub）模式。RedisTemplate 类用于消息生产。spring data 创建专门的消息容器监听并异步接受消息，并将原始消息转化成对应的POJOs (MDPs)。

### 发布消息

要发布消息，您可以像其他操作一样使用低级 RedisConnection 或高级 RedisTemplate。 两个实体都提供发布方法，该方法接受消息和目标通道作为参数。 虽然 RedisConnection 需要原始数据（字节数组），但 RedisTemplate 允许将任意对象作为消息传入，如下例所示：

```java
// send message through connection RedisConnection con = ...
byte[] msg = ...
byte[] channel = ...
con.publish(msg, channel); // send message through RedisTemplate
RedisTemplate template = ...
template.convertAndSend("hello!", "world");
```

### 接受消息

在接收端，可以通过直接命名或使用模式匹配来订阅一个或多个频道。 后一种方法非常有用，因为它不仅允许使用一个命令创建多个订阅，而且还可以侦听订阅时尚未创建的频道（只要它们匹配模式）。

在底层，RedisConnection 提供了 subscribe 和 pSubscribe 方法，它们分别映射了 Redis 命令以按频道或按模式订阅。 请注意，可以使用多个通道或模式作为参数。 要更改连接的订阅或查询它是否正在侦听，RedisConnection 提供了 getSubscription 和 isSubscribed 方法。

> Spring Data Redis 中的订阅命令被阻塞。 也就是说，在连接上调用 subscribe 会导致当前线程在开始等待消息时阻塞。 只有在取消订阅时才会释放线程，通过另一个线程在同一连接上调用 unsubscribe 或 pUnsubscribe实现 。 

如前所述，一旦订阅，连接就会开始等待消息。 仅允许添加新订阅、修改现有订阅和取消现有订阅的命令。 调用 subscribe、pSubscribe、unsubscribe 或 pUnsubscribe 以外的任何内容都会引发异常。

为了订阅消息，需要实现 MessageListener 回调。 每次有新消息到达时，都会调用回调并且由 onMessage 方法运行用户代码。 该接口不仅可以访问实际消息，还可以访问到频道等信息。 

### 消息监听器容器

RedisMessageListenerContainer 充当消息侦听容器。 它用于从 Redis 通道接收消息并驱动注入其中的 MessageListener 实例。 侦听器容器负责消息接收的所有线程并分派到侦听器中进行处理。 消息侦听器容器是 MDP 和消息提供者之间的中介，负责注册接收消息、资源获取和释放、异常转换等。 这让您作为应用程序开发人员可以编写与接收消息相关的业务逻辑，并将样板代码委托给框架。

`MessageListener` 还可以实现 `SubscriptionListener` 以在订阅/取消订阅确认时接收通知。 同步调用时，侦听订阅通知很有用。

此外，为了最小化应用程序占用空间，RedisMessageListenerContainer 允许多个侦听器共享一个连接和一个线程，即使它们不共享订阅。 因此，无论应用程序跟踪多少个侦听器或通道，运行时成本在其整个生命周期中都保持不变。 此外，容器允许运行时配置更改，以便您可以在应用程序运行时添加或删除侦听器，而无需重新启动。 此外，容器使用延迟订阅方法，仅在需要时使用 RedisConnection。 如果所有侦听器都取消订阅，则自动执行清理，并释放线程。

为了支持处理消息的异步特性，容器需要一个 java.util.concurrent.Executor（或 Spring 的 TaskExecutor）来分发消息。 根据负载、侦听器的数量或运行时环境，您应该更改或调整Executor以更好地满足您的需求。 

#### MessageListenerAdapter

 简而言之，它允许您将几乎任何类公开为 MDP（尽管有一些限制）。

```java
public interface MessageDelegate {
  void handleMessage(String message);
  void handleMessage(Map message); 
  void handleMessage(byte[] message);
  void handleMessage(Serializable message);
  // pass the channel/pattern as well
  void handleMessage(Serializable message, String channel);
 }
```

请注意，虽然该接口没有继承MessageListener 接口，但它仍然可以通过使用 MessageListenerAdapter 类用作 MDP。 还要注意各种消息处理方法如何根据它们可以接收和处理的各种消息类型的内容进行强类型化。 此外，消息发送到的通道或模式可以作为字符串类型的第二个参数传递给方法：

```java
public class DefaultMessageDelegate implements MessageDelegate {
  // implementation elided for clarity...
}
```

请注意 MessageDelegate 接口（上面的 DefaultMessageDelegate 类）完全没有 Redis 依赖项。 它确实是一个 POJO，我们使用以下配置将其制成 MDP：

```xml
<redis:listener-container>
  <redis:listener ref="listener" method="handleMessage" topic="chatroom" />
</redis:listener-container>
<bean id="listener" class="redisexample.DefaultMessageDelegate"/>
```

每次接收到消息时，适配器都会自动透明地执行低级格式和所需对象类型之间的转换（使用配置的 RedisSerializer）。 任何由方法调用引起的异常都会被容器捕获并处理（默认情况下，异常会被记录下来）。

## 事务

Redis 通过multi、exec 和discard 命令为事务提供支持。 这些操作在 RedisTemplate 上可用。 但是，RedisTemplate 不保证在同一连接下运行事务中的所有操作。

Spring Data Redis 提供了 SessionCallback 接口，供同一个连接需要执行多个操作时使用，例如使用Redis事务时。 下面的例子使用了multi方法：

```java
//execute a transaction
List<Object> txResults = redisTemplate.execute(new SessionCallback<List<Object>>() {
  public List<Object> execute(RedisOperations operations) throws DataAccessException {
    operations.multi();
    operations.opsForSet().add("key", "value1");

    // This will contain the results of all operations in the transaction
    return operations.exec();
  }
});
System.out.println("Number of items added to set: " + txResults.get(0));
```

exec 方法的返回结果会被对应的序列化器反序列化，还有一个重载方法，可以执行序列化器。

### @Transactional

默认情况下，RedisTemplate 不参与 Spring 事务。 如果您希望RedisTemplate 在使用@Transactional 或TransactionTemplate 时使用Redis 事务，则需要通过设置setEnableTransactionSupport(true) 为每个RedisTemplate 显式启用事务支持。 启用事务支持将 RedisConnection 绑定到由 ThreadLocal 支持的当前事务。 如果事务完成且没有错误，Redis 事务将使用 EXEC 提交，否则使用 DISCARD 回滚。 Redis 事务是面向批处理的。 在正在进行的事务期间发出的命令被排队，并且仅在提交事务时应用。

Spring Data Redis 在正在进行的事务中区分只读和写命令。 只读命令（例如 KEYS）通过管道传输到新的（非线程绑定的）RedisConnection 以允许读取。 写入命令由 RedisTemplate 排队并在提交时应用。

```java
@Configuration
@EnableTransactionManagement           //1                      
public class RedisTxContextConfiguration {

  @Bean
  public StringRedisTemplate redisTemplate() {
    StringRedisTemplate template = new StringRedisTemplate(redisConnectionFactory());
    // explicitly enable transaction support
    template.setEnableTransactionSupport(true);    //2          
    return template;
  }

  @Bean
  public RedisConnectionFactory redisConnectionFactory() {
    // jedis || Lettuce
  }

  @Bean
  public PlatformTransactionManager transactionManager() throws SQLException {
    return new DataSourceTransactionManager(dataSource());   //3
  }

  @Bean
  public DataSource dataSource() throws SQLException {
    // ...
  }
}
```

1. 配置 Spring Context 以启用声明式事务管理。
2. 通过将连接绑定到当前线程来配置 RedisTemplate 以参与事务。
3. 事务管理需要 PlatformTransactionManager。Spring Data Redis 不附带 PlatformTransactionManager 实现。假设您的应用程序使用 JDBC，Spring Data Redis 可以使用现有的事务管理器参与事务。

## Pipelining

Redis 提供对流水线的支持，这涉及向服务器发送多个命令而无需等待回复，然后一步读取回复。 当您需要连续发送多个命令时，流水线可以提高性能，例如将许多元素添加到同一个 List。

Spring Data Redis 提供了几个 RedisTemplate 方法来在管道中运行命令。 如果您不关心流水线操作的结果，您可以使用标准的 execute 方法，为流水线参数传递 true。 executePipelined 方法在管道中运行提供的 RedisCallback 或 SessionCallback 并返回结果，如以下示例所示：

```java
//pop a specified number of items from a queue
List<Object> results = stringRedisTemplate.executePipelined(
  new RedisCallback<Object>() {
    public Object doInRedis(RedisConnection connection) throws DataAccessException {
      StringRedisConnection stringRedisConn = (StringRedisConnection)connection;
      for(int i=0; i< batchSize; i++) {
        stringRedisConn.rPop("myqueue");
      }
    return null;
  }
});
```

请注意，从 RedisCallback 返回的值必须为 null，因为为了返回流水线命令的结果而丢弃该值。



## redis工具包

包 org.springframework.data.redis.support 提供了各种依赖 Redis 作为后备存储的可重用组件。 目前，该包包含基于 Redis 的各种 JDK 接口实现，例如原子计数器和 JDK 集合。以RedisList为列：

```xml
  <bean id="queue" class="org.springframework.data.redis.support.collections.DefaultRedisList">
    <constructor-arg ref="redisTemplate"/>
    <constructor-arg value="queue-key"/>
  </bean>
```

```java
public class AnotherExample {
  // injected
  private Deque<String> queue;
  public void addTag(String tag) {
    queue.push(tag);
  }
}
```

## redis缓存

Spring Redis 通过 org.springframework.data.redis.cache 包提供了 Spring 缓存抽象的实现。 要使用 Redis 作为后备实现，请将 RedisCacheManager 添加到您的配置中，如下所示：

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
	return RedisCacheManager.create(connectionFactory);
}
```

RedisCacheManager 行为可以使用 RedisCacheManagerBuilder 进行配置，让您可以设置默认的 RedisCacheConfiguration、事务行为和预定义的缓存。

```java
RedisCacheManager cm = RedisCacheManager.builder(connectionFactory)
	.cacheDefaults(defaultCacheConfig())
	.withInitialCacheConfigurations(singletonMap("predefined", defaultCacheConfig().disableCachingNullValues()))
	.transactionAware()
	.build();
```

使用 RedisCacheManager 创建的 RedisCache 的行为是使用 RedisCacheConfiguration 定义的。 该配置允许您设置key过期时间、前缀和 RedisSerializer 实现，如以下示例所示：

```java
RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
    .entryTtl(Duration.ofSeconds(1))
	.disableCachingNullValues();
```

RedisCacheManager 默认为无锁 RedisCacheWriter (用于读取和写入二进制值)。 无锁缓存提高了吞吐量。 缺少条目锁定可能会导致 putIfAbsent 和 clean 方法出现重叠的非原子命令，因为这些方法需要将多个命令发送到 Redis。 锁定对应物通过设置显式锁定key并检查此key的存在来防止命令重叠，这会导致额外的请求和潜在的命令等待时间。可以选择加入锁定行为，如下所示：

```java
RedisCacheManager cm = RedisCacheManager.build(RedisCacheWriter.lockingRedisCacheWriter())
	.cacheDefaults(defaultCacheConfig())
	...
```

默认情况下，缓存条目的任何键都以实际缓存名称为前缀，后跟两个冒号。 此行为可以更改为静态和计算前缀。

```java
// static key prefix
RedisCacheConfiguration.defaultCacheConfig().prefixKeysWith("( ͡° ᴥ ͡°)");
// computed key prefix
RedisCacheConfiguration.defaultCacheConfig().computePrefixWith(cacheName -> "¯\_(ツ)_/¯" + cacheName);
```

RedisCacheManager默认设置：

| Setting             | Value                                          |
| :------------------ | :--------------------------------------------- |
| Cache Writer        | Non-locking                                    |
| Cache Configuration | `RedisCacheConfiguration#defaultConfiguration` |
| Initial Caches      | None                                           |
| Transaction Aware   | No                                             |

**RedisCacheConfiguration** 默认设置

| Cache `null`       | Yes                                                          |
| ------------------ | ------------------------------------------------------------ |
| Prefix Keys        | Yes                                                          |
| Default Prefix     | The actual cache name                                        |
| Key Serializer     | `StringRedisSerializer`                                      |
| Value Serializer   | `JdkSerializationRedisSerializer`                            |
| Conversion Service | `DefaultFormattingConversionService` with default cache key converters |

默认情况下 RedisCache，统计信息被禁用。 使用 RedisCacheManagerBuilder.enableStatistics() 通过 RedisCache#getStatistics() 收集本地命中和未命中，返回所收集数据的快照。

# redis存储库

**Redis Repositories 至少需要 Redis Server 版本 2.8.0，并且不支持事务。 确保使用禁用事务支持的 RedisTemplate。**

## 使用

```java
@RedisHash("people")
public class Person {

  @Id String id;
  String firstname;
  String lastname;
  Address address;
}
public interface PersonRepository extends CrudRepository<Person, String> {

}
@Configuration
@EnableRedisRepositories
public class ApplicationConfig {

  @Bean
  public RedisConnectionFactory connectionFactory() {
    return new JedisConnectionFactory();
  }

  @Bean
  public RedisTemplate<?, ?> redisTemplate(RedisConnectionFactory redisConnectionFactory) {

    RedisTemplate<byte[], byte[]> template = new RedisTemplate<byte[], byte[]>();
    template.setConnectionFactory(redisConnectionFactory);
    return template;
  }
}
@Autowired PersonRepository repo;

public void basicCrudOperations() {

  Person rand = new Person("rand", "al'thor");
  rand.setAddress(new Address("emond's field", "andor"));

  repo.save(rand);                                         

  repo.findOne(rand.getId());                              

  repo.count();                                            

  repo.delete(rand);                                       
}
```

## 对象映射基础

### 创建对象

Spring Data 会自动检测实体的构造函数。 解析算法的工作原理如下：

- 如果只有一个构造函数，则使用它。
- 如果有多个构造函数，并且只有一个用 @PersistenceConstructor 注释，则使用它。
- 如果有无参数构造函数，则使用它。 其他构造函数将被忽略。

值解析假定构造函数参数名称与实体的属性名称匹配，即解析将被执行。 这还需要类文件中可用的参数名称信息或构造函数中存在的 @ConstructorProperties 注释。

值解析可以通过使用 Spring Framework 的 @Value 值注释使用特定于store的 SpEL 表达式进行自定义。

为了避免反射的开销，Spring Data 在运行时生成对应的工厂类，它会直接调用域类构造函数。例如：

```java
class Person {
  Person(String firstname, String lastname) { … }
}
```

运行时创建工厂类：

```java
class PersonObjectInstantiator implements ObjectInstantiator {

  Object newInstance(Object... args) {
    return new Person((String) args[0], (String) args[1]);
  }
}
```

这使我们比反射提高了大约 10% 的性能。 对于实体类还有一些限制：

- 不能是私有类
- 不能是非静态内部类
- 不能是CGLib代理的类
- 构造函数不能是私有的

如果不能满足上面的任一条件，将使用反射。

### 属性填充

一旦创建了实体的实例，Spring Data 就会填充该类的所有剩余持久属性。 除非实体的构造函数已经填充（即通过其构造函数参数列表），首先填充标识符属性以解决循环对象引用。 之后，所有尚未由构造函数填充的非瞬态属性。 为此，我们使用以下算法：

- 如果属性是不可变的，但公开了 with... 方法（见下文），我们使用 with... 方法创建一个具有新属性值的新实体实例。
- 如果定义了属性访问（即通过 getter 和 setter 访问），我们将调用 setter 方法。
- 如果属性是可变的，我们直接设置字段。
- 如果属性是不可变的，我们将使用持久性操作（请参阅对象创建）使用的构造函数来创建实例的副本。
- 默认情况下，我们直接设置字段值。

与我们在对象构造中的优化类似，我们也使用 Spring Data 运行时生成的访问器类与实体实例进行交互。

```java
class Person {

  private final Long id;
  private String firstname;
  private @AccessType(Type.PROPERTY) String lastname;

  Person() {
    this.id = null;
  }

  Person(Long id, String firstname, String lastname) {
    // Field assignments
  }

  Person withId(Long id) {
    return new Person(id, this.firstname, this.lastame);
  }

  void setLastname(String lastname) {
    this.lastname = lastname;
  }
}
class PersonPropertyAccessor implements PersistentPropertyAccessor {

  private static final MethodHandle firstname;              //2

  private Person person;           // 1                         

  public void setProperty(PersistentProperty property, Object value) {

    String name = property.getName();

    if ("firstname".equals(name)) {
      firstname.invoke(person, (String) value);      //2       
    } else if ("id".equals(name)) {
      this.person = person.withId((Long) value);         //3   
    } else if ("lastname".equals(name)) {
      this.person.setLastname((String) value);       //4       
    }
  }
}
```

1. PropertyAccessor 持有底层对象的可变实例。
2. 默认情况下，Spring Data 使用字段访问来读取和写入属性值。 根据私有字段的可见性规则，MethodHandles 用于与字段交互。
3. 该类公开了一个用于设置标识符的 withId(...) 方法，例如 当一个实例被插入到数据存储中并且一个标识符已经生成时。 调用 withId(...) 创建一个新的 Person 对象。 所有后续突变都将在新实例中发生，而前一个则保持不变。
4. 使用属性访问,而不是 MethodHandles 的直接方法调用。

这使我们比反射提高了大约 25% 的性能。 但是域类也有一些限制：

- 类型不得位于 java 包下。
- 类型及其构造函数必须是public
- 作为内部类的类型必须是静态的。
- 使用的 Java 运行时必须允许在原始 ClassLoader 中声明类。 Java 9 和更新版本施加了某些限制。

默认情况下，Spring Data 会尝试使用生成的属性访问器，并在检测到限制时回退到基于反射的访问器。

```java
class Person {

  private final @Id Long id;               // 1                                 
  private final String firstname, lastname;    //2                             
  private final LocalDate birthday;
  private final int age;                          //3                          

  private String comment;                            //4                       
  private @AccessType(Type.PROPERTY) String remarks;    //5                    

  static Person of(String firstname, String lastname, LocalDate birthday) { //6

    return new Person(null, firstname, lastname, birthday,
      Period.between(birthday, LocalDate.now()).getYears());
  }

  Person(Long id, String firstname, String lastname, LocalDate birthday, int age) {  //6

    this.id = id;
    this.firstname = firstname;
    this.lastname = lastname;
    this.birthday = birthday;
    this.age = age;
  }

  Person withId(Long id) {                    //1                              
    return new Person(id, this.firstname, this.lastname, this.birthday, this.age);
  }

  void setRemarks(String remarks) {       //5                                  
    this.remarks = remarks;
  }
}
```

1. id是final并且在构造函数中设置为null。该类公开了一个用于设置标识符的 withId(...) 方法，例如 当一个实例被插入到数据存储中并且生成一个标识符。创建新实例时，原始 Person 实例保持不变。相同的模式通常应用于由存储管理但可能必须为持久性操作更改的其他属性。wither 方法是可选的，因为持久性构造函数（参见 6）实际上是一个复制构造函数，并且设置该属性将被转换为创建一个应用新标识符值的新实例。
2. firstname 和 lastname 属性是可能通过 getter 公开的普通不可变属性。
3. age 属性是不可变的，但从birthday 属性派生而来。使用所示设计，数据库值将胜过默认值，因为 Spring Data 使用唯一声明的构造函数。即使意图是计算应该是首选，重要的是此构造函数也将年龄作为参数（可能会忽略它），否则属性填充步骤将尝试设置年龄字段 。并由于它不可变且不存在with......方法存在 而失败 。
4. comment属性是可变的，可以直接设置。
5. remarks 属性是可变的，通过setter设置
6. 该类公开了一个工厂方法和一个用于创建对象的构造函数。这里的核心思想是使用工厂方法而不是额外的构造函数，避免通过@PersistenceConstructor 来消除构造函数歧义。相反，属性的默认设置是在工厂方法中处理的。

### 域对象建议

- 尽量使用不可变对象：仅构造函数实现比属性填充快 30%。
- 提供全参数的构造函数：即使您不能或不想将实体建模为不可变值，提供将实体的所有属性（包括可变属性）作为参数的构造函数仍然有价值，因为这允许对象映射跳过属性填充 以获得最佳性能。
- 使用工厂方法而不是重载构造函数来避免@PersistenceConstructor
- 确保实例化器和属性访问器遵守约束
- 对于要生成的标识符，仍然使用 final 字段与全参数持久性构造函数（首选）或 with... 方法的组合
- 使用 Lombok 避免样板代码

## Object-to-Hash

Redis 存储库支持将对象持久化为哈希。 这需要由 RedisConverter 完成Object-to-Hash 转换。例如上面的Person类，映射结果如下：

```java
_class = org.example.Person                 
id = e2c7dcee-b8cd-4424-883e-736ce564363e
firstname = rand                            
lastname = al’thor
address.city = emond's field                
address.country = andor
```

默认的映射规则

| Type                                | Sample                                                       | Mapped Value                                                 |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Simple Type (for example, String)   | String firstname = "rand";                                   | firstname = "rand"                                           |
| Byte array (`byte[]`)               | byte[] image = "rand".getBytes();                            | image = "rand"                                               |
| Complex Type (for example, Address) | Address address = new Address("emond’s field");              | address.city = "emond’s field"                               |
| List of Simple Type                 | List nicknames = asList("dragon reborn", "lews therin");     | nicknames.[0] = "dragon reborn", nicknames.[1] = "lews therin" |
| Map of Simple Type                  | Map<String, String> atts = asMap({"eye-color", "grey"}, {"…  | atts.[eye-color] = "grey", atts.[hair-color] = "…            |
| List of Complex Type                | List addresses = asList(new Address("em…                     | addresses.[0].city = "emond’s field", addresses.[1].city = "… |
| Map of Complex Type                 | Map<String, Address> addresses = asMap({"home", new Address("em… | addresses.[home].city = "emond’s field", addresses.[work].city = "… |

> 由于扁平表示结构，Map 键需要是简单的类型，例如 String 或 Number。

映射行为可以通过在 RedisCustomConversions 中注册相应的 Converter 来自定义。 这些转换器可以处理单个 byte[] 以及 Map<String,byte[]> 的转换。 第一个适用于（例如）将复杂类型转换为（例如）二进制 JSON 表示。 第二个选项提供对结果散列的完全控制。

> 将对象写入 Redis 散列会从散列中删除内容并重新创建整个散列，因此未映射的数据将丢失。

以下示例显示了两个示例字节数组转换器：

```java
@WritingConverter
public class AddressToBytesConverter implements Converter<Address, byte[]> {

  private final Jackson2JsonRedisSerializer<Address> serializer;

  public AddressToBytesConverter() {

    serializer = new Jackson2JsonRedisSerializer<Address>(Address.class);
    serializer.setObjectMapper(new ObjectMapper());
  }

  @Override
  public byte[] convert(Address value) {
    return serializer.serialize(value);
  }
}

@ReadingConverter
public class BytesToAddressConverter implements Converter<byte[], Address> {

  private final Jackson2JsonRedisSerializer<Address> serializer;

  public BytesToAddressConverter() {

    serializer = new Jackson2JsonRedisSerializer<Address>(Address.class);
    serializer.setObjectMapper(new ObjectMapper());
  }

  @Override
  public Address convert(byte[] value) {
    return serializer.deserialize(value);
  }
}
```

使用前面的字节数组 Converter 会产生类似于以下内容的输出：

```java
_class = org.example.Person
id = e2c7dcee-b8cd-4424-883e-736ce564363e
firstname = rand
lastname = al’thor
address = { city : "emond's field", country : "andor" }
```

以下示例显示了 Map 转换器的两个示例：

```java
@WritingConverter
public class AddressToMapConverter implements Converter<Address, Map<String,byte[]>> {

  @Override
  public Map<String,byte[]> convert(Address source) {
    return singletonMap("ciudad", source.getCity().getBytes());
  }
}

@ReadingConverter
public class MapToAddressConverter implements Converter<Map<String, byte[]>, Address> {

  @Override
  public Address convert(Map<String,byte[]> source) {
    return new Address(new String(source.get("ciudad")));
  }
}
```

使用前面的 Map Converter 会产生类似于以下内容的输出：

```java
_class = org.example.Person
id = e2c7dcee-b8cd-4424-883e-736ce564363e
firstname = rand
lastname = al’thor
ciudad = "emond's field"
```

### 自定义类型映射

如果你不想序列化的内容包含整个类名，而是希望使用一个key代替，你可以在类型上使用@TypeAlias注解。如果你想进一步定制化，可以参考TypeInformationMapper接口，默认实现是DefaultRedisTypeMapper（被MappingRedisConverter使用）。

```java
@TypeAlias("pers")
class Person {

}
```

生成的文档包含 pers 作为 _class 字段中的值。

以下示例演示了如何在 MappingRedisConverter 中配置自定义 RedisTypeMapper：

```java
class CustomRedisTypeMapper extends DefaultRedisTypeMapper {
  //implement custom type mapping here
}
@Configuration
class SampleRedisConfiguration {

  @Bean
  public MappingRedisConverter redisConverter(RedisMappingContext mappingContext,
        RedisCustomConversions customConversions, ReferenceResolver referenceResolver) {

    MappingRedisConverter mappingRedisConverter = new MappingRedisConverter(mappingContext, null, referenceResolver,
            customTypeMapper());

    mappingRedisConverter.setCustomConversions(customConversions);

    return mappingRedisConverter;
  }

  @Bean
  public RedisTypeMapper customTypeMapper() {
    return new CustomRedisTypeMapper();
  }
}
```

## Keyspaces

Keyspaces 为保存对象的redis hash定义前缀，前缀设置为 getClass().getName()。您可以通过在聚合根级别设置 @RedisHash 或设置编程配置来更改此默认值。 但是，@Keyspace覆盖其他配置。

以下示例显示如何使用 @EnableRedisRepositories 注解设置键空间配置：

```java
@Configuration
@EnableRedisRepositories(keyspaceConfiguration = MyKeyspaceConfiguration.class)
public class ApplicationConfig {

  //... RedisConnectionFactory and RedisTemplate Bean definitions omitted

  public static class MyKeyspaceConfiguration extends KeyspaceConfiguration {

    @Override
    protected Iterable<KeyspaceSettings> initialConfiguration() {
      return Collections.singleton(new KeyspaceSettings(Person.class, "people"));
    }
  }
}
```

以下示例显示了如何以编程方式设置键空间：

```java
@Configuration
@EnableRedisRepositories
public class ApplicationConfig {

  //... RedisConnectionFactory and RedisTemplate Bean definitions omitted

  @Bean
  public RedisMappingContext keyValueMappingContext() {
    return new RedisMappingContext(
      new MappingConfiguration(new IndexConfiguration(), new MyKeyspaceConfiguration()));
  }

  public static class MyKeyspaceConfiguration extends KeyspaceConfiguration {

    @Override
    protected Iterable<KeyspaceSettings> initialConfiguration() {
      return Collections.singleton(new KeyspaceSettings(Person.class, "people"));
    }
  }
}
```

## 二级索引

二级索引用于启用基于本机 Redis 结构的查找操作。 每次保存时都会将值写入相应的索引，并在对象被删除或过期时将其删除。

### 简单的属性索引

给定前面显示的示例 Person 实体，我们可以通过使用 @Indexed 注释属性来为 firstname 创建索引，如以下示例所示：

```java
@RedisHash("people")
public class Person {

  @Id String id;
  @Indexed String firstname;
  String lastname;
  Address address;
}
```

索引是为实际属性值建立的。 保存两个 Person（例如，“rand”和“aviendha”）会导致设置类似于以下内容的索引：

```
SADD people:firstname:rand e2c7dcee-b8cd-4424-883e-736ce564363e
SADD people:firstname:aviendha a9d4b3a0-50d3-4538-a2fc-f7fc2581ee56
```

嵌套元素也可以有索引。 假设Address有一个用@Indexed 注释的city属性。 在这种情况下，一旦 person.address.city 不为空，我们就有了每个city的 Sets，如下例所示：

```
SADD people:address.city:tear e2c7dcee-b8cd-4424-883e-736ce564363e
```

此外，可以对map和list属性创建索引，如以下示例所示：

```java
@RedisHash("people")
public class Person {

  // ... other properties omitted

  Map<String,String> attributes;      
  Map<String Person> relatives;       
  List<Address> addresses;            
}
SADD people:attributes.map-key:map-value e2c7dcee-b8cd-4424-883e-736ce564363e
SADD people:relatives.map-key.firstname:tam e2c7dcee-b8cd-4424-883e-736ce564363e
SADD people:addresses.city:tear e2c7dcee-b8cd-4424-883e-736ce564363e
```

与键空间一样，您可以配置索引而无需注释实际域类型，如以下示例所示：

```java
@Configuration
@EnableRedisRepositories(indexConfiguration = MyIndexConfiguration.class)
public class ApplicationConfig {

  //... RedisConnectionFactory and RedisTemplate Bean definitions omitted

  public static class MyIndexConfiguration extends IndexConfiguration {

    @Override
    protected Iterable<IndexDefinition> initialConfiguration() {
      return Collections.singleton(new SimpleIndexDefinition("people", "firstname"));
    }
  }
}
```

同样，与键空间一样，您可以以编程方式配置索引，如以下示例所示：

```java
@Configuration
@EnableRedisRepositories
public class ApplicationConfig {

  //... RedisConnectionFactory and RedisTemplate Bean definitions omitted

  @Bean
  public RedisMappingContext keyValueMappingContext() {
    return new RedisMappingContext(
      new MappingConfiguration(
        new KeyspaceConfiguration(), new MyIndexConfiguration()));
  }

  public static class MyIndexConfiguration extends IndexConfiguration {

    @Override
    protected Iterable<IndexDefinition> initialConfiguration() {
      return Collections.singleton(new SimpleIndexDefinition("people", "firstname"));
    }
  }
}
```

### Geospatial 索引

假设 Address 类型包含 Point 类型的 location 属性，该属性保存特定地址的地理坐标。 通过使用 @GeoIndexed 注释属性，Spring Data Redis 使用 Redis GEO 命令添加这些值，如以下示例所示

```java
@RedisHash("people")
public class Person {

  Address address;

  // ... other properties omitted
}

public class Address {

  @GeoIndexed Point location;

  // ... other properties omitted
}

public interface PersonRepository extends CrudRepository<Person, String> {

  List<Person> findByAddressLocationNear(Point point, Distance distance);     
  List<Person> findByAddressLocationWithin(Circle circle);                    
}

Person rand = new Person("rand", "al'thor");
rand.setAddress(new Address(new Point(13.361389D, 38.115556D)));

repository.save(rand);                                                        

repository.findByAddressLocationNear(new Point(15D, 37D), new Distance(200)); 
GEOADD people:address:location 13.361389 38.115556 e2c7dcee-b8cd-4424-883e-736ce564363e
GEORADIUS people:address:location 15.0 37.0 200.0 km
```

## Query by Example

Query by Example API 包含三部分：

- Probe：域对象
- ExampleMatcher：ExampleMatcher 包含有关如何匹配特定字段的详细信息。 它可以在多个示例中重复使用。
- Example：由Probe和ExampleMatcher组成，用于创建查询。

Query by Example 非常适合以下几个用例：

- 使用一组静态或动态约束查询您的数据存储。
- 频繁重构域对象而不必担心破坏现有查询。
- 独立于底层数据存储 API 工作。

Query by Example 也有几个限制：

- 不支持嵌套或分组的属性约束，例如 `firstname = ?0 or (firstname = ?1 and lastname = ?2)`。
- 仅支持字符串的开始/包含/结束/正则表达式匹配以及其他属性类型的精确匹配。

### ExampleMatcher

您可以使用 ExampleMatcher 为字符串匹配、空处理和特定于属性的设置指定自己的默认值，如以下示例所示：

```java
Person person = new Person();                          
person.setFirstname("Dave");                           

ExampleMatcher matcher = ExampleMatcher.matching()     
  .withIgnorePaths("lastname")                         
  .withIncludeNullValues()                             
  .withStringMatcher(StringMatcher.ENDING);            

Example<Person> example = Example.of(person, matcher); 
```

默认情况下， ExampleMatcher 期望在探测器上设置的所有值都匹配。 如果您只想满足部分条件，请使用 ExampleMatcher.matchingAny()。

您可以为单个属性指定行为，如以下示例所示：

```java
ExampleMatcher matcher = ExampleMatcher.matching()
  .withMatcher("firstname", endsWith())
  .withMatcher("lastname", startsWith().ignoreCase());
}
```

另一种配置匹配器选项的方法是使用 lambdas（在 Java 8 中引入）:

```java
ExampleMatcher matcher = ExampleMatcher.matching()
  .withMatcher("firstname", match -> match.endsWith())
  .withMatcher("firstname", match -> match.startsWith());
}
```

### 使用

```java
interface PersonRepository extends QueryByExampleExecutor<Person> {
}

class PersonService {

  @Autowired PersonRepository personRepository;

  List<Person> findPeople(Person probe) {
    return personRepository.findAll(Example.of(probe));
  }
}
```

Redis 存储库及其二级索引支持Query by Example的部分功能。 特别是，仅使用精确、区分大小写和非空值来构造查询。

二级索引使用基于集合的操作（集合交集、集合并集）来确定匹配的键。 向未编入索引的查询添加属性不会返回任何结果，因为不存在索引。 Query by Example 支持检查索引配置以仅包含查询中由索引覆盖的属性。 这是为了防止意外包含非索引属性。

不区分大小写的查询和不受支持的 StringMatcher 实例在运行时被拒绝。以下列表显示了支持的示例查询选项：

- 区分大小写，简单和嵌套属性的精确匹配
- Any/All匹配模式
- 标准值的值转换
- 从条件中排除空值

以下列表显示了 Query by Example 不支持的属性：

- 不区分大小写的匹配
- Regex, prefix/contains/suffix String-matching
- 查询关联、集合和类似Map的属性
- 从条件中包含空值
- findAll 排序

## 生存时间

存储在 Redis 中的对象可能仅在一定时间内有效。 这对于在 Redis 中持久化短期对象特别有用，而无需在它们达到生命周期结束时手动删除它们。 可以使用 @RedisHash(timeToLive=… ) 以及使用 KeyspaceSettings（参见 Keyspaces）设置以秒为单位的过期时间。

可以通过在数字属性或方法上使用 @TimeToLive 注释来设置更灵活的到期时间。 但是，不要在同一类中的方法和属性上同时应用 @TimeToLive。 以下示例显示了属性和方法上的 @TimeToLive 注释：

```java
public class TimeToLiveOnProperty {

  @Id
  private String id;

  @TimeToLive
  private Long expiration;
}

public class TimeToLiveOnMethod {

  @Id
  private String id;

  @TimeToLive
  public long getTimeToLive() {
  	return new Random().nextLong();
  }
}
```

使用 @TimeToLive 显式注释属性会从 Redis 读回实际的 TTL 或 PTTL 值。 -1 表示对象没有关联的过期时间。

存储库实现确保通过 RedisMessageListenerContainer 订阅 Redis key空间通知。

当到期时间设置为正值时，将运行相应的 EXPIRE 命令。 除了保留原始副本外，Redis 中还保留了一个幻影副本，并设置为在原始副本之后五分钟过期。 这样做是为了让 Repository 支持发布 RedisKeyExpiredEvent，只要一个键过期，就会在 Spring 的 ApplicationEventPublisher 中持有过期的值，即使原始值已经被删除。 在使用 Spring Data Redis 存储库的所有连接的应用程序上接收到期事件。

默认情况下，初始化应用程序时禁用key过期侦听器。 可以在 @EnableRedisRepositories 或 RedisKeyValueAdapter 中调整启动模式，以使用应用程序或在第一次插入具有 TTL 的实体时启动侦听器。 有关可能的值，请参阅 EnableKeyspaceEvents。

RedisKeyExpiredEvent 保存过期域对象的副本以及key。

延迟或禁用到期事件侦听器启动会影响 RedisKeyExpiredEvent 发布。 禁用的事件侦听器不会发布到期事件。 由于侦听器初始化延迟，延迟启动可能会导致事件丢失。

keyspace 通知消息侦听器会更改 Redis 中的 notify-keyspace-events 设置（如果尚未设置）。 现有设置不会被覆盖，因此您必须正确设置这些设置（或将它们留空）。 请注意，AWS ElastiCache 上禁用了 CONFIG，启用侦听器会导致错误。

Redis Pub/Sub 消息不是持久的。 如果在应用程序关闭时某个键过期，则不会处理过期事件，这可能会导致二级索引包含对过期对象的引用。

@EnableKeyspaceEvents(shadowCopy = OFF) 禁用虚拟副本的存储并减少 Redis 中的数据大小。 RedisKeyExpiredEvent 将只包含过期键的 id。

## @Reference

使用 @Reference 标记属性允许存储一个简单的键引用，而不是将值复制到哈希本身中。 从 Redis 加载时，引用会自动解析并映射回对象，如以下示例所示：

```java
_class = org.example.Person
id = e2c7dcee-b8cd-4424-883e-736ce564363e
firstname = rand
lastname = al’thor
mother = people:a9d4b3a0-50d3-4538-a2fc-f7fc2581ee56  
```

Reference 存储被引用对象的整个键（keyspace:id）。

> 保存引用对象的ID,还需要手动保留引用对象，redis不会自动操作。

## 部分更新

在某些情况下，您不需要加载和重写整个实体，只修改部分属性。例如修改session的访问时间。 PartialUpdate 允许您在现有对象上定义设置和删除操作，同时负责更新实体和索引结构的过期时间。 以下示例显示了部分更新：

```java
PartialUpdate<Person> update = new PartialUpdate<Person>("e2c7dcee", Person.class)
  .set("firstname", "mat")                                                           
  .set("address.city", "emond's field")                                              
  .del("age");                                                                       

template.update(update);

update = new PartialUpdate<Person>("e2c7dcee", Person.class)
  .set("address", new Address("caemlyn", "andor"))                                   
  .set("attributes", singletonMap("eye-color", "grey"));                             

template.update(update);

update = new PartialUpdate<Person>("e2c7dcee", Person.class)
  .refreshTtl(true);                                                                 
  .set("expiration", 1000);

template.update(update);
```

## 查询方法

查询方法允许从方法名称自动派生简单的查找器查询，如以下示例所示：

```java
public interface PersonRepository extends CrudRepository<Person, String> {

  List<Person> findByFirstname(String firstname);
}
```

> 请确保在 finder 方法中使用的属性设置为索引。

> Redis 存储库的查询方法仅支持查询和分页查询实体。

使用派生查询方法可能并不总是足以对要运行的查询进行建模。 RedisCallback 提供了对索引结构甚至自定义索引的实际匹配的更多控制。 为此，请提供一个 RedisCallback 返回一组或可迭代的 id 值，如以下示例所示：

```java
String user = //...

List<RedisSession> sessionsByUser = template.find(new RedisCallback<Set<byte[]>>() {

  public Set<byte[]> doInRedis(RedisConnection connection) throws DataAccessException {
    return connection
      .sMembers("sessions:securityContext.authentication.principal.username:" + user);
  }}, RedisSession.class);
```

下表概述了 Redis 支持的关键字，以及包含该关键字的方法的基本含义：

|              |                                                              |                                              |
| ------------ | ------------------------------------------------------------ | -------------------------------------------- |
| Keyword      | Sample                                                       | Redis snippet                                |
| `And`        | `findByLastnameAndFirstname`                                 | `SINTER …:firstname:rand …:lastname:al’thor` |
| `Or`         | `findByLastnameOrFirstname`                                  | `SUNION …:firstname:rand …:lastname:al’thor` |
| `Is, Equals` | `findByFirstname`, `findByFirstnameIs`, `findByFirstnameEquals` | `SINTER …:firstname:rand`                    |
| `IsTrue`     | `FindByAliveIsTrue`                                          | `SINTER …:alive:1`                           |
| `IsFalse`    | `findByAliveIsFalse`                                         | `SINTER …:alive:0`                           |
| `Top,First`  | `findFirst10ByFirstname`,`findTop5ByFirstname`               |                                              |

Redis 存储库允许使用各种方法来定义排序顺序。 Redis 本身在检索散列或集合时不支持动态排序。 因此，Redis 存储库查询方法构造了一个 Comparator，该 Comparator 在将结果作为 List 返回之前应用于结果。 让我们看一下下面的例子：

```java
interface PersonRepository extends RedisRepository<Person, String> {

  List<Person> findByFirstnameOrderByAgeDesc(String firstname); 

  List<Person> findByFirstname(String firstname, Sort sort);   
}
```

## 在集群上运行的 Redis 存储库

您可以在集群 Redis 环境中使用 Redis 存储库支持。下表显示了集群上的数据详细信息（基于前面的示例）：

| Key                                         | Type        | Slot  | Node           |
| ------------------------------------------- | ----------- | ----- | -------------- |
| people:e2c7dcee-b8cd-4424-883e-736ce564363e | id for hash | 15171 | 127.0.0.1:7381 |
| people:a9d4b3a0-50d3-4538-a2fc-f7fc2581ee56 | id for hash | 7373  | 127.0.0.1:7380 |
| people:firstname:rand                       | index       | 1700  | 127.0.0.1:7379 |

某些命令（例如 SINTER 和 SUNION）只有在所有涉及的键都映射到同一个槽时才能在服务器端处理。 否则，必须在客户端进行计算。 因此，将键空间固定到单个插槽非常有用，这让我们可以立即使用 Redis 服务器端计算。 下表显示了当您执行此操作时会发生什么：

| Key                                           | Type        | Slot | Node           |
| --------------------------------------------- | ----------- | ---- | -------------- |
| {people}:e2c7dcee-b8cd-4424-883e-736ce564363e | id for hash | 2399 | 127.0.0.1:7379 |
| {people}:a9d4b3a0-50d3-4538-a2fc-f7fc2581ee56 | id for hash | 2399 | 127.0.0.1:7379 |
| {people}:firstname:rand                       | index       | 2399 | 127.0.0.1:7379 |

使用 Redis 集群时，通过使用 @RedisHash("{yourkeyspace}") 定义键空间并将其固定到特定插槽。

## 存储库底层细节

```java
@RedisHash("people")
public class Person {

  @Id String id;
  @Indexed String firstname;
  String lastname;
  Address hometown;
}

public class Address {

  @GeoIndexed Point location;
}

repository.save(new Person("rand", "al'thor"));
```

redis操作：

```java
HMSET "people:19315449-cda2-4f5c-b696-9cb8018fa1f9" "_class" "Person" "id" "19315449-cda2-4f5c-b696-9cb8018fa1f9" "firstname" "rand" "lastname" "al'thor" 
SADD  "people" "19315449-cda2-4f5c-b696-9cb8018fa1f9"                           
SADD  "people:firstname:rand" "19315449-cda2-4f5c-b696-9cb8018fa1f9"            
# 以跟踪要在删除/更新时清理的索引
SADD  "people:19315449-cda2-4f5c-b696-9cb8018fa1f9:idx" "people:firstname:rand" 
```

替换实体：

```java
repository.save(new Person("e82908cf-e7d3-47c2-9eec-b4e0967ad0c9", "Dragon Reborn", "al'thor"));
# 直接覆盖hash
DEL       "people:e82908cf-e7d3-47c2-9eec-b4e0967ad0c9"                           
HMSET     "people:e82908cf-e7d3-47c2-9eec-b4e0967ad0c9" "_class" "Person" "id" "e82908cf-e7d3-47c2-9eec-b4e0967ad0c9" "firstname" "Dragon Reborn" "lastname" "al'thor" 
SADD      "people" "e82908cf-e7d3-47c2-9eec-b4e0967ad0c9"                     

# 移除变更的索引
SMEMBERS  "people:e82908cf-e7d3-47c2-9eec-b4e0967ad0c9:idx"  

TYPE      "people:firstname:rand"                          
# 删除旧索引和实体的关联
SREM      "people:firstname:rand" "e82908cf-e7d3-47c2-9eec-b4e0967ad0c9"          
DEL       "people:e82908cf-e7d3-47c2-9eec-b4e0967ad0c9:idx"   

# 添加第二索引
SADD      "people:firstname:Dragon Reborn" "e82908cf-e7d3-47c2-9eec-b4e0967ad0c9" 
SADD      "people:e82908cf-e7d3-47c2-9eec-b4e0967ad0c9:idx" "people:firstname:Dragon Reborn"
```

简单查找

```java
repository.findByFirstname("egwene");
SINTER  "people:firstname:egwene"                     
HGETALL "people:d70091b5-0b9a-4c0a-9551-519e61bc9ef3"
```
