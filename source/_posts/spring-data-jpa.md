---
title: spring data jpa
tags: 
 - spring-data
 - jpa
categories:
 - spring
---

#  Spring Data Repositories

Spring Data Repositories的目的就是减少数据访问层的样本代码,提高开发效率

## 核心概念

Spring Data Repositories抽象的核心接口是Repository(此接口是一个空接口,主要用作标记)。 它将域类以及域类的ID类型作为类型参数进行管理。CrudRepository是其扩展接口,为正在管理的实体类提供复杂的CRUD功能。

```java
public interface CrudRepository<T, ID extends Serializable> extends Repository<T, ID> {
  <S extends T> S save(S entity);
  Optional<T> findById(ID primaryKey);
  Iterable<T> findAll();
  long count();
  void delete(T entity);
  boolean existsById(ID primaryKey);
  // … more functionality omitted.
}
```

> 我们还提供特定于某项持久化技术的抽象，例如JpaRepository或MongoRepository。这些接口扩展了CrudRepository的功能.

在CrudRepository之上,我们还提供了基于分页的接口:

```java
public interface PagingAndSortingRepository<T, ID extends Serializable> extends CrudRepository<T, ID> {
  Iterable<T> findAll(Sort sort);
  Page<T> findAll(Pageable pageable);
}
```

例如,我们要查询第二页数据,可以这样使用

```java
PagingAndSortingRepository<User, Long> repository = // … get access to a bean
Page<User> users = repository.findAll(new PageRequest(1, 20));
```

除了这些查询之外,我们还可以使用基于删除或者统计的隐含查询,例如:

```java
interface UserRepository extends CrudRepository<User, Long> {
  long deleteByLastname(String lastname);
  List<User> removeByLastname(String lastname);
  long countByLastname(String lastname);
}
```

## 快速入门

1. 开启JPA注解

```java
@EnableJpaRepositories
class Config {}
```

>  @EnableJpaRepositories:开启JPA支持,默认扫描标注该注解的类所在包的子包中的Repository接口,可以通过basePackage自定义扫描包

2. 声明Repository接口

```java
public interface UserRepository extends Repository<UserEntity, Integer> {
    UserEntity findById(int id);
}
```

3. 测试

```java
	@Autowired
	private UserRepository userRepository;
	@Test
	public void contextLoads() {
		UserEntity entity = userRepository.findById(1);
		System.err.println(entity);
	}
```

## 定义Repository接口

自定义的Repository需要扩展Repository接口或其子接口,并指定域对象和域ID.上面的快速入门,我们已经演示,我们还可以通过注解的方式来定义Repository接口,例如:

```java
@RepositoryDefinition(domainClass = UserEntity.class,idClass = Integer.class)
public interface User2Repository {
    UserEntity findById(int id);
}
```

CrudRepository接口中提供了很多操作底层数据的方法,但很多时候,我们只想暴露其中的一部分方法,我们可以定义一个 baseRepository接口,在里面声明我们想暴露的方法,例如:

```java
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {
  Optional<T> findById(ID id);
  <S extends T> S save(S entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
```

> @NoRepositoryBean:该注解删除也不会影响使用,他的作用是在运行时不要创建该接口的代理实例.

> 😫 spring data查询返回单个实体时,返回的是Optional包装的对象,如果不使用Optional包装的话,返回null.当查询多个实体时,返回的是空的集合,而不是null.

### spring的运行时非空检验

`package-info.java`

```java
@org.springframework.lang.NonNullApi
package com.acme;
```

> 要想开启非空检验,必须在包上声明@NonNullApi

具体类上的声明：

```java
interface UserRepository extends Repository<User, Long> {

  User getByEmailAddress(EmailAddress emailAddress); //查询结果为空,抛出EmptyResultDataAccessException

  @Nullable
  User findByEmailAddress(@Nullable EmailAddress emailAdress);  //允许查询结果为空

  Optional<User> findOptionalByEmailAddress(EmailAddress emailAddress); //参数为空,抛出IllegalArgumentException
}
```

### 多模块Repository

有时，应用程序需要使用多个Spring Data模块。 在这种情况下，Repository需要区分持久化技术。 当它在类路径上检测到多个Repository时，Spring Data进入严格的Repository配置模式。 严格配置使用Repository或域类的详细信息来区分：

1. repository扩展自特定模块,例如JpaRepository
2. 域类上面标识了特定模块的注解,例如@Entity

虽然上面的两种方式可以有效帮我们区分具体的持久化技术,但是并不是万能的.为了区分不同的repository,可以使用下面的方式:

```java
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo")
interface Configuration { }
```

## 定义查询方法

### 查询策略

spring data中有两种查询方式: 

* 通过解析方法名称构建查询语句

* 自定义查询语句

而所谓的查询策略就是选择上面的哪一种.通过使用@Enable${store}Repositories的query-lookup-strategy属性来指定查询策略,查询策略分为三种:

1. CREATE :使用方法查询
2. USE_DECLARED_QUERY:使用声明的查询语句查询,找不到则抛出异常
3. CREATE_IF_NOT_FOUND :先找声明语句,找不到使用方法查询,系统默认.

### 方法查询

通过剥离方法上的关键字来构建查询语句,例如find…By, read…By, query…By, count…By,下面是具体的列子

```java
  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
```

1. 表达式通常是属性遍历与可以连接的运算符相结合.您可以将属性表达式与AND和OR组合使用。对于属性表达式，您还可以获得诸如Between，LessThan，GreaterThan和Like之类的运算符的支持。
2. 方法解析器支持为各个属性设置IgnoreCase标志（例如，findByLastnameIgnoreCase）或支持忽略所有属性大小写(findByLastnameAndFirstnameAllIgnoreCase（…））。
3. 您可以通过将OrderBy子句附加到引用属性的查询方法并提供排序方向（Asc或Desc）来应用静态排序。

属性表达式只能引用被管实体的直接属性，如前面的示例所示。 在创建查询时，您已确保已解析的属性是托管域类的属性。 但是，您也可以通过遍历嵌套属性来定义约束。看下面的例子:

```java
List<Person> findByAddressZipCode(ZipCode zipCode);
```

假如Persion包含属性Address,Address包含属性ZipCode.该方法创建属性遍历x.address.zipCode。

1. 解析算法首先将整个部分（AddressZipCode）解释为属性，并检查域类中是否具有该名称的属性（未大写）。如果算法成功，则使用该属性。
2. 如果没有，算法使用驼峰法则从右侧分成头部和尾部，并尝试查找相应的属性 - 在我们的示例中，AddressZip和Code 
3. 如果算法找到具有该头部的属性，则它采用尾部并继续从那里构建树，以刚刚描述的方式将尾部分开 
4. 如果第一个分割不匹配，算法会将分割点移动到左侧（Address，ZipCode）并继续。

虽然这应该适用于大多数情况，但算法可能会选择错误的属性。假设Person类也有一个addressZip属性。算法将在第一个拆分轮中匹配，选择错误的属性，然后失败（因为addressZip的类型可能没有code属性）。

要解决这种歧义，可以在方法名称中使用`_`来手动定义遍历点。 所以我们的方法名称如下：

```java
List<Person> findByAddress_ZipCode(ZipCode zipCode);
```

因为我们将下划线字符视为保留字符，所以我们强烈建议遵循标准Java命名约定（即，不在属性名称中使用下划线，而是使用camel case）。

除了在方法名称上做一些限制之外,我们还可以在方法参数上使用限制条件,例如:

```java
Page<User> findByLastname(String lastname, Pageable pageable);

Slice<User> findByLastname(String lastname, Pageable pageable);

List<User> findByLastname(String lastname, Sort sort);

List<User> findByLastname(String lastname, Pageable pageable);
```

#### 查询结果

查询方法的结果可以通过使用first或top关键字来限制，这些关键字可以互换使用。 可选的数值可以附加到top或first，以指定要返回的最大结果大小。如果省略该数字，则假定结果大小为1。 以下示例显示如何限制查询大小

```java
User findFirstByOrderByLastnameAsc();

User findTopByOrderByAgeDesc();

Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);

Slice<User> findTop3ByLastname(String lastname, Pageable pageable);

List<User> findFirst10ByLastname(String lastname, Sort sort);

List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

**java8 流式结果**

```java
@Query("select u from User u")
Stream<User> findAllByCustomQueryAndStream();

Stream<User> readAllByFirstnameNotNull();

@Query("select u from User u")
Stream<User> streamAllPaged(Pageable pageable);
```

**异步查询结果**

```java
@Async
Future<User> findByFirstname(String firstname);

@Async
CompletableFuture<User> findOneByFirstname(String firstname);

@Async
ListenableFuture<User> findOneByLastname(String lastname);
```

## 自定义Repository

有的时候,spring data 提供的Repository不能满足我们的需求,需要我们提供自定义的扩展,自定义需要下面几步

1.定义接口

```java
public interface CustomizedUserRepository {
    void someCustomMethod(UserEntity user);
}
```

2.定义实现

```java
public class CustomizedUserRepositoryImpl implements CustomizedUserRepository {
    @Override
    public void someCustomMethod(UserEntity user) {
        System.err.println("自定义的实现类");
    }
}
```

> 类名必须以Impl结尾,如果想自己设定后缀，需要修改@EnableJpaRepositories的repositoryImplementationPostfix属性 

3.使用

```java
public interface User3Repository extends CrudRepository<UserEntity,Integer> , CustomizedUserRepository{

}
```

4.测试

```java
user3Repository.someCustomMethod(new UserEntity());
```

🥰需要注意的时,我们自定义的会与系统的方法重名,这时候优先选择自定义的

```java
interface CustomizedSave<T> {
  <S extends T> S save(S entity);
}

class CustomizedSaveImpl<T> implements CustomizedSave<T> {
  public <S extends T> S save(S entity) {
    // Your custom implementation
  }
}
interface UserRepository extends CrudRepository<User, Long>, CustomizedSave<User> {
}

interface PersonRepository extends CrudRepository<Person, Long>, CustomizedSave<Person> {
}
```

如果自定义两个接口有相同的方法,同时继承这两个接口调用该方法的时候,按照声明的顺序优先调用.

当您要自定义基本Repository行为以便所有存储库都受到影响时,可以创建一个扩展特定于持久性技术的存储库基类的实现。 然后，此类充当存储库代理的自定义基类，如以下示例所示：

```java
class MyRepositoryImpl<T, ID extends Serializable>
  extends SimpleJpaRepository<T, ID> {

  private final EntityManager entityManager;

  MyRepositoryImpl(JpaEntityInformation entityInformation,
                          EntityManager entityManager) {
    super(entityInformation, entityManager);

    // Keep the EntityManager around to used from the newly introduced methods.
    this.entityManager = entityManager;
  }

  @Transactional
  public <S extends T> S save(S entity) {
    // implementation goes here
  }
}
@Configuration
@EnableJpaRepositories(repositoryBaseClass = MyRepositoryImpl.class)
class ApplicationConfiguration { … }
```

##  聚合根发布事件

存储库管理的实体是聚合根。 在域驱动设计应用程序中，这些聚合根通常发布域事件。 Spring Data 提供了一个名为 @DomainEvents 的注释，您可以在聚合根的方法上使用该注释，以使该发布尽可能简单，如以下示例所示：

```java
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString(exclude = "domainEvents")
public class Person  {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private Integer gender;//1:male;2:female

    @DomainEvents
    Collection<Object> domainEvents() {
        List<Object> events= new ArrayList<Object>();
        events.add(new PersonSavedEvent(this.id,this.gender));
        return events;
    }

    @AfterDomainEventPublication
    void callbackMethod() {
        //
    }

}
@Component
public class GenderStatProcessor {
    @Autowired
    GenderRepository genderRepository;

    @Async
    @TransactionalEventListener
    public void handleAfterPersonSavedComplete(PersonSavedEvent event){

        GenderStat genderStat = genderRepository.findOne(1l);
        if(event.getGender()==1){
            genderStat.setMaleCount(genderStat.getMaleCount()+1);
        }else {
            genderStat.setFemaleCount(genderStat.getFemaleCount()+1);
        }
        genderRepository.save(genderStat);
    }
}
```

每次调用 Spring Data 存储库 save(…)、saveAll(…)、delete(…) 或 delete All(…) 方法之一时都会调用这些方法。

## spring data扩展

### Querydsl

Querydsl是一个框架，可以通过其流畅的API构建静态类型的SQL类查询。

几个Spring Data模块通过QuerydslPredicateExecutor与Querydsl的集成，如以下示例所示：

```java
public interface QuerydslPredicateExecutor<T> {
  Optional<T> findById(Predicate predicate);
  Iterable<T> findAll(Predicate predicate);
  long count(Predicate predicate);
  boolean exists(Predicate predicate);
  // … more functionality omitted.
}
```

要使用Querydsl支持，请在存储库接口上扩展QuerydslPredicateExecutor，如以下示例所示:

```java
interface UserRepository extends CrudRepository<User, Long>, QuerydslPredicateExecutor<User> {
}
```

使用如下:

```java
Predicate predicate = user.firstname.equalsIgnoreCase("dave")
  .and(user.lastname.startsWithIgnoreCase("mathews"));
userRepository.findAll(predicate);
```

### web支持

开启

```java
@Configuration
@EnableWebMvc
@EnableSpringDataWebSupport
class WebConfiguration {}
```

@EnableSpringDataWebSupport作用如下:

1. 注册DomainClassConverter,让Spring MVC从请求参数或路径变量中解析存储库管理的域类的实例。
2. HandlerMethodArgumentResolver实现让Spring MVC从请求参数中解析Pageable和Sort实例。

```java
@Controller
@RequestMapping("/users")
class UserController {
  @RequestMapping("/{id}")
  String showUserForm(@PathVariable("id") User user, Model model) {
    model.addAttribute("user", user);
    return "userForm";
  }
}
```

如您所见，该方法直接接收User实例，无需进一步查找。可以通过让SpringMVC首先将路径变量转换为域类的id类型来解析实例，并最终通过在为域类型注册的存储库实例上调用findById（…）来访问实例。

```java
@Controller
@RequestMapping("/users")
class UserController {
  private final UserRepository repository;
  UserController(UserRepository repository) {
    this.repository = repository;
  }
  @RequestMapping
  String showUsers(Model model, Pageable pageable) {
    model.addAttribute("users", repository.findAll(pageable));
    return "users";
  }
}
```

# JPA使用

##  非spring boot配置JPA

```java
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
class ApplicationConfig {

  @Bean
  public DataSource dataSource() {
    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    return builder.setType(EmbeddedDatabaseType.HSQL).build();
  }

  @Bean
  public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    vendorAdapter.setGenerateDdl(true);
    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setJpaVendorAdapter(vendorAdapter);
    factory.setPackagesToScan("com.acme.domain"); //设置域对象
    factory.setDataSource(dataSource());
    return factory;
  }

  @Bean
  public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
    JpaTransactionManager txManager = new JpaTransactionManager();
    txManager.setEntityManagerFactory(entityManagerFactory);
    return txManager;
  }
}
```

### 启动模式

默认情况下，Spring Data JPA repository 注册为Spring bean。 它们是单例并被early初始化。 在启动期间，他们已经与JPA EntityManager交互以进行验证和元数据分析。 Spring Framework支持在后台线程中初始化JPA EntityManagerFactory，因为该进程通常在Spring应用程序中占用大量的启动时间。 为了有效地利用后台初始化，我们需要确保尽可能晚地初始化JPA repository。

可以配置@EnableJpaRepositories的BootstrapMode指定加载模式: DEFAULT ,LAZY, DEFERRED :

除非使用@Lazy明确注解，否则将early实例化repository。如果没有客户端bean依赖repository实例，那么懒加载才会生效。

LAZY隐式声明所有存储库bean都是惰性的，客户端bean依赖的repository实例也是惰性的。这意味着，如果客户端bean只是将repository实例赋予字段中而不是在初始化期间使用，则不会实例化repository。初始化实例发生首次交互时。

DEFERRED基本上与LAZY具有相同的操作模式，但是在ContextRefreshedEvent事件触发repository初始化，以便在应用程序完全启动之前验证repository。 如果您异步引导JPA，不要使用default模式

如果您异步引导JPA，DEFERRED是一个合理的默认值，因为它将确保Spring Data JPA引导程序仅等待EntityManagerFactory设置，如果EntityManagerFactory本身比初始化所有其他应用程序组件花费更长时间。尽管如此，它确保在应用程序启动之前正确初始化和验证存储库。

LAZY是测试场景和本地开发的不错选择。

## 持久化实体

可以使用CrudRepository.save（…）方法执行保存实体。 它通过使用JPA EntityManager持久化或合并给定实体。 如果实体尚未持久化，则Spring Data JPA会通过调用entityManager.persist（…）方法来保存实体。 否则，它调用entityManager.merge（…）方法。

Spring Data JPA提供以下策略来检测实体是否是新实体： . Id-Property检查（默认）：默认情况下，Spring Data JPA检查给定实体的identifier属性。如果identifier属性为null，则假定该实体是新的。否则，它被认为不是新的。 . 实现Persistable：如果实体实现了Persistable，Spring Data JPA会将检新委托给实体的isNew（…）方法。 . 实现EntityInformation：您可以通过创建JpaRepositoryFactory的子类并相应地重写getEntityInformation（…）方法来自定义SimpleJpaRepository实现中使用的EntityInformation抽象。然后，您必须将JpaRepositoryFactory的自定义实现注册为Spring bean。请注意，通常没必要这么做。

## 查询方法

###  查询策略

查询实体有三种方式: query method ,named query和query

### query method

```java
public interface UserRepository extends Repository<User, Long> {

  List<User> findByEmailAddressAndLastname(String emailAddress, String lastname);
}
```

我们使用JPA标准API创建一个查询，但实质上，这转换为以下查询：`select u from User u where u.emailAddress = ?1 and u.lastname = ?2`

| Keyword           | Sample                                                  | JPQL snippet                                                 |
| ----------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| And               | findByLastnameAndFirstname                              | … where x.lastname = ?1 and x.firstname = ?2                 |
| Or                | findByLastnameOrFirstname                               | … where x.lastname = ?1 or x.firstname = ?2                  |
| Is,Equals         | findByFirstname,findByFirstnameIs,findByFirstnameEquals | … where x.firstname = ?1                                     |
| Between           | findByStartDateBetween                                  | … where x.startDate between ?1 and ?2                        |
| LessThan          | findByAgeLessThan                                       | … where x.age < ?1                                           |
| LessThanEqual     | findByAgeLessThanEqual                                  | … where x.age ⇐ ?1                                           |
| GreaterThan       | findByAgeGreaterThan                                    | … where x.age > ?1                                           |
| GreaterThanEqual  | findByAgeGreaterThanEqual                               | … where x.age >= ?1                                          |
| After             | findByStartDateAfter                                    | … where x.startDate > ?1                                     |
| Before            | findByStartDateBefore                                   | … where x.startDate < ?1                                     |
| IsNull            | findByAgeIsNull                                         | … where x.age is null                                        |
| IsNotNull,NotNull | findByAge(Is)NotNull                                    | … where x.age not null                                       |
| Like              | findByFirstnameLike                                     | … where x.firstname like ?1                                  |
| NotLike           | findByFirstnameNotLike                                  | … where x.firstname not like ?1                              |
| StartingWith      | findByFirstnameStartingWith                             | … where x.firstname like ?1 (parameter bound with appended %) |
| EndingWith        | findByFirstnameEndingWith                               | … where x.firstname like ?1 (parameter bound with prepended %) |
| Containing        | findByFirstnameContaining                               | … where x.firstname like ?1 (parameter bound wrapped in %)   |
| OrderBy           | findByAgeOrderByLastnameDesc                            | … where x.age = ?1 order by x.lastname desc                  |
| Not               | findByLastnameNot                                       | … where x.lastname <> ?1                                     |
| In                | findByAgeIn(Collection<Age> ages)                       | … where x.age in ?1                                          |
| NotIn             | findByAgeNotIn(Collection<Age> ages)                    | … where x.age not in ?1                                      |
| True              | findByActiveTrue()                                      | … where x.active = true                                      |
| False             | findByActiveFalse()                                     | … where x.active = false                                     |
| IgnoreCase        | findByFirstnameIgnoreCase                               | … where UPPER(x.firstame) = UPPER(?1)                        |

#### name query

```java
@Entity
@NamedQuery(name = "User.findByEmailAddress",
  query = "select u from User u where u.emailAddress = ?1")
public class User {

}
public interface UserRepository extends JpaRepository<User, Long> {

  List<User> findByLastname(String lastname);

  User findByEmailAddress(String emailAddress);
}
```

name query需要声明在实体类上,不能是其他地方.如果sql少的话,这样很方便,sql多的话就不便维护了.

#### query

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.emailAddress = ?1")
  User findByEmailAddress(String emailAddress);
}
```

一般情况下,named query和query使用的都是JPQL,如果要使用SQL,请参考下面的示例:

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE EMAIL_ADDRESS = ?1", nativeQuery = true)
  User findByEmailAddress(String emailAddress);
}
```

#### 使用分页

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE LASTNAME = ?1",
    countQuery = "SELECT count(*) FROM USERS WHERE LASTNAME = ?1",
    nativeQuery = true)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

#### query使用排序

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.lastname like ?1%")
  List<User> findByAndSort(String lastname, Sort sort);

  @Query("select u.id, LENGTH(u.firstname) as fn_len from User u where u.lastname like ?1%")
  List<Object[]> findByAsArrayAndSort(String lastname, Sort sort);
}

repo.findByAndSort("lannister", new Sort("firstname"));
repo.findByAndSort("stark", new Sort("LENGTH(firstname)")); //抛出异常,默认情况下拒绝排序的时候使用函数
repo.findByAndSort("targaryen", JpaSort.unsafe("LENGTH(firstname)")); //如果要使用函数,需要使用JpaSort.unsafe
repo.findByAsArrayAndSort("bolton", new Sort("fn_len"));
```

#### 具名参数

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
  User findByLastnameOrFirstname(@Param("lastname") String lastname,
                                 @Param("firstname") String firstname);
}
```

####  Spel表达式

```java
@Entity
public class User {

  @Id
  @GeneratedValue
  Long id;

  String lastname;
}

public interface UserRepository extends JpaRepository<User,Long> {

  @Query("select u from #{#entityName} u where u.lastname = ?1")
  List<User> findByLastname(String lastname);
}
```

#### 更改或删除实体

更改实体

```java
@Modifying
@Query("update User u set u.firstname = ?1 where u.lastname = ?2")
int setFixedFirstnameFor(String firstname, String lastname);
```

删除实体

```java
interface UserRepository extends Repository<User, Long> {

  void deleteByRoleId(long roleId);

  @Modifying
  @Query("delete from User u where user.role.id = ?1")
  void deleteInBulkByRoleId(long roleId);
}
```

#### Query Hints

```java
public interface UserRepository extends Repository<User, Long> {

  @QueryHints(value = { @QueryHint(name = "name", value = "value")},
              forCounting = false)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

#### Fetch and LoadGraphs

在面对一对多的映射关系的时候,JPA默认采用的是懒加载.此时如果我们要取出集合中的内容,可能会发出多条Sql语句,这样就会出现sql过多的情况,JPA提供了@NamedEntityGraph注解来解决这个问题,

```java
@Entity
@NamedEntityGraph(name = "GroupInfo.detail",
  attributeNodes = @NamedAttributeNode("members")) 
public class GroupInfo {

  // default fetch mode is lazy.
  @ManyToMany
  List<GroupMember> members = new ArrayList<GroupMember>();

  …
}
```

```java
@Repository
public interface GroupRepository extends CrudRepository<GroupInfo, String> {

  @EntityGraph(value = "GroupInfo.detail", type = EntityGraphType.LOAD) 
  GroupInfo getByGroupName(String name);

}
```

> 引用LoadGraphs,EntityGraphType.LOAD的作用是设定该字段是eager加载,其他字段跟随默认.EntityGraphType.FETCH也是eager加载,但其他字段懒加载 

上面的配置方式比较繁琐,可以通过下面的简化:

```java
@Repository
public interface GroupRepository extends CrudRepository<GroupInfo, String> {

  @EntityGraph(attributePaths = { "members" })
  GroupInfo getByGroupName(String name);

}
```

#### 投影

Spring Data查询方法通常返回由存储库管理的聚合根的一个或多个实例。但是，有时可能需要根据这些类型的某些属性创建投影。SpringData允许建模专用返回类型，以更有选择地检索托管聚合的部分视图。

假设我们的聚合根是下面的列子:

```java
class Person {

  @Id UUID id;
  String firstname, lastname;
  Address address;

  static class Address {
    String zipCode, city, street;
  }
}

interface PersonRepository extends Repository<Person, UUID> {

  Collection<Person> findByLastname(String lastname);
}
```

现在假设我们只想检索Person的姓名属性。 Spring Data提供了什么方法来实现这一目标？ 本章的其余部分回答了这个问题。

##### 基于接口的投影

将查询结果限制为仅名称属性的最简单方法是声明一个接口，该接口公开要读取的属性的访问器方法，如下:

```
interface NamesOnly {

  String getFirstname();
  String getLastname();
}
```

这里重要的一点是，此处定义的属性与聚合根中的属性完全匹配。 这样做可以添加查询方法，如下所示：

```java
interface PersonRepository extends Repository<Person, UUID> {

  Collection<NamesOnly> findByLastname(String lastname);
}
```

查询引擎在运行时为返回的每个元素创建该接口的代理实例，并将暴露方法的调用转发给目标对象。

可以递归使用。 如果您还想包含一些地址信息，请为其创建一个投影接口，并从getAddress（）声明中返回该接口，如以下示例所示：

```java
interface PersonSummary {

  String getFirstname();
  String getLastname();
  AddressSummary getAddress();

  interface AddressSummary {
    String getCity();
  }
}
```

###### 闭合投影

其访问器方法都与目标聚合的属性匹配的投影接口被认为是闭合投影。下面是一个闭合投影的例子

```java
interface NamesOnly {

  String getFirstname();
  String getLastname();
}
```

如果使用闭合投影，Spring Data可以优化查询执行，因为我们知道投影代理所需的所有属性。

###### 开放投影

投影接口中的访问器方法也可以使用@Value注释来计算新值 ,例如:

```java
interface NamesOnly {

  @Value("#{target.firstname + ' ' + target.lastname}") 
  String getFullName();
  …
}
```

在这种情况下，Spring Data无法应用查询执行优化，因为SpEL表达式可以使用聚合根的任何属性。

@Value中使用的表达式不应该太复杂 - 应该避免使用el表达式。 对于非常简单的表达式，一个选项可能是采用默认方法（在Java 8中引入），如以下示例所示：

```java
interface NamesOnly {

  String getFirstname();
  String getLastname();

  default String getFullName() {
    return getFirstname().concat(" ").concat(getLastname());
  }
}
```

这种方法要求您能够纯粹基于投影接口上公开的其他访问器方法实现逻辑。第二个更灵活的选项是在Spring bean中实现自定义逻辑，然后从SpEL表达式调用它，如以下示例所示：

```java
@Component
class MyBean {

  String getFullName(Person person) {
    …
  }
}

interface NamesOnly {

  @Value("#{@myBean.getFullName(target)}")
  String getFullName();
  …
}
```

注意SpEL表达式如何引用myBean并调用getFullName（…）方法并将投影目标转发为方法参数。

由SpEL表达式支持的方法也可以使用方法参数，然后可以从表达式引用它们。 方法参数可通过名为args的Object数组获得。 以下示例显示如何从args数组获取方法参数：

```java
interface NamesOnly {

  @Value("#{args[0] + ' ' + target.firstname + '!'}")
  String getSalutation(String prefix);
}
```

再次强调，对于更复杂的表达式，您应该使用Spring bean并让表达式调用方法

##### 基于类的投影(DTO)

定义投影的另一种方法是使用值类型DTO（数据传输对象），它包含应该检索的字段的属性。这些DTO类型使用方式和接口几乎相同，除了不发生代理并且不能应用嵌套投影。

如果要通过限定字段来优化查询效率,被查询的字段通过构造函数的参数被暴露,例如:

```java
class NamesOnly {

  private final String firstname, lastname;

  NamesOnly(String firstname, String lastname) {

    this.firstname = firstname;
    this.lastname = lastname;
  }

  String getFirstname() {
    return this.firstname;
  }

  String getLastname() {
    return this.lastname;
  }

  // equals(…) and hashCode() implementations
}
```

##### 动态投影

到目前为止，我们已经使用投影类型作为集合的返回类型或元素类型。 但是，您可能希望选择要在调用时使用的类型（动态类型）。 要应用动态投影，请使用查询方法，如以下示例中所示：

```java
interface PersonRepository extends Repository<Person, UUID> {

  <T> Collection<T> findByLastname(String lastname, Class<T> type);
}
```

如以下示例所示：

```java
void someMethod(PersonRepository people) {

  Collection<Person> aggregates =
    people.findByLastname("Matthews", Person.class);

  Collection<NamesOnly> aggregates =
    people.findByLastname("Matthews", NamesOnly.class);
}
```

## 存储过程

JPA 2.1规范增加了使用JPA条件查询API调用存储过程的功能。 我们引入了@Procedure注释，用于在repository方法上声明存储过程元数据。

下面是我们声明的存储过程:

```sql
/;
DROP procedure IF EXISTS plus1inout
/;
CREATE procedure plus1inout (IN arg int, OUT res int)
BEGIN ATOMIC
 set res = arg + 1;
END
/;
```

在实体类上声明存储过程

```java
@Entity
@NamedStoredProcedureQuery(name = "User.plus1", procedureName = "plus1inout", parameters = {
  @StoredProcedureParameter(mode = ParameterMode.IN, name = "arg", type = Integer.class),
  @StoredProcedureParameter(mode = ParameterMode.OUT, name = "res", type = Integer.class) })
public class User {}
```

在存储库方法上调用存储过程,有多种方式,例如:

value形式

```java
@Procedure("plus1inout")
Integer explicitlyNamedPlus1inout(Integer arg);
```

procedureName形式

```java
@Procedure(procedureName = "plus1inout")
Integer plus1inout(Integer arg);
```

name形式

```java
@Procedure(name = "User.plus1IO") //需要测试一下,IO代表什么,或许是文档错误
Integer entityAnnotatedCustomNamedProcedurePlus1IO(@Param("arg") Integer arg);
```

隐式形式

```java
@Procedure
Integer plus1(@Param("arg") Integer arg);
```

## Specifications

JPA2引入了一个标准API，您可以使用它以编程方式构建查询。通过编写criteria，可以为域类定义查询的where子句。再退一步，可以将这些标准视为JPA标准API约束描述的实体的谓词(predicate)。

Spring Data JPA采用Eric Evans的书“Domain Driven Design”中的Specifications概念。要支持此功能，可以让你的存储库接口继承JpaSpecificationExecutor接口，如下所示:

```java
public interface CustomerRepository extends CrudRepository<Customer, Long>, JpaSpecificationExecutor {

}
```

JpaSpecificationExecutor接口

```java
public interface JpaSpecificationExecutor<T> {

  Optional<T> findOne(@Nullable Specification<T> spec);

  List<T> findAll(@Nullable Specification<T> spec);

  Page<T> findAll(@Nullable Specification<T> spec, Pageable pageable);

  List<T> findAll(@Nullable Specification<T> spec, Sort sort);

  long count(@Nullable Specification<T> spec);
}
```

Specification接口

```java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder);
}
```

可以轻松地使用Specification在实体之上构建可扩展的predicates ，然后可以将其与JpaRepository结合使用，而无需为每个所需组合声明查询（方法），如以下示例所示：

```java
public class CustomerSpecs {

  public static Specification<Customer> isLongTermCustomer() {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<Customer> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         LocalDate date = new LocalDate().minusYears(2);
         return builder.lessThan(root.get(_Customer.createdAt), date);
      }
    };
  }

  public static Specification<Customer> hasSalesOfMoreThan(MontaryAmount value) {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         // build query here
      }
    };
  }
}
```

```java
List<Customer> customers = customerRepository.findAll(isLongTermCustomer());
```

> _Customer类型是使用JPA Metamodel生成器生成的元模型类型（有关示例，[Metamodel](JPA-Metamodel-Generator.adoc)）。因此，表达式_Customer.createdAt假定Customer具有Date类型的createdAt属性。

Specification 可以组合使用,例如:

```java
MonetaryAmount amount = new MonetaryAmount(200.0, Currencies.DOLLAR);
List<Customer> customers = customerRepository.findAll(
  isLongTermCustomer().or(hasSalesOfMoreThan(amount)));
```

## 使用Example查询

Example查询（QBE）是一种用户友好的查询技术，具有简单的接口。 它允许动态创建查询，并且不需要您编写包含字段名称的查询。 实际上，Query by Example不要求您使用特定于存储的查询语言来编写查询。

Example API包括三部分:

1. Probe(探测):域对象的实际实例
2. ExampleMatcher:ExampleMatcher包含有关如何匹配特定字段的详细信息。 它可以在多个示例中重用。
3. Example:Example包含Probe和ExampleMatcher。 它用于创建查询

Example 适用于一下场景: . 使用一组静态或动态约束查询数据存储 . 频繁重构域对象，而不必担心破坏现有查询。 . 独立于底层数据存储API工作。

Example有如下限制:

1. 不支持嵌套或分组的属性约束,例如: `firstname = ?0 or (firstname = ?1 and lastname = ?2)`
2. 仅支持字符串的开始/包含/结束/正则表达式匹配,以及其他属性类型的精确匹配

假如有如下实体:

```java
public class Person {

  @Id
  private String id;
  private String firstname;
  private String lastname;
  private Address address;

  // … getters and setters omitted
}
```

默认情况下，将忽略具有空值的字段，并使用特定于存储的默认值匹配字符串。 可以使用工厂方法或使用ExampleMatcher构建Example.Example是不可变的。 以下清单显示了一个简单的示例：

```java
Person person = new Person();
person.setFirstname("Dave");

Example<Person> example = Example.of(person);
```

使用Example,需要你自己的存储库继承QueryByExampleExecutor 接口

```java
public interface QueryByExampleExecutor<T> {

  <S extends T> S findOne(Example<S> example);

  <S extends T> Iterable<S> findAll(Example<S> example);

  // … more functionality omitted.
}
```

Example不限于默认设置。 您可以使用ExampleMatcher为字符串匹配，空值处理和属性特定设置指定自己的默认值，如以下示例所示：

```java
Person person = new Person();
person.setFirstname("Dave");

ExampleMatcher matcher = ExampleMatcher.matching()
  .withIgnorePaths("lastname")
  .withIncludeNullValues()
  .withStringMatcherEnding();

Example<Person> example = Example.of(person, matcher);
```

默认情况下，ExampleMatcher期望上设置的Probe所有值都匹配。 如果要获得与隐式定义的任意一个predicate匹配的结果，请使用ExampleMatcher.matchingAny（）。

您可以为单个属性指定行为（例如“firstname”和“lastname”，或者对于嵌套属性，“address.city”）。 您可以使用匹配选项和区分大小写来调整它，如以下示例所示：

```java
ExampleMatcher matcher = ExampleMatcher.matching()
  .withMatcher("firstname", endsWith())
  .withMatcher("lastname", startsWith().ignoreCase());
}
```

也可以使用lamada表达式

```java
ExampleMatcher matcher = ExampleMatcher.matching()
  .withMatcher("firstname", match -> match.endsWith())
  .withMatcher("firstname", match -> match.startsWith());
}
```

使用Example

```java
public interface PersonRepository extends JpaRepository<Person, String> { … }

public class PersonService {

  @Autowired PersonRepository personRepository;

  public List<Person> findPeople(Person probe) {
    return personRepository.findAll(Example.of(probe));
  }
}
```

##  事务

默认情况下，存储库实例上的CRUD方法是事务性的。对于读取操作，事务配置readOnly标志设置为true。所有其他配置都使用普通的@Transactional，以便应用默认事务配置。 有关详细信息，请参阅 [SimpleJpaRepository的JavaDoc](https://docs.spring.io/spring-data/data-jpa/docs/current/api/index.html?org/springframework/data/jpa/repository/support/SimpleJpaRepository.html).如果需要调整存储库中声明的某个方法的事务配置，请重新声明存储库接口中的方法，如下所示:

```java
public interface UserRepository extends CrudRepository<User, Long> {

  @Override
  @Transactional(timeout = 10)
  public List<User> findAll();

  // Further query method declarations
}
```

更改事务行为的另一种方法是使用（通常）覆盖多个存储库的服务实现。 其目的是为非CRUD操作定义事务边界

```java
@Service
class UserManagementImpl implements UserManagement {

  private final UserRepository userRepository;
  private final RoleRepository roleRepository;

  @Autowired
  public UserManagementImpl(UserRepository userRepository,
    RoleRepository roleRepository) {
    this.userRepository = userRepository;
    this.roleRepository = roleRepository;
  }

  @Transactional
  public void addRoleToAllUsers(String roleName) {

    Role role = roleRepository.findByName(roleName);

    for (User user : userRepository.findAll()) {
      user.addRole(role);
      userRepository.save(user);
    }
}
```

在上面的示例中,addRoleToAllUsers（）在事务内部运行（使用已有事务或创建新事务（如果没有已运行））。存储库中的事务配置会被忽略，因为外部事务配置确定所使用的实际配置.请注意，您必须激活<tx：annotation-driven />或显式使用@EnableTransactionManagement以使基于注释的配置起作用。

##  锁

关于锁的介绍,请参考 [JPA锁](JPA锁.adoc)

```java
interface UserRepository extends Repository<User, Long> {

  // Plain query method
  @Lock(LockModeType.READ)
  List<User> findByLastname(String lastname);
}
```

## Auditing

Spring Data支持透明地跟踪创建或更改实体的人员以及更改发生的时间.要从该功能中受益，您必须为实体类配备审计元数据，该元数据可以使用注释或通过实现接口来定义。

我们提供@CreatedBy和@LastModifiedBy来捕获创建或修改实体的用户以及@CreatedDate和@LastModifiedDate以捕获更改发生的时间。例如:

```java
class Customer {

  @CreatedBy
  private User user;

  @CreatedDate
  private DateTime createdDate;

}
```

如果您使用@CreatedBy或@LastModifiedBy，架构需要以某种方式了解当前主体。 为此，我们提供了一个AuditorAware<T>SPI接口，您必须实现该接口，以告知基础架构当前用户。 泛型类型T定义了使用@CreatedBy或@LastModifiedBy注释的属性的类型。

以下示例显示了使用Spring Security的Authentication对象的接口的实现：

```java
class SpringSecurityAuditorAware implements AuditorAware<User> {

  public Optional<User> getCurrentAuditor() {

    return Optional.ofNullable(SecurityContextHolder.getContext())
        .map(SecurityContext::getAuthentication)
        .filter(Authentication::isAuthenticated)
        .map(Authentication::getPrincipal)
        .map(User.class::cast);
  }
}
```

Spring Data JPA附带了一个实体监听器，可用于触发审计信息的捕获。 首先，您必须注册AuditingEntityListener以用于orm.xml文件中持久性上下文中的所有实体，如以下示例所示：

```xml
<persistence-unit-metadata>
  <persistence-unit-defaults>
    <entity-listeners>
      <entity-listener class="….data.jpa.domain.support.AuditingEntityListener" />
    </entity-listeners>
  </persistence-unit-defaults>
</persistence-unit-metadata>
```

您还可以使用@EntityListeners批注在每个实体上启用AuditingEntityListener，如下所示：

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class MyEntity {

}
```

从Spring Data JPA 1.5开始，您可以通过使用@EnableJpaAuditing批注对配置类进行批注来启用审计。 您仍然必须修改orm.xml文件并在类路径上使用spring-aspects.jar。 以下示例显示如何使用@EnableJpaAuditing批注：

```java
@Configuration
@EnableJpaAuditing
class Config {

  @Bean
  public AuditorAware<AuditableUser> auditorProvider() {
    return new AuditorAwareImpl();
  }
}
```

## 其他注意事项

### 在自定义实现中使用`JpaContext`

当使用多个 EntityManager 实例和自定义存储库实现时，您需要将正确的 EntityManager 连接到存储库实现类中。 您可以通过在 @PersistenceContext 注释中显式命名 EntityManager 来实现，或者EntityManager 使用 @Qualifier 代替@Autowired。

从 Spring Data JPA 1.9 开始，Spring Data JPA 包含一个名为 JpaContext 的类，它允许您通过托管域类获取 EntityManager，假设应用程序中仅由一个 EntityManager 实例管理。 以下示例显示了如何在自定义存储库中使用 JpaContext：

```java
class UserRepositoryImpl implements UserRepositoryCustom {
  private final EntityManager em;
  @Autowired
  public UserRepositoryImpl(JpaContext context) {
    this.em = context.getEntityManagerByManagedType(User.class);
  }
  …
}
```

这种方法的优点是，如果域类型被分配给不同的持久性单元，则不必修改存储库来更改对持久性单元的引用。

### 合并 persistence units

Spring 支持拥有多个持久化单元。 然而，有时您可能希望对应用程序进行模块化，但仍要确保所有这些模块都在单个持久性单元中运行。 为了实现这种行为，Spring Data JPA 提供了 PersistenceUnitManager 实现，它根据名称自动合并持久性单元.



# 附录

* 测试一对多查询
* 测试fetch查询的特点
* 测试级联保存
* getOne的场景测试