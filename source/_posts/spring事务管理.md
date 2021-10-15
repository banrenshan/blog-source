---
title: spring事务管理
tags: 
 - spring
 - 事务
categories:
 - 事务
---



# 概述

* 跨不同事务API（如Java事务API（JTA）、JDBC、Hibernate和Java持久性API（JPA））的一致编程模型。
* 支持声明式事务管理
* 与复杂事务API（如JTA）相比，spring事务管理的API更简单。
* 与Spring的数据访问抽象完美集成。

常见的两种事务管理模型：全局事务和本地事务。

## 全局事务

全局事务允许您使用多个事务资源，通常是关系数据库和消息队列。应用服务器通过JTA管理全局事务，JTA是一个麻烦的API（部分原因是它的异常模型）。此外，JTA用户事务通常需要JNDI的支持。全局事务的使用限制了应用程序代码的重用性，因为JTA通常仅在应用程序服务器环境中可用。

## 本地事务

本地事务是特定于资源的，例如与JDBC连接关联的事务。本地事务可能更易于使用，但有一个明显的缺点：它们不能跨多个事务资源工作。例如，使用JDBC连接管理事务的代码不能在全局JTA事务中运行。因为应用服务器不参与事务管理，所以它无法确保跨多个资源的正确性。（值得注意的是，大多数应用程序使用单个事务资源。）另一个缺点是本地事务会侵入编程模型。

## spring的一致编程模型

Spring解决了全局和本地事务的缺点。它允许应用程序开发人员在任何环境中使用一致的编程模型。您只需编写一次代码，就可以从不同环境中的不同事务管理策略中获益。Spring框架提供声明式和编程式事务管理。大多数用户更喜欢声明式事务管理，这是我们在大多数情况下推荐的。

Spring框架的事务管理支持改变了企业Java应用程序何时需要应用服务器的传统规则。特别是，您不需要一个应用服务器来通过EJB声明事务。事实上，即使您的应用服务器具有强大的JTA功能，您也可能认为Spring框架的声明性事务比EJBCMT提供了更强大的功能和更高效的编程模型。

通常，只有当应用程序需要处理跨多个事务资源时，才需要应用程序服务器的JTA功能。大多数应用程序使用单一的、高度可扩展的数据库（如Oracle RAC）。独立事务管理器（如Atomikos事务和JOTM）是另外的解决方案。当然，您可能需要其他应用服务器功能，如Java消息服务（JMS）和Java EE连接器体系结构（JCA）。

# 事务抽象模型

事务策略的概念是Spring事务抽象的关键。事务策略由TransactionManager定义，特别是用于强制事务管理的`org.springframework.transaction.PlatformTransactionManager`接口和用于被动（reactive ）事务管理的`org.springframework.transaction.ReactiveTransactionManager`接口。以下列表显示了`PlatformTransactionManager` API的定义：

```java
public interface PlatformTransactionManager extends TransactionManager {

    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;

    void rollback(TransactionStatus status) throws TransactionException;
}
```

这主要是一个服务提供者接口（ServiceProviderInterface，SPI），尽管您可以从应用程序代码中以编程方式使用它。由于PlatformTransactionManager是一个接口，因此可以根据需要轻松地对其进行模拟或存根。它与查找策略（如JNDI）无关。PlatformTransactionManager实现的定义与SpringFramework IoC容器中的任何其他对象（或bean）一样。单凭这一优势就可以使Spring框架事务成为一个有价值的抽象，即使在使用JTA时也是如此。您可以比直接使用JTA更容易地测试事务性代码。

同样，与Spring的理念一致，任何PlatformTransactionManager接口的方法都可以抛出TransactionException（即，它扩展了java.lang.RuntimeException类）。事务基础设施故障几乎总是致命的。虽然我们很少在事务失败的情况下继续执行，但是应用程序开发人员仍然可以选择捕获并处理TransactionException。

getTransaction（..）方法根据TransactionDefinition参数返回TransactionStatus对象。如果当前调用堆栈中存在匹配的事务，则返回的TransactionStatus可能表示新事务，也可能表示现有事务。后一种情况的含义是，与JavaEE事务上下文一样，TransactionStatus与执行线程相关联。

从SpringFramework5.2开始，Spring还为使用被动类型或Kotlin协同路由的被动应用程序提供事务管理抽象。下面的列表显示了org.springframework.transaction.ReactiveTransactionManager定义的事务策略：

```java
public interface ReactiveTransactionManager extends TransactionManager {

    Mono<ReactiveTransaction> getReactiveTransaction(TransactionDefinition definition) throws TransactionException;

    Mono<Void> commit(ReactiveTransaction status) throws TransactionException;

    Mono<Void> rollback(ReactiveTransaction status) throws TransactionException;
}
```

反应式事务管理器主要是一个服务提供者接口（SPI），尽管您可以从应用程序代码中以编程方式使用它。因为ReactiveTransactionManager是一个接口，所以可以根据需要轻松地对其进行模拟或存根。

TransactionDefinition :

* Propagation(传播方式):通常，事务范围内的所有代码都在该事务中运行。但是，如果在事务上下文已经存在的情况下运行事务方法，则可以指定行为。例如，代码可以在现有事务中继续运行（常见情况），或者可以暂停现有事务并创建新事务。Spring提供了EJBCMT中熟悉的所有事务传播选项。要了解Spring中事务传播的语义，请参阅事务传播。
* Isolation(隔离级别):此事务与其他事务的工作隔离的程度。例如，此事务是否可以看到来自其他事务的未提交写入？
* Timeout(超时时间):此事务在超时和被底层事务基础结构自动回滚之前运行的时间。
* Read-only status(只读状态):当代码读取但不修改数据时，可以使用只读事务。在某些情况下，只读事务可能是一种有用的优化，例如在使用Hibernate时。

TransactionStatus接口为事务代码提供了控制事务执行和查询事务状态的简单方法。这些概念应该很熟悉，因为它们对于所有事务API都是通用的。下表显示了TransactionStatus接口：

```java
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {

    @Override
    boolean isNewTransaction();

    boolean hasSavepoint();

    @Override
    void setRollbackOnly();

    @Override
    boolean isRollbackOnly();

    void flush();

    @Override
    boolean isCompleted();
}
```

无论您在Spring中选择声明式还是编程式事务管理，定义正确的TransactionManager实现都是绝对必要的。您通常通过依赖项注入来定义此实现。

TransactionManager实现通常需要了解其工作环境：JDBC、JTA、Hibernate等。以下示例显示如何定义本地PlatformTransactionManager实现（在本例中，使用普通JDBC）

您可以通过创建类似以下内容的bean来定义JDBC数据源：

```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>
```

然后，相关的PlatformTransactionManager bean定义引用了数据源定义。它应该类似于以下示例：

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

如果您在JavaEE容器中使用JTA，那么您将使用通过JNDI获得的容器数据源以及Spring的JtaTransactionManager。以下示例显示了JTA和JNDI查找：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/jee
        https://www.springframework.org/schema/jee/spring-jee.xsd">

    <jee:jndi-lookup id="dataSource" jndi-name="jdbc/jpetstore"/>

    <bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager" />

    <!-- other <bean/> definitions here -->

</beans>
```

JtaTransactionManager不需要知道数据源（或任何其他特定资源），因为它使用容器的全局事务管理。

> 前面的DataSourceBean定义使用来自jee名称空间的<jndi lookup/>标记。有关更多信息，请参阅[JEE模式](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#xsd-schemas-jee)。

在所有Spring事务设置中，应用程序代码不需要更改。您可以仅通过更改配置来更改事务的管理方式，即使这种更改意味着从本地事务转移到全局事务，反之亦然。

## Hibernate 事务设置

您还可以轻松地使用Hibernate本地事务，如以下示例所示。在本例中，您需要定义Hibernate LocalSessionFactoryBean，应用程序代码可以使用它来获取Hibernate会话实例。
DataSourceBean定义类似于前面显示的本地JDBC示例，因此在下面的示例中没有显示。

本例中的txManager bean是HibernateTransactionManager类型。正如DataSourceTransactionManager需要对DataSource的引用一样，HibernateTransactionManager需要对SessionFactory的引用。以下示例声明sessionFactory和txManager bean：

```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mappingResources">
        <list>
            <value>org/springframework/samples/petclinic/hibernate/petclinic.hbm.xml</value>
        </list>
    </property>
    <property name="hibernateProperties">
        <value>
            hibernate.dialect=${hibernate.dialect}
        </value>
    </property>
</bean>

<bean id="txManager" class="org.springframework.orm.hibernate5.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>
```

如果使用JavaEE容器管理的JTA事务设置Hibernate，你应该使用JtaTransactionManager，Hibernate也需要通过coordinator 知道JTA：

```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mappingResources">
        <list>
            <value>org/springframework/samples/petclinic/hibernate/petclinic.hbm.xml</value>
        </list>
    </property>
    <property name="hibernateProperties">
        <value>
            hibernate.dialect=${hibernate.dialect}
            hibernate.transaction.coordinator_class=jta
            hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_STATEMENT
        </value>
    </property>
</bean>

<bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager"/>
```

或者将JtaTransactionManager传递给LocalSessionFactoryBean：

```java
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mappingResources">
        <list>
            <value>org/springframework/samples/petclinic/hibernate/petclinic.hbm.xml</value>
        </list>
    </property>
    <property name="hibernateProperties">
        <value>
            hibernate.dialect=${hibernate.dialect}
        </value>
    </property>
    <property name="jtaTransactionManager" ref="txManager"/>
</bean>

<bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager"/>
```

# 锁定事务资源

现在应该清楚如何创建不同的事务管理器以及它们如何链接到需要与事务同步的相关资源（例如DataSourceTransactionManager到JDBC数据源、HibernateTransactionManager到Hibernate SessionFactory等等）。本节描述应用程序代码（通过使用持久性API（如JDBC、Hibernate或JPA）直接或间接地）如何确保这些资源被正确创建、重用和清理。本节还讨论了如何（可选）通过相关TransactionManager触发事务同步。

## 高级同步方法

首选的方法是使用Spring最高级别的基于模板的持久性集成API，或者使用本机ORMAPI和事务感知工厂bean或代理来管理本机资源工厂。这些事务感知解决方案在内部处理资源创建和重用、清理、资源的可选事务同步以及异常映射。因此，用户数据访问代码不必处理这些任务，而只需关注非样板持久性逻辑。通常，您使用本机ORMAPI，或者通过使用JdbcTemplate采用模板方法进行JDBC访问。这些解决方案将在本参考文档的后续章节中详细介绍。

## 低层同步方法

DataSourceUtils（用于JDBC）、EntityManagerFactoryUtils（用于JPA）、SessionFactoryUtils（用于Hibernate）等类存在于较低级别。当您希望应用程序代码直接处理本机持久性API的资源类型时，可以使用这些类来确保获得正确的Spring Framework托管实例，同步事务（可选），并将过程中发生的异常正确映射到一致的API。

例如，对于JDBC，您可以使用Spring的org.springframework.JDBC.DataSource.DataSourceUtils类，而不是传统的JDBC方法调用DataSource（）方法，如下所示：

```java
Connection conn = DataSourceUtils.getConnection(dataSource);
```

如果现有事务已经有一个与之同步（链接）的连接，则返回该实例。否则，方法调用将触发新连接的创建，该连接（可选）与任何现有事务同步，并可供后续在该事务中重用。如前所述，任何SQLException都包装在Spring框架CannotGetJdbcConnectionException中，这是Spring框架的未检查DataAccessException类型层次结构之一。这种方法为您提供了比从SQLException轻松获得的更多的信息，并确保了跨数据库甚至跨不同持久性技术的可移植性。

这种方法也可以在没有Spring事务管理的情况下工作（事务同步是可选的），因此无论是否使用Spring进行事务管理，都可以使用它。
当然，一旦您使用了Spring的JDBC支持、JPA支持或Hibernate支持，您通常不喜欢使用DataSourceUtils或其他帮助器类，因为通过Spring抽象工作要比直接使用相关API快乐得多。例如，如果您使用Spring JdbcTemplate或jdbc.object包来简化jdbc的使用，那么正确的连接检索会在后台进行，您无需编写任何特殊代码。

## `TransactionAwareDataSourceProxy`

在最底层存在TransactionWaredataSourceProxy类。这是目标数据源的代理，它包装目标数据源以增加对Spring管理的事务的感知。在这方面，它类似于事务性JNDI数据源，由JavaEE服务器提供。

除了必须调用和传递标准JDBC数据源接口实现的现有代码外，您几乎不需要或不想使用此类。在这种情况下，该代码可能可用，但正在参与Spring管理的事务。您可以使用前面提到的高级抽象来编写新代码。

# 声明式事务管理

Spring Framework 的声明式事务管理底层使用AOP实现。

Spring Framework 的声明式事务管理类似于 EJB CMT，因为您可以将事务行为（或缺少它）指定到单个方法级别。 如有必要，您可以在事务上下文中进行 setRollbackOnly() 调用。 两种类型的事务管理之间的区别是：

* 与绑定到 JTA 的 EJB CMT 不同，Spring Framework 的声明式事务管理可在任何环境中工作。 它可以通过调整配置文件使用 JDBC、JPA 或 Hibernate 来处理 JTA 事务或本地事务。
* 您可以将 Spring Framework 声明式事务管理应用于任何类，而不仅仅是诸如 EJB 之类的特殊类。
* Spring Framework 提供了声明式回滚规则，该功能没有 EJB 等效项。 提供了对回滚规则的编程和声明性支持。
* Spring Framework 允许您使用 AOP 自定义事务行为。 例如，您可以在事务回滚的情况下插入自定义行为。 您还可以添加任意建议以及事务建议。 使用 EJB CMT，您不能影响容器的事务管理，除非使用 setRollbackOnly()。
* Spring Framework 不支持跨远程调用传播事务上下文，高端应用程序服务器支持。 如果您需要此功能，我们建议您使用 EJB。 但是，在使用此类功能之前请仔细考虑，因为通常情况下，人们不希望事务跨越远程调用。

回滚规则的概念很重要。 它们让您指定哪些异常（和可抛出的）应该导致自动回滚。 您可以在配置中（而不是在 Java 代码中）以声明方式指定此项。 因此，尽管您仍然可以在 TransactionStatus 对象上调用 setRollbackOnly() 来回滚当前事务，但大多数情况下您可以指定 MyApplicationException 必须始终导致回滚的规则。 此选项的显着优点是业务对象不依赖于事务基础结构。 例如，他们通常不需要导入 Spring 事务 API 或其他 Spring API。

尽管 EJB 容器默认行为会在系统异常（通常是运行时异常）时自动回滚事务，但 EJB CMT 不会在应用程序异常（即 java.rmi.RemoteException 以外的已检查异常）时自动回滚事务。 虽然声明性事务管理的 Spring 默认行为遵循 EJB 约定（仅在未检查的异常时自动回滚），但自定义此行为通常很有用。

## 理解 Spring 框架的声明式事务实现

仅仅告诉您使用 @Transactional 注释来注释您的类，将 @EnableTransactionManagement 添加到您的配置中，并期望您了解它是如何工作的，这是不够的。 为了提供更深入的理解，本节在事务相关问题的上下文中解释了 Spring 框架的声明式事务基础结构的内部工作原理。

关于 Spring Framework 的声明式事务支持，需要掌握的最重要的概念是这种支持是通过 AOP 代理启用的，并且事务 advice 是由元数据（目前基于 XML 或注解）驱动的。 AOP 与事务元数据的组合产生了一个 AOP 代理，它使用 TransactionInterceptor 结合适当的 TransactionManager 实现来驱动围绕方法调用的事务。

Spring Framework 的 TransactionInterceptor 为命令式和反应式编程模型提供事务管理。 拦截器通过检查方法返回类型来检测所需的事务管理风格。 返回响应式类型的方法，例如 Publisher 或 Kotlin Flow（或它们的子类型）有资格进行响应式事务管理。 所有其他返回类型包括 void 使用代码路径进行命令式事务管理。

事务管理风格影响需要哪个事务管理器。 命令式事务需要 PlatformTransactionManager，而反应式事务使用 ReactiveTransactionManager 实现。

@Transactional 通常与由 PlatformTransactionManager 管理的线程绑定事务一起使用，将事务暴露给当前执行线程内的所有数据访问操作。 注意：这不会传播到方法中新启动的线程。

由 ReactiveTransactionManager 管理的反应式事务使用 Reactor 上下文而不是线程本地属性。 因此，所有参与的数据访问操作都需要在同一个反应管道中的同一个 Reactor 上下文中执行。

![tx](spring%E4%BA%8B%E5%8A%A1%E7%AE%A1%E7%90%86/tx.png)

考虑以下接口及其伴随的实现。 此示例使用 Foo 和 Bar 类作为占位符，以便您可以专注于事务使用，而无需关注特定的域模型。 对于本示例而言，DefaultFooService 类在每个实现的方法的主体中抛出 UnsupportedOperationException 实例这一事实很好。 该行为让您可以看到正在创建的事务，然后回滚以响应 UnsupportedOperationException 实例。 以下清单显示了 FooService 接口：

```java
// the service interface that we want to make transactional

package x.y.service;

public interface FooService {

    Foo getFoo(String fooName);

    Foo getFoo(String fooName, String barName);

    void insertFoo(Foo foo);

    void updateFoo(Foo foo);

}
```

以下示例显示了上述接口的实现：

```java
package x.y.service;

public class DefaultFooService implements FooService {

    @Override
    public Foo getFoo(String fooName) {
        // ...
    }

    @Override
    public Foo getFoo(String fooName, String barName) {
        // ...
    }

    @Override
    public void insertFoo(Foo foo) {
        // ...
    }

    @Override
    public void updateFoo(Foo foo) {
        // ...
    }
}
```

假设 FooService 接口的前两个方法 getFoo(String) 和 getFoo(String, String) 必须在具有只读语义的事务上下文中运行，而其他方法 insertFoo(Foo) 和 updateFoo(Foo )，必须在具有读写语义的事务上下文中运行。 下面几段详细解释下面的配置：

```xml
<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- the transactional advice (what 'happens'; see the <aop:advisor/> bean below) -->
    <tx:advice id="txAdvice" transaction-manager="txManager">
        <!-- the transactional semantics... -->
        <tx:attributes>
            <!-- all methods starting with 'get' are read-only -->
            <tx:method name="get*" read-only="true"/>
            <!-- other methods use the default transaction settings (see below) -->
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- ensure that the above transactional advice runs for any execution
        of an operation defined by the FooService interface -->
    <aop:config>
        <aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
    </aop:config>

    <!-- don't forget the DataSource -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
        <property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
        <property name="username" value="scott"/>
        <property name="password" value="tiger"/>
    </bean>

    <!-- similarly, don't forget the TransactionManager -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->

</beans>
```

检查前面的配置。 它假定您想要创建一个服务对象，即 fooService bean，事务性的。 要应用的事务语义封装在 <tx:advice/> 定义中。 <tx:advice/> 定义读作“所有以 get 开头的方法将在只读事务的上下文中运行，所有其他方法将使用默认事务语义运行”。 <tx:advice/> 标记的 transaction-manager 属性设置为将驱动事务的 TransactionManager bean 的名称（在本例中为 txManager bean）。

<aop:pointcut/> 元素中定义的表达式是一个 AspectJ 切入点表达式。 有关 Spring 中切入点表达式的更多详细信息，请参阅 AOP 部分。

一个常见的要求是使整个服务层具有事务性。 最好的方法是更改切入点表达式以匹配服务层中的任何操作。 以下示例显示了如何执行此操作：

```xml
<aop:config>
    <aop:pointcut id="fooServiceMethods" expression="execution(* x.y.service.*.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceMethods"/>
</aop:config>
```

既然我们已经分析了配置，您可能会问自己，“所有这些配置实际上做了什么？”

前面显示的配置用于围绕从 fooService bean 定义创建的对象创建事务代理。 代理使用事务建议进行配置，以便在代理上调用适当的方法时，根据与该方法关联的事务配置，启动、暂停、标记为只读等事务。 考虑以下测试驱动前面显示的配置的程序：

```java
public final class Boot {

    public static void main(final String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("context.xml");
        FooService fooService = ctx.getBean(FooService.class);
        fooService.insertFoo(new Foo());
    }
}
```

运行上述程序的输出应类似于以下内容（为清楚起见，已截断了来自 DefaultFooService 类的 insertFoo(..) 方法抛出的 UnsupportedOperationException 的 Log4J 输出和堆栈跟踪）：

```sh
<!-- the Spring container is starting up... -->
[AspectJInvocationContextExposingAdvisorAutoProxyCreator] - Creating implicit proxy for bean 'fooService' with 0 common interceptors and 1 specific interceptors

<!-- the DefaultFooService is actually proxied -->
[JdkDynamicAopProxy] - Creating JDK dynamic proxy for [x.y.service.DefaultFooService]

<!-- ... the insertFoo(..) method is now being invoked on the proxy -->
[TransactionInterceptor] - Getting transaction for x.y.service.FooService.insertFoo

<!-- the transactional advice kicks in here... -->
[DataSourceTransactionManager] - Creating new transaction with name [x.y.service.FooService.insertFoo]
[DataSourceTransactionManager] - Acquired Connection [org.apache.commons.dbcp.PoolableConnection@a53de4] for JDBC transaction

<!-- the insertFoo(..) method from DefaultFooService throws an exception... -->
[RuleBasedTransactionAttribute] - Applying rules to determine whether transaction should rollback on java.lang.UnsupportedOperationException
[TransactionInterceptor] - Invoking rollback for transaction on x.y.service.FooService.insertFoo due to throwable [java.lang.UnsupportedOperationException]

<!-- and the transaction is rolled back (by default, RuntimeException instances cause rollback) -->
[DataSourceTransactionManager] - Rolling back JDBC transaction on Connection [org.apache.commons.dbcp.PoolableConnection@a53de4]
[DataSourceTransactionManager] - Releasing JDBC Connection after transaction
[DataSourceUtils] - Returning JDBC Connection to DataSource

Exception in thread "main" java.lang.UnsupportedOperationException at x.y.service.DefaultFooService.insertFoo(DefaultFooService.java:14)
<!-- AOP infrastructure stack trace elements removed for clarity -->
at $Proxy0.insertFoo(Unknown Source)
at Boot.main(Boot.java:11)
```

## 回滚声明式事务

上一节概述了如何在应用程序中以声明方式为类（通常是服务层类）指定事务设置的基础知识。 本节介绍如何以简单的声明方式控制事务的回滚。

向 Spring Framework 的事务基础结构指示要回滚事务的工作的推荐方法是从当前在事务上下文中执行的代码抛出异常。 Spring Framework 的事务基础结构代码会捕获任何未处理的异常，因为它会冒泡调用堆栈并确定是否将事务标记为回滚。

在其默认配置中，Spring Framework 的事务基础结构代码仅在运行时、未检查异常的情况下才将事务标记为回滚。 也就是说，当抛出的异常是 RuntimeException 的实例或子类时。 （默认情况下，Error 实例也会导致回滚）。 从事务方法抛出的已检查异常不会导致默认配置中的回滚。

您可以准确配置哪些异常类型将事务标记为回滚，包括已检查的异常。 以下 XML 片段演示了如何为已检查的特定于应用程序的异常类型配置回滚：

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
    <tx:method name="get*" read-only="true" rollback-for="NoProductInStockException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

如果您不希望在抛出异常时回滚事务，您还可以指定“无回滚规则”。 下面的示例告诉 Spring Framework 的事务基础结构即使面对未处理的 InstrumentNotFoundException 也提交伴随的事务：

```xml
<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="updateStock" no-rollback-for="InstrumentNotFoundException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

当 Spring Framework 的事务基础架构捕获异常并查询配置的回滚规则以确定是否将事务标记为回滚时，最强匹配规则获胜。 因此，在以下配置的情况下，除 InstrumentNotFoundException 之外的任何异常都会导致伴随事务回滚：

```xml
<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="*" rollback-for="Throwable" no-rollback-for="InstrumentNotFoundException"/>
    </tx:attributes>
</tx:advice>
```

您还可以以编程方式指示需要的回滚。 虽然简单，但这个过程非常具有侵入性，并且将您的代码与 Spring Framework 的事务基础结构紧密耦合。 以下示例显示了如何以编程方式指示所需的回滚：

```java
public void resolvePosition() {
    try {
        // some business logic...
    } catch (NoProductInStockException ex) {
        // trigger rollback programmatically
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

## 为不同的 Bean 配置不同的事务语义

考虑这样一个场景，您有多个服务层对象，并且您希望对每个对象应用完全不同的事务配置。 您可以通过定义具有不同切入点和建议引用属性值的不同 <aop:advisor/> 元素来实现。

作为比较，首先假设您的所有服务层类都定义在根 x.y.service 包中。 要使作为该包（或子包）中定义的类的实例且名称以 Service 结尾的所有 bean 具有默认事务配置，您可以编写以下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>

        <aop:pointcut id="serviceOperation"
                expression="execution(* x.y.service..*Service.*(..))"/>

        <aop:advisor pointcut-ref="serviceOperation" advice-ref="txAdvice"/>

    </aop:config>

    <!-- these two beans will be transactional... -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>
    <bean id="barService" class="x.y.service.extras.SimpleBarService"/>

    <!-- ... and these two beans won't -->
    <bean id="anotherService" class="org.xyz.SomeService"/> <!-- (not in the right package) -->
    <bean id="barManager" class="x.y.service.SimpleBarManager"/> <!-- (doesn't end in 'Service') -->

    <tx:advice id="txAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- other transaction infrastructure beans such as a TransactionManager omitted... -->

</beans>
```

以下示例显示如何使用完全不同的事务设置配置两个不同的 bean：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>

        <aop:pointcut id="defaultServiceOperation"
                expression="execution(* x.y.service.*Service.*(..))"/>

        <aop:pointcut id="noTxServiceOperation"
                expression="execution(* x.y.service.ddl.DefaultDdlManager.*(..))"/>

        <aop:advisor pointcut-ref="defaultServiceOperation" advice-ref="defaultTxAdvice"/>

        <aop:advisor pointcut-ref="noTxServiceOperation" advice-ref="noTxAdvice"/>

    </aop:config>

    <!-- this bean will be transactional (see the 'defaultServiceOperation' pointcut) -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- this bean will also be transactional, but with totally different transactional settings -->
    <bean id="anotherFooService" class="x.y.service.ddl.DefaultDdlManager"/>

    <tx:advice id="defaultTxAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <tx:advice id="noTxAdvice">
        <tx:attributes>
            <tx:method name="*" propagation="NEVER"/>
        </tx:attributes>
    </tx:advice>

    <!-- other transaction infrastructure beans such as a TransactionManager omitted... -->

</beans>
```

#### `<tx:advice/>` 设置

本节总结了您可以使用 <tx:advice/> 标签指定的各种事务设置。 默认的 <tx:advice/> 设置是：

* 传播设置是必需的。

* 隔离级别为默认值。

* 事务是读写的。

* 事务超时默认为底层事务系统的默认超时，如果不支持超时，则无。

* 任何 RuntimeException 都会触发回滚，而任何已检查的 Exception 都不会。

您可以更改这些默认设置。 下表总结了嵌套在 <tx:advice/> 和 <tx:attributes/> 标签中的 <tx:method/> 标签的各种属性：

#### `@Transactional`

除了用于事务配置的基于 XML 的声明性方法之外，您还可以使用基于注释的方法。 直接在 Java 源代码中声明事务语义会使声明更接近受影响的代码。 不存在过度耦合的危险，因为旨在以事务方式使用的代码几乎总是以这种方式部署的。

```java
// the service class that we want to make transactional
@Transactional
public class DefaultFooService implements FooService {

    @Override
    public Foo getFoo(String fooName) {
        // ...
    }

    @Override
    public Foo getFoo(String fooName, String barName) {
        // ...
    }

    @Override
    public void insertFoo(Foo foo) {
        // ...
    }

    @Override
    public void updateFoo(Foo foo) {
        // ...
    }
}
```

如上在类级别使用，注释指示声明类（及其子类）的所有方法的默认值。 或者，每个方法都可以单独注释。 有关 Spring 将哪些方法视为事务性的更多详细信息，请参阅方法可见性和 @Transactional。 请注意，类级别的注释不适用于在类层次结构中向上的祖先类； 在这种情况下，需要在本地重新声明继承的方法才能参与子类级别的注释。

当上面的 POJO 类在 Spring 上下文中定义为 bean 时，您可以通过 @Configuration 类中的 @EnableTransactionManagement 注释使 bean 实例具有事务性。 有关完整详细信息，请参阅 javadoc。

在 XML 配置中， <tx:annotation-driven/> 标签提供了类似的便利：

```xml
<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- enable the configuration of transactional behavior based on annotations -->
    <!-- a TransactionManager is still required -->
    <tx:annotation-driven transaction-manager="txManager"/> 

    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- (this dependency is defined somewhere else) -->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->

</beans>
```

当您在 Spring 的标准配置中使用事务代理时，您应该只将 @Transactional 注释应用于具有公共可见性的方法。 如果您使用 @Transactional 注释对受保护的、私有的或包可见的方法进行注释，则不会引发错误，但带注释的方法不会显示配置的事务设置。 如果您需要注释非公共方法，请考虑以下段落中有关基于类的代理的提示，或考虑使用 AspectJ 编译时或加载时编织（稍后描述）。

在 @Configuration 类中使用 @EnableTransactionManagement 时，通过注册自定义 transactionAttributeSource bean，也可以将受保护或包可见的方法设为基于类的代理的事务性，如下例所示。 但是请注意，基于接口的代理中的事务方法必须始终是公共的并在代理接口中定义。

```java
/**
 * Register a custom AnnotationTransactionAttributeSource with the
 * publicMethodsOnly flag set to false to enable support for
 * protected and package-private @Transactional methods in
 * class-based proxies.
 *
 * @see ProxyTransactionManagementConfiguration#transactionAttributeSource()
 */
@Bean
TransactionAttributeSource transactionAttributeSource() {
    return new AnnotationTransactionAttributeSource(false);
}
```

您可以将 @Transactional 注释应用于接口定义、接口上的方法、类定义或类上的方法。 然而，仅仅存在 @Transactional 注释并不足以激活事务行为。 @Transactional 注解仅仅是元数据，可以被一些运行时基础设施使用，这些基础设施是 @Transactional-aware 并且可以使用元数据来配置具有事务行为的适当 bean。 在前面的示例中， <tx:annotation-driven/> 元素开启了事务行为。

Spring 团队建议您只使用 @Transactional 注释来注释具体类（和具体类的方法），而不是注释接口。 您当然可以将 @Transactional 注释放在接口（或接口方法）上，但是如果您使用基于接口的代理，这只能像您期望的那样工作。 Java 注释不是从接口继承的事实意味着，如果您使用基于类的代理 (proxy-target-class="true") 或基于编织的方面 (mode="aspectj")，则事务设置不是 被代理和编织基础设施识别，并且对象没有包含在事务代理中。

在代理模式下（默认），只有通过代理进入的外部方法调用才会被拦截。 这意味着自调用（实际上，目标对象中的一个方法调用目标对象的另一个方法）在运行时不会导致实际事务，即使被调用的方法用@Transactional 标记。 此外，代理必须完全初始化以提供预期的行为，因此您不应在初始化代码中依赖此功能 — 例如，在@PostConstruct 方法中。

如果您希望自调用也与事务一起包装，请考虑使用 AspectJ 模式（请参阅下表中的 mode 属性）。 在这种情况下，首先没有代理。 相反，目标类被编织（即，它的字节码被修改）以支持任何类型的方法上的@Transactional 运行时行为。

| xml属性                  | 注解属性                                                     | 默认                      | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ | ------------------------- | ------------------------------------------------------------ |
| transaction-manager      | [`TransactionManagementConfigurer`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/transaction/annotation/TransactionManagementConfigurer.html) | transactionManager        | 要使用的事务管理器的bean名称。                               |
| mode                     | mode                                                         | proxy                     | proxy：aop代理的方式。aspectj：织入字节码的方式              |
| proxy-target-class<br /> | proxyTargetClass                                             | false                     | 仅在proxy模式下配置改选项，true基于类的代理，false基于jdk接口的代理 |
| order                    | order                                                        | Ordered.LOWEST_PRECEDENCE | 定义advice advice在多个aop切面中的顺序                       |

##### `@Transactional`设置

| 属性                                                         | 类型                                      | 说明                                                         |
| ------------------------------------------------------------ | ----------------------------------------- | ------------------------------------------------------------ |
| [value](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-multiple-tx-mgrs-with-attransactional) | String                                    | 指定要使用的事务管理器的bean名称，可选。                     |
| [propagation](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-propagation) | enum`: `Propagation                       | 传播机制                                                     |
| isolation                                                    | enum`: `Isolation                         | 隔离级别，仅适用于 REQUIRED 或 REQUIRES_NEW 的传播值。       |
| timeout                                                      | int,以秒为单位                            | 仅适用于 REQUIRED 或 REQUIRES_NEW 的传播值。                 |
| readOnly                                                     | boolean                                   | 仅适用于 REQUIRED 或 REQUIRES_NEW 的传播值。                 |
| rollbackFor                                                  | Class 对象的数组，必须从 Throwable 派生。 | 那些异常回滚事务                                             |
| rollbackForClassName                                         | 类名数组。 这些类必须从 Throwable 派生。  | 那些异常回滚事务                                             |
| noRollbackFor                                                | Class 对象的数组，必须从 Throwable 派生。 | 那些异常不回滚事务                                           |
| noRollbackForClassName                                       | 类名数组。 这些类必须从 Throwable 派生。  | 那些异常不回滚事务                                           |
| label                                                        | string数组                                | 标签可以由事务管理器评估以将特定于实现的行为与实际事务相关联。 |
|                                                              |                                           |                                                              |

目前，您无法显式控制事务的名称，其中“名称”表示出现在事务监视器（如果适用）（例如，WebLogic 的事务监视器）和日志输出中的事务名称。 对于声明性事务，事务名称始终是完全限定的类名 + 建议类的方法名称。 例如，如果 BusinessService 类的 handlePayment(..) 方法启动了一个事务，那么该事务的名称将是：com.example.BusinessService.handlePayment。

使用@Transactional 的多个事务管理器

大多数 Spring 应用程序只需要一个事务管理器，但在某些情况下，您可能需要在一个应用程序中使用多个独立的事务管理器。 您可以使用 @Transactional 注释的 value 或 transactionManager 属性来选择性地指定要使用的 TransactionManager 的标识。 这可以是事务管理器 bean 的 bean 名称或限定符值。 例如，使用限定符表示法，您可以将以下 Java 代码与应用程序上下文中的以下事务管理器 bean 声明相结合：

```java
public class TransactionalService {

    @Transactional("order")
    public void setSomething(String name) { ... }

    @Transactional("account")
    public void doSomething() { ... }

    @Transactional("reactive-account")
    public Mono<Void> doSomethingReactive() { ... }
}
```

```java
<tx:annotation-driven/>

    <bean id="transactionManager1" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        ...
        <qualifier value="order"/>
    </bean>

    <bean id="transactionManager2" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        ...
        <qualifier value="account"/>
    </bean>

    <bean id="transactionManager3" class="org.springframework.data.r2dbc.connectionfactory.R2dbcTransactionManager">
        ...
        <qualifier value="reactive-account"/>
    </bean>
```

在这种情况下， TransactionalService 上的各个方法在单独的事务管理器下运行，由订单、帐户和反应帐户限定符区分。 如果没有找到特别限定的 TransactionManager bean，则仍然使用默认的 <tx:annotation-driven> 目标 bean 名称 transactionManager。

## 事务的传播

本节介绍 Spring 中事务传播的一些语义。 请注意，本节不是对事务传播的正确介绍。 相反，它详细介绍了有关 Spring 中事务传播的一些语义。

在 Spring 管理的事务中，注意物理事务和逻辑事务之间的区别，以及传播设置如何应用于这种区别。

![tx prop required](spring%E4%BA%8B%E5%8A%A1%E7%AE%A1%E7%90%86/tx_prop_required.png)

编程的方式使用事务

Spring 框架提供了两种以编程的方式管理事务，通过使用：

* TransactionTemplate 或 TransactionalOperator。

* 直接一个 TransactionManager 实现。

Spring 团队一般推荐 TransactionTemplate 用于命令式流中的程序化事务管理，而 TransactionalOperator 用于响应式代码。 第二种方法类似于使用 JTA UserTransaction API，尽管异常处理不那么麻烦。

#### `TransactionTemplate`

TransactionTemplate 采用与其他 Spring 模板相同的方法，例如 JdbcTemplate。 

```java
public class SimpleService implements Service {

    // single TransactionTemplate shared amongst all methods in this instance
    private final TransactionTemplate transactionTemplate;

    // use constructor-injection to supply the PlatformTransactionManager
    public SimpleService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }

    public Object someServiceMethod() {
        return transactionTemplate.execute(new TransactionCallback() {
            // the code in this method runs in a transactional context
            public Object doInTransaction(TransactionStatus status) {
                updateOperation1();
                return resultOfUpdateOperation2();
            }
        });
    }
}
```

如果没有返回值，可以使用方便的 TransactionCallbackWithoutResult 类和匿名类，如下：

```java
transactionTemplate.execute(new TransactionCallbackWithoutResult() {
    protected void doInTransactionWithoutResult(TransactionStatus status) {
        updateOperation1();
        updateOperation2();
    }
});
```

回调中的代码可以通过调用提供的 TransactionStatus 对象上的 setRollbackOnly() 方法来回滚事务，如下所示：

```java
transactionTemplate.execute(new TransactionCallbackWithoutResult() {

    protected void doInTransactionWithoutResult(TransactionStatus status) {
        try {
            updateOperation1();
            updateOperation2();
        } catch (SomeBusinessException ex) {
            status.setRollbackOnly();
        }
    }
});
```

配置事务：

```java
public class SimpleService implements Service {

    private final TransactionTemplate transactionTemplate;

    public SimpleService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);

        // the transaction settings can be set here explicitly if so desired
        this.transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_UNCOMMITTED);
        this.transactionTemplate.setTimeout(30); // 30 seconds
        // and so forth...
    }
}
```

最后，TransactionTemplate 类的实例是线程安全的，因为该实例不维护任何会话状态。 然而，TransactionTemplate 实例会维护配置状态。 因此，虽然多个类可能共享 TransactionTemplate 的单个实例，但如果一个类需要使用具有不同设置（例如，不同的隔离级别）的 TransactionTemplate，则需要创建两个不同的 TransactionTemplate 实例。

#### `TransactionOperator`

```java
public class SimpleService implements Service {

    // single TransactionOperator shared amongst all methods in this instance
    private final TransactionalOperator transactionalOperator;

    // use constructor-injection to supply the ReactiveTransactionManager
    public SimpleService(ReactiveTransactionManager transactionManager) {
        this.transactionOperator = TransactionalOperator.create(transactionManager);
    }

    public Mono<Object> someServiceMethod() {

        // the code in this method runs in a transactional context

        Mono<Object> update = updateOperation1();

        return update.then(resultOfUpdateOperation2).as(transactionalOperator::transactional);
    }
}
```

回调中的代码可以通过调用提供的 ReactiveTransaction 对象上的 setRollbackOnly() 方法来回滚事务，如下所示：

```java
transactionalOperator.execute(new TransactionCallback<>() {

    public Mono<Object> doInTransaction(ReactiveTransaction status) {
        return updateOperation1().then(updateOperation2)
                    .doOnError(SomeBusinessException.class, e -> status.setRollbackOnly());
        }
    }
});
```



```java
public class SimpleService implements Service {

    private final TransactionalOperator transactionalOperator;

    public SimpleService(ReactiveTransactionManager transactionManager) {
        DefaultTransactionDefinition definition = new DefaultTransactionDefinition();

        // the transaction settings can be set here explicitly if so desired
        definition.setIsolationLevel(TransactionDefinition.ISOLATION_READ_UNCOMMITTED);
        definition.setTimeout(30); // 30 seconds
        // and so forth...

        this.transactionalOperator = TransactionalOperator.create(transactionManager, definition);
    }
}
```

#### `TransactionManager`

对于命令式事务，您可以直接使用 org.springframework.transaction.PlatformTransactionManager 来管理您的事务。 为此，请通过 bean 引用将您使用的 PlatformTransactionManager 的实现传递给 bean。 然后，通过使用 TransactionDefinition 和 TransactionStatus 对象，您可以启动事务、回滚和提交。 以下示例显示了如何执行此操作：

```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
// explicitly setting the transaction name is something that can be done only programmatically
def.setName("SomeTxName");
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

TransactionStatus status = txManager.getTransaction(def);
try {
    // put your business logic here
} catch (MyException ex) {
    txManager.rollback(status);
    throw ex;
}
txManager.commit(status);
```

使用反应式事务时，您可以直接使用 org.springframework.transaction.ReactiveTransactionManager 来管理您的事务。 为此，请通过 bean 引用将您使用的 ReactiveTransactionManager 的实现传递给您的 bean。 然后，通过使用 TransactionDefinition 和 ReactiveTransaction 对象，您可以启动事务、回滚和提交。 以下示例显示了如何执行此操作：

```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
// explicitly setting the transaction name is something that can be done only programmatically
def.setName("SomeTxName");
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

Mono<ReactiveTransaction> reactiveTx = txManager.getReactiveTransaction(def);

reactiveTx.flatMap(status -> {

    Mono<Object> tx = ...; // put your business logic here

    return tx.then(txManager.commit(status))
            .onErrorResume(ex -> txManager.rollback(status).then(Mono.error(ex)));
});
```

