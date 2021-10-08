# 容器

Inversion of Control (IoC) ：控制反转

## Spring IoC 容器和 Bean 介绍

控制反转也叫依赖注入(DI)。这是一个过程，对象 通过构造函数参数、工厂方法参数、对象被实例化（构造函数或工厂方法）后设置其属性 来定义依赖项，然后容器在创建bean时注入这些依赖项。在这个过程中，bean控制着自身创建，通过类上的构造函数等机制搜索所需要的依赖，因此被称为依赖反转。

`org.springframework.beans` 和 `org.springframework.context` 包是 Spring Framework 的 IoC 容器的基础。 `BeanFactory` 接口提供了一种能够管理任何类型对象的高级配置机制。 `ApplicationContext` 是 `BeanFactory` 的一个子接口。 它补充：

* 更容易与 Spring 的 AOP 特性集成
* 消息资源处理（用于国际化）
* 事件发布
* 应用层特定上下文，例如用于 Web 应用程序的 `WebApplicationContext`。

简而言之，`BeanFactory` 提供了配置框架和基本功能，`ApplicationContext` 增加了更多企业特定的功能。 `ApplicationContext` 是 `BeanFactory` 的完整超集，在本章中专门用于 Spring 的 IoC 容器的描述。

在 Spring 中，构成应用程序主干并由 Spring IoC 容器管理的对象称为 bean。 bean 是由 Spring IoC 容器实例化、组装和管理的对象。 本质上来说，bean 只是应用程序中众多对象之一。 Bean 以及它们之间的依赖关系反映在容器使用的配置元数据中。

## 容器概览

`org.springframework.context.ApplicationContext` 接口代表 Spring IoC 容器，负责实例化、配置和组装 bean。 容器通过读取配置元数据来获取有关要实例化、配置和组装哪些对象的指令。 配置元数据以 XML、Java 注释或 Java 代码表示。 它可以让您表达组成应用程序的对象以及这些对象之间丰富的相互依赖关系。

Spring 提供了 ApplicationContext 接口的几个实现。 在独立应用程序中，通常创建 `ClassPathXmlApplicationContext` 或 `FileSystemXmlApplicationContext` 的实例。 虽然 XML 一直是定义配置元数据的传统格式，但您可以通过提供少量 XML 配置来声明性地启用对其他元数据格式的支持，从而指示容器使用 Java 注释或代码作为元数据格式。

在大多数应用场景中，不需要显式的用户代码来实例化一个或多个 Spring IoC 容器实例。 例如，在 Web 应用程序场景中，应用程序的 web.xml 文件中简单的八行（左右）样板 XML 通常就足够了（请参阅 Web 应用程序的便捷 ApplicationContext 实例化）。 如果您使用 Spring Tools for Eclipse（一个 Eclipse 驱动的开发环境），您可以通过点击几下鼠标或按键轻松创建这个样板配置。

下图显示了 Spring 如何工作的高级视图。 您的应用程序类与配置元数据相结合，因此在 ApplicationContext 被创建和初始化之后，您就有了一个完全配置且可执行的系统或应用程序。

![container magic](spring-core/container-magic.png)

### 配置元数据

如上图所示，Spring IoC 容器消费配置元数据。 此配置元数据告诉 Spring 容器实例化、配置和组装应用程序中的对象。

配置元数据传统上以简单直观的 XML 格式提供，本章的大部分内容使用这种格式来传达 Spring IoC 容器的关键概念和特性。

有关在 Spring 容器中使用其他形式的元数据的信息，请参阅：

* 基于注解的配置：Spring 2.5 引入了对基于注解的配置元数据的支持。
* 基于java的配置：从 Spring 3.0 开始，Spring JavaConfig 项目提供的许多特性成为核心 Spring Framework 的一部分。 因此，您可以使用 Java 而不是 XML 文件来定义应用程序类外部的 bean。 要使用这些新功能，请参阅 @Configuration、@Bean、@Import 和 @DependsOn 注释。

Spring 配置包含至少一个并且通常不止一个容器必须管理的 bean 定义。 基于 XML 的配置元数据将这些 bean 配置为顶级 <beans/> 元素内的 <bean/> 元素。 Java 配置通常在@Configuration 类中使用@Bean 注释的方法。

这些 bean 定义对应于构成应用程序的实际对象。 通常，您定义服务层对象、数据访问对象 (DAO)、表示对象（例如 Struts Action 实例）、基础结构对象（例如 Hibernate SessionFactories、JMS 队列等）。 通常，不会在容器中配置细粒度的域对象，因为创建和加载域对象通常是 DAO 和业务逻辑的责任。 但是，您可以使用 Spring 与 AspectJ 的集成来配置在 IoC 容器控制之外创建的对象。 请参阅使用 AspectJ 通过 Spring 依赖注入域对象。

以下示例显示了基于 XML 的配置元数据的基本结构：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">  
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>
```

* id 属性是一个字符串，用于标识单个 bean 定义。
* class 属性定义了 bean 的类型并使用了完全限定的Class名称。

### 实例化一个容器

提供给 ApplicationContext 构造函数的一个或多个位置路径是资源字符串，它允许容器从各种外部资源（例如本地文件系统、Java CLASSPATH 等）加载配置元数据。

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

**services.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->

    <bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
        <property name="accountDao" ref="accountDao"/>
        <property name="itemDao" ref="itemDao"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->

</beans>
```

**daos.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountDao"
        class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>
```

在前面的示例中，服务层由 PetStoreServiceImpl 类和 JpaAccountDao 和 JpaItemDao 类型的两个数据访问对象组成（基于 JPA Object-Relational Mapping 标准）。 property name 元素是指 JavaBean 属性的名称，ref 元素是指另一个 bean 定义的名称。 id 和 ref 元素之间的这种联系表达了协作对象之间的依赖关系。 

#### 基于xml的配置元数据

让 bean 定义跨越多个 XML 文件会很有用。 通常，每个单独的 XML 配置文件都代表您架构中的一个逻辑层或模块。

您可以使用应用程序上下文构造函数从所有这些 XML 片段加载 bean 定义。 此构造函数采用多个 Resource 位置，如上一节所示。 或者，使用一个或多个 <import/> 元素从另一个文件或多个文件加载 bean 定义。 以下示例显示了如何执行此操作：

```XML
<beans>
    <import resource="services.xml"/>
    <import resource="resources/messageSource.xml"/>
    <import resource="/resources/themeSource.xml"/>

    <bean id="bean1" class="..."/>
    <bean id="bean2" class="..."/>
</beans>
```

在前面的示例中，外部 bean 定义是从三个文件加载的：services.xml、messageSource.xml 和 themeSource.xml。 所有位置路径都相对于执行导入的定义文件，因此 services.xml 必须与执行导入的文件位于同一目录或类路径位置，而 messageSource.xml 和 themeSource.xml 必须位于该位置下方的resource位置 。 如您所见，**前导斜杠被忽略**。 但是，鉴于这些路径是相对的，最好不使用斜杠。 根据 Spring Schema，被导入文件的内容，包括顶级 <beans/> 元素，必须是有效的 XML bean 定义。

> 可以但不建议使用相对“../”路径引用父目录中的文件。 这样做会创建对当前应用程序之外的文件的依赖关系。 特别是，不建议将此引用用于类路径：URL（例如，类路径：../services.xml），其中运行时解析过程选择“最近的”类路径根，然后查看其父目录。 类路径配置更改可能会导致选择不同的、不正确的目录。
>
> 您始终可以使用完全限定的资源位置而不是相对路径：例如，file:C:/config/services.xml 或 classpath:/config/services.xml。 但是，请注意您将应用程序的配置耦合到特定的绝对位置。 通常最好为这样的绝对位置保持一个间接地址 — ，例如，通过在运行时根据 JVM 系统属性解析的“${… }”占位符。

命名空间本身提供了导入指令功能。 Spring 提供的一系列 XML 命名空间中提供了超出普通 bean 定义的更多配置功能——例如，context 和 util 命名空间。

#### Groovy Bean 定义 DSL

作为外部化配置元数据的另一个示例，bean 定义也可以在 Spring 的 Groovy Bean Definition DSL 中表达，如 Grails 框架中所知。 通常，此类配置位于“.groovy”文件中，其结构如下例所示：

```groovy
beans {
    dataSource(BasicDataSource) {
        driverClassName = "org.hsqldb.jdbcDriver"
        url = "jdbc:hsqldb:mem:grailsDB"
        username = "sa"
        password = ""
        settings = [mynew:"setting"]
    }
    sessionFactory(SessionFactory) {
        dataSource = dataSource
    }
    myService(MyService) {
        nestedBean = { AnotherBean bean ->
            dataSource = dataSource
        }
    }
}
```

这种配置风格在很大程度上等同于 XML bean 定义，甚至支持 Spring 的 XML 配置命名空间。 它还允许通过 importBeans 指令导入 XML bean 定义文件。

### 使用容器

ApplicationContext 是高级工厂的接口，能够维护不同 bean 及其依赖项的注册表。 通过使用方法 T getBean(String name, Class<T> requiredType)，您可以检索 bean 的实例。

ApplicationContext 允许您读取 bean 定义并访问它们，如以下示例所示：

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

使用 Groovy 配置，引导看起来非常相似。 它有一个不同的上下文实现类，它是 Groovy 感知的（但也理解 XML bean 定义）。 以下示例显示了 Groovy 配置：

```java
ApplicationContext context = new GenericGroovyApplicationContext("services.groovy", "daos.groovy");
```

最灵活的变体是 GenericApplicationContext 与读取器委托的结合 — 例如，与 XML 文件的 XmlBeanDefinitionReader 结合使用，如以下示例所示：

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();
```

您还可以对 Groovy 文件使用 GroovyBeanDefinitionReader，如以下示例所示：

```java
GenericApplicationContext context = new GenericApplicationContext();
new GroovyBeanDefinitionReader(context).loadBeanDefinitions("services.groovy", "daos.groovy");
context.refresh();
```

您可以在同一个 ApplicationContext 上混合和匹配此类读取器委托，从不同的配置源读取 bean 定义。

然后，您可以使用 getBean 来检索 bean 的实例。 ApplicationContext 接口有一些其他方法来检索 bean，但理想情况下，您的应用程序代码永远不应该使用它们。 实际上，您的应用程序代码根本不应该调用 getBean() 方法，因此完全不依赖 Spring API。 例如，Spring 与 Web 框架的集成为各种 Web 框架组件（例如控制器和 JSF 管理的 bean）提供了依赖注入，让您可以通过元数据（例如自动装配注释）声明对特定 bean 的依赖。

## Bean概览

Spring IoC 容器管理一个或多个 bean。 这些 bean 是使用您提供给容器的配置元数据创建的（例如，以 XML <bean/> 定义的形式）。

在容器本身内，这些 bean 定义表示为 BeanDefinition 对象，其中包含（除其他信息外）以下元数据：

* 包限定的类名：通常是定义的 bean 的实际实现类。
* Bean 行为配置元素，它说明 Bean 在容器中的行为方式（范围、生命周期回调等）。
* 对 bean 执行其工作所需的其他 bean 的引用。 这些引用也称为协作者或依赖项。
* 要在新创建的对象中设置的其他配置设置 — 例如，池的大小限制或在管理连接池的 bean 中使用的连接数。

此元数据转换为组成每个 bean 定义的一组属性。 下表描述了这些属性：

| Property                 | Explained in…                                                |
| :----------------------- | :----------------------------------------------------------- |
| Class                    | [Instantiating Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-class) |
| Name                     | [Naming Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beanname) |
| Scope                    | [Bean Scopes](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-scopes) |
| Constructor arguments    | [Dependency Injection](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-collaborators) |
| Properties               | [Dependency Injection](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-collaborators) |
| Autowiring mode          | [Autowiring Collaborators](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-autowire) |
| Lazy initialization mode | [Lazy-initialized Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lazy-init) |
| Initialization method    | [Initialization Callbacks](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lifecycle-initializingbean) |
| Destruction method       | [Destruction Callbacks](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lifecycle-disposablebean) |

除了包含有关如何创建特定 bean 的定义之外，ApplicationContext 实现还允许注册在容器外部（由用户创建）创建的现有对象。 这是通过 getBeanFactory() 方法访问 ApplicationContext 的 BeanFactory 来完成的，该方法返回 BeanFactory DefaultListableBeanFactory 实现。 DefaultListableBeanFactory 通过 registerSingleton(..) 和 registerBeanDefinition(..) 方法支持这种注册。 但是，典型的应用程序仅使用通过常规 bean 定义元数据定义的 bean。

> Bean 元数据和手动提供的单例实例需要尽早注册，以便容器在自动装配和其他内省步骤中正确推理它们。 虽然在某种程度上支持覆盖现有元数据和现有单例实例，但官方不支持在运行时注册新 bean（与实时访问工厂同时）并可能导致并发访问异常、bean 容器中的状态不一致，或 两个都。

### bean命名

每个 bean 都有一个或多个标识符。 这些标识符在承载 bean 的容器中必须是唯一的。 一个 bean 通常只有一个标识符。 但是，如果它需要多个，则可以将多余的视为别名。

在基于 XML 的配置元数据中，您可以使用 id 属性、name 属性或两者来指定 bean 标识符。 id 属性允许您指定一个 id。 通常，这些名称是字母加数字（'myBean'、'someService' 等），但它们也可以包含特殊字符。 如果要为 bean 引入其他别名，也可以在 name 属性中指定它们，用逗号 (,)、分号 (;) 或空格分隔。 作为历史记录，在 Spring 3.1 之前的版本中，id 属性被定义为 xsd:ID 类型，它限制了可能的字符。 从 3.1 开始，它被定义为 xsd:string 类型。 请注意，bean id 唯一性仍由容器强制执行，但不再由 XML 解析器强制执行。

您不需要为 bean 提供名称或 ID。 如果您没有明确提供名称或 id，容器将为该 bean 生成一个唯一的名称。 但是，如果要通过名称引用该 bean，通过使用 ref 元素或服务定位器样式查找，则必须提供名称。 

> bean 命名规范：在命名 bean 时对实例字段名称使用标准 Java 约定。 也就是说，bean 名称以小写字母开头，并从那里开始使用驼峰式大小写。 此类名称的示例包括 accountManager、accountService、userDao、loginController 等。
>
> 始终如一地命名 bean 使您的配置更易于阅读和理解。 此外，如果您使用 Spring AOP，一组按名称相关的 bean 时会很有帮助。

通过类路径中的组件扫描，Spring 为未命名的组件生成 bean 名称，遵循前面描述的规则：本质上，采用简单的类名并将其初始字符转换为小写。 但是，在有多个字符且第一个和第二个字符都是大写的（不寻常的）特殊情况下，原始大小写被保留。 这些规则与 java.beans.Introspector.decapitalize（Spring 在这里使用的）定义的规则相同。

#### 在 Bean 定义之外给 Bean 取别名

在 bean 定义本身中，您可以为 bean 提供多个名称，方法是使用 id 属性指定的最多一个名称和 name 属性中任意数量的其他名称的组合。 这些名称可以是同一个 bean 的等效别名，并且在某些情况下很有用，例如让应用程序中的每个组件通过使用特定于该组件本身的 bean 名称来引用公共依赖项。

然而，在实际定义 bean 的地方指定所有别名并不总是足够的。 有时需要为在别处定义的 bean 引入别名。 这在大型系统中很常见，其中配置在每个子系统之间拆分，每个子系统都有自己的一组对象定义。 在基于 XML 的配置元数据中，您可以使用 <alias/> 元素来完成此操作。 以下示例显示了如何执行此操作：

```xml
<alias name="fromName" alias="toName"/>
```

在这种情况下，一个名为 fromName 的 bean（在同一个容器中）在使用这个别名定义之后，也可以被称为 toName。

例如，子系统 A 的配置元数据可以通过subsystemA-dataSource名称来引用数据源。 子系统 B 的配置元数据可以通过subsystemB-dataSource的名称来引用数据源。 在组合使用这两个子系统的主应用程序时，主应用程序通过名称 myApp-dataSource 引用 DataSource。 要让所有三个名称都引用同一个对象，您可以将以下别名定义添加到配置元数据中：

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```

现在，每个组件和主应用程序都可以通过一个唯一的名称来引用 dataSource，并且保证不会与任何其他定义发生冲突（有效地创建一个命名空间），但它们引用的是同一个 bean。

> 如果使用 Javaconfiguration，则可以使用 @Bean 批注来提供别名。 有关详细信息，请参阅使用 @Bean 注释。

### 实例化bean

bean 定义本质上是创建一个或多个对象的食谱。 当被询问时，容器会查看命名 bean 的配方，并使用该 bean 定义封装的配置元数据来创建（或获取）实际对象。

如果使用基于 XML 的配置元数据，则要在 <bean/> 元素的 class 属性中指定实例化的对象的类型（或类）。 这个类属性（在内部，它是 BeanDefinition 实例上的 Class 属性）通常是强制性的。 （对于例外情况，请参阅使用实例工厂方法和 Bean 定义继承进行实例化。）您可以通过以下两种方式之一使用 Class 属性：

* 通常，在容器本身通过反射调用其构造函数直接创建 bean 的情况下，指定要构造的 bean 类，有点等同于带有 new 运算符的 Java 代码。
* 不太常见的情况下，指定包含被调用以创建对象的静态工厂方法的实际类，容器调用类上的静态工厂方法来创建 bean。 调用静态工厂方法返回的对象类型可能是同一个类，也可能完全是另一个类。

> 嵌套类名：如果要为嵌套类配置 bean 定义，可以使用嵌套类的二进制名称或源名称。例如，如果您在 com.example 包中有一个名为 SomeThing 的类，并且这个 SomeThing 类有一个名为 OtherThing 的静态嵌套类，则它们可以用美元符号 (`$`) 或点 (.) 分隔。 因此，bean 定义中类属性的值将是 `com.example.SomeThing$​OtherThing` 或 `com.example.SomeThing.OtherThing`。

#### 构造函数实例化bean

当您通过构造函数方法创建 bean 时，所有普通类都可以被 Spring 使用并与 Spring 兼容。 也就是说，正在开发的类不需要实现任何特定的接口或以特定的方式进行编码。 只需指定 bean 类就足够了。 但是，根据您对该特定 bean 使用的 IoC 类型，您可能需要一个默认（空）构造函数。

Spring IoC 容器几乎可以管理您希望它管理的任何类。 它不仅限于管理真正的 JavaBean。 大多数 Spring 用户更喜欢实际的 JavaBeans，它只有一个默认（无参数）构造函数和适当的 setter 和 getter，它们以容器中的属性为模型。 您还可以在您的容器中拥有更多异国情调的非 bean 风格的类。 例如，如果您需要使用绝对不符合 JavaBean 规范的遗留连接池，Spring 也可以管理它。

使用基于 XML 的配置元数据，您可以按如下方式指定 bean 类：

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>

<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```

#### 静态工厂方法实例化bean

定义使用静态工厂方法创建的 bean 时，使用 class 属性指定包含静态工厂方法的类，并使用名为 factory-method 的属性指定工厂方法本身的名称。 您应该能够调用此方法（带有可选参数，如下所述）并返回一个活动对象，随后将其视为通过构造函数创建的。 这种 bean 定义的一种用途是在遗留代码中调用静态工厂。

以下 bean 定义指定通过调用工厂方法来创建 bean。 定义中没有指定返回对象的类型（类），只指定包含工厂方法的类。 在这个例子中，createInstance() 方法必须是一个静态方法。 以下示例显示了如何指定工厂方法：

```xml
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/>
```

以下示例显示了一个可以与前面的 bean 定义一起使用的类：

```java
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        return clientService;
    }
}
```

#### 实例工厂方法实例化bean

与通过静态工厂方法进行实例化类似，使用实例工厂方法进行实例化会从容器中调用现有 bean 的非静态方法来创建新 bean。 要使用此机制，请将 class 属性留空，并在 factory-bean 属性中指定当前（或父或祖先）容器中 bean 的名称，该容器包含要调用以创建对象的实例方法。 使用 factory-method 属性设置工厂方法本身的名称。 以下示例显示了如何配置此类 bean：

```java
<!-- the factory bean, which contains a method called createInstance() -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>
```

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }
}
```

一个工厂类也可以包含多个工厂方法，如下例所示：

```xml
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>

<bean id="accountService"
    factory-bean="serviceLocator"
    factory-method="createAccountServiceInstance"/>
```

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    private static AccountService accountService = new AccountServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }

    public AccountService createAccountServiceInstance() {
        return accountService;
    }
}
```

#### 确定 Bean 的运行时类型

确定特定 bean 的运行时类型并非易事。 bean 元数据定义中的指定类只是一个初始类引用，声明的工厂方法、FactoryBean 方式创建实例，这些导致 bean 的不同运行时类型。 此外，AOP 代理可以使用基于接口的代理包装 bean 实例，并限制暴露目标 bean 的实际类型（接口代理的方式）。

找出特定 bean 的实际运行时类型的推荐方法是： BeanFactory.getBean(beanName,beanType),该方法考虑了上述所有情况。

## 依赖

### 依赖注入

依赖注入 (DI) 是一个过程，其中对象仅通过构造函数参数、静态工厂方法的参数或在对象实例被构造后注入这些依赖项或 从实例工厂方法返回。 这个过程基本上是 bean 本身的逆过程（因此得名，控制反转），通过使用类的直接构造或服务定位器模式自行控制其依赖项的实例化。

DI 原则使代码更清晰，当对象提供依赖关系时，解耦更有效。 该对象不查找其依赖项，也不知道依赖项的位置或类。 因此，您的类变得更容易测试，特别是当依赖项位于接口或抽象基类上时，这允许在单元测试中使用存根或模拟实现。

#### 构造函数注入

基于构造函数的 DI 是通过容器调用具有多个参数的构造函数来完成的，每个参数代表一个依赖项。 调用带有特定参数的静态工厂方法来构造 bean 几乎是等效的，本讨论将类似地处理构造函数和静态工厂方法的参数。 以下示例显示了一个只能使用构造函数注入进行依赖注入的类：

```java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on a MovieFinder
    private final MovieFinder movieFinder;

    // a constructor so that the Spring container can inject a MovieFinder
    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```

请注意，这个类没有什么特别之处。 它是一个不依赖于容器特定接口、基类或注解的 POJO。

##### 构造函数参数解析

构造函数参数解析匹配通过使用参数的类型发生。 如果 bean 定义的构造函数参数中不存在潜在的歧义，那么在 bean 定义中定义构造函数参数的顺序就是在实例化 bean 时将这些参数提供给适当的构造函数的顺序。 考虑以下类：

```java
package x.y;

public class ThingOne {

    public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
        // ...
    }
}
```

假设 ThingTwo 和 ThingThree 类没有通过继承关联，则不存在潜在的歧义。 因此，以下配置工作正常，您不需要在 <constructor-arg/> 元素中显式指定构造函数参数索引或类型。

```xml
<beans>
    <bean id="beanOne" class="x.y.ThingOne">
        <constructor-arg ref="beanTwo"/>
        <constructor-arg ref="beanThree"/>
    </bean>

    <bean id="beanTwo" class="x.y.ThingTwo"/>

    <bean id="beanThree" class="x.y.ThingThree"/>
</beans>
```

当另一个 bean 被引用时，类型是已知的，并且可以发生匹配（就像前面的例子一样）。 当使用简单类型时，例如 <value>true</value>，Spring 无法确定值的类型，因此无法在没有帮助的情况下按类型进行匹配。 考虑以下类：

```java
package examples;

public class ExampleBean {

    // Number of years to calculate the Ultimate Answer
    private final int years;

    // The Answer to Life, the Universe, and Everything
    private final String ultimateAnswer;

    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

**构造函数参数类型匹配**

在上述场景中，如果您使用 type 属性显式指定构造函数参数的类型，则容器可以使用简单类型的类型匹配，如下例所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
```

**构造函数索引**匹配

您可以使用 index 属性明确指定构造函数参数的索引，如以下示例所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg index="0" value="7500000"/>
    <constructor-arg index="1" value="42"/>
</bean>
```

除了解决多个简单值的歧义之外，指定索引还可以解决构造函数具有两个相同类型参数的歧义。

**构造参数名称匹配**

您还可以使用构造函数参数名称进行值消歧，如以下示例所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```

请记住，要使这项工作开箱即用，您的代码必须在启用调试标志的情况下进行编译，以便 Spring 可以从构造函数中查找参数名称。 如果您不能或不想使用调试标志编译代码，则可以使用 @ConstructorProperties JDK 注释显式命名构造函数参数。 示例类必须如下所示：

```java
package examples;

public class ExampleBean {

    // Fields omitted

    @ConstructorProperties({"years", "ultimateAnswer"})
    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

#### 基于setter方式的注入

基于 Setter 的 DI 是通过容器在调用无参数构造函数或无参数静态工厂方法来实例化 bean 后调用 bean 上的 setter 方法来完成的。

以下示例显示了一个只能使用纯 setter 注入进行依赖注入的类。 这个类是传统的Java。 它是一个不依赖于容器特定接口、基类或注解的 POJO。

```java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on the MovieFinder
    private MovieFinder movieFinder;

    // a setter method so that the Spring container can inject a MovieFinder
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```

ApplicationContext 为其管理的 bean 支持基于构造函数和基于 setter 的 DI。 在已经通过构造函数方法注入了一些依赖项之后，它还支持基于 setter 的 DI。 您以 BeanDefinition 的形式配置依赖项，将其与 PropertyEditor 实例结合使用以将属性从一种格式转换为另一种格式。 但是，大多数 Spring 用户并不直接（即以编程方式）使用这些类，而是使用 XML bean 定义、基于 Java 的 @Configuration 类带注释的组件（即用 @Component、@Controller 等注释的类 或 @Bean 方法)。  然后这些源在内部转换为 BeanDefinition 的实例，并用于加载整个 Spring IoC 容器实例。

> 基于构造函数还是基于 setter 的 DI？
>
> 由于您可以混合使用基于构造函数和基于 setter 的 DI，因此根据经验，对强制依赖项使用构造函数，对可选依赖项使用 setter 方法或配置方法是一个很好的经验法则。 请注意，在 setter 方法上使用 @Required 注释可用于使属性成为必需的依赖项； 但是，最好使用带有参数编程验证的构造函数注入。
>
> Spring 团队通常提倡构造函数注入，因为它可以让您将应用程序组件实现为不可变对象，并确保所需的依赖项不为空。 此外，构造函数注入的组件总是以完全初始化的状态返回给客户端（调用）代码。 作为旁注，大量的构造函数参数是一种糟糕的代码味道，这意味着该类可能有太多的责任，应该重构以更好地解决适当的关注点分离问题。
>
> Setter 注入应该主要仅用于可以在类中分配合理默认值的可选依赖项。 否则，必须在代码使用依赖项的任何地方执行非空检查。 setter 注入的一个好处是 setter 方法使该类的对象可以在以后重新配置或重新注入。 因此，通过 JMX MBean 进行管理是 setter 注入的一个引人注目的用例。
>
>  有时，在处理您没有源码的第三方类时，需要您自己做出的选择。 例如，如果第三方类不公开任何 setter 方法，则构造函数注入可能是 DI 的唯一可用形式。

#### 依赖解析过程

容器执行bean依赖解析如下：

* ApplicationContext 是使用描述所有 bean 的配置元数据创建和初始化的。 配置元数据可以由 XML、Java 代码或注释指定。
* 对于每个 bean，它的依赖关系以属性、构造函数参数或静态工厂方法的参数（如果您使用它而不是普通构造函数）的形式表示。 在实际创建 bean 时，将这些依赖关系提供给 bean。
* 每个属性或构造函数参数都是要设置的值的实际定义，或者是对容器中另一个 bean 的引用。
* 作为值的每个属性或构造函数参数都从其指定格式转换为该属性或构造函数参数的实际类型。 默认情况下，Spring 可以将字符串格式提供的值转换为所有内置类型，例如 int、long、String、boolean 等。

Spring 容器在创建容器时验证每个 bean 的配置。 但是，在实际创建 bean 之前不会设置 bean 属性本身。 创建容器时会创建单例范围并设置为预实例化（默认）的 Bean。 范围在 Bean 范围中定义。 否则，仅在请求时才创建 bean。 创建 bean 可能会导致创建 bean 图，因为 bean 的依赖项及其依赖项的依赖项（等等）被创建和分配。 

> 循环依赖
>
> 如果您主要使用构造函数注入，则可能会出现无法解决的循环依赖场景。例如：A类通过构造函数注入B类的实例，B类通过构造函数注入A类的实例。 如果您将类 A 和 B 的 bean 配置为相互注入，则 Spring IoC 容器在运行时检测到此循环引用，并抛出 BeanCurrentlyInCreationException。
>
> 一种可能的解决方案是编辑一些类的源代码，以便由 setter 而不是构造函数来配置。 或者，避免构造函数注入并仅使用 setter 注入。 也就是说，虽然不推荐，但是可以通过setter注入来配置循环依赖。
>
> 与典型情况（没有循环依赖）不同，bean A 和 bean B 之间的循环依赖迫使其中一个 bean 在完全初始化之前注入另一个 bean（经典的鸡和蛋场景）。

您通常可以相信 Spring 会做正确的事情。 它在容器加载时检测配置问题，例如对不存在的 bean 的引用和循环依赖。 Spring 在真正创建 bean 时尽可能晚地设置属性并解析依赖项。 这意味着，如果创建该对象或其依赖项之一时出现问题，spring 容器仍然正常启动，只在你调用改bean时，才会告诉你异常。

在容器启动时就创建bean,虽然会花费启动时间和系统内存，但是可以提前发现配置文件中的错误。你可以覆盖此默认的early方式，以便bean延时初始化。

如果不存在循环依赖，当一个或多个协作 bean 被注入依赖 bean 时，每个协作 bean 在注入依赖 bean 之前都已完全配置。 这意味着，如果 bean A 依赖 bean B，则 Spring IoC 容器在调用 bean A 上的 setter 方法之前完全配置 bean B。换句话说，bean 被实例化（如果它不是预实例化的单例） )，设置其依赖，调用相关生命周期方法（如配置的init方法或InitializingBean回调方法）。

#### 依赖注入示例

以下示例将基于 XML 的配置元数据用于基于 setter 的 DI。 Spring XML 配置文件的一小部分指定了一些 bean 定义，如下所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- setter injection using the nested ref element -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>

    <!-- setter injection using the neater ref attribute -->
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

以下示例显示了相应的 ExampleBean 类：

```java
public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public void setBeanOne(AnotherBean beanOne) {
        this.beanOne = beanOne;
    }

    public void setBeanTwo(YetAnotherBean beanTwo) {
        this.beanTwo = beanTwo;
    }

    public void setIntegerProperty(int i) {
        this.i = i;
    }
}
```

在前面的示例中，setter 被声明为与 XML 文件中指定的属性匹配。 以下示例使用基于构造函数的 DI：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- constructor injection using the nested ref element -->
    <constructor-arg>
        <ref bean="anotherExampleBean"/>
    </constructor-arg>

    <!-- constructor injection using the neater ref attribute -->
    <constructor-arg ref="yetAnotherBean"/>

    <constructor-arg type="int" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

以下示例显示了相应的 ExampleBean 类：

```java
public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public ExampleBean(
        AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {
        this.beanOne = anotherBean;
        this.beanTwo = yetAnotherBean;
        this.i = i;
    }
}
```

bean 定义中指定的构造函数参数用作 ExampleBean 的构造函数的参数。

现在考虑这个例子的一个变体，其中不使用构造函数，而是告诉 Spring 调用静态工厂方法来返回对象的实例：

```xml
<bean id="exampleBean" class="examples.ExampleBean" factory-method="createInstance">
    <constructor-arg ref="anotherExampleBean"/>
    <constructor-arg ref="yetAnotherBean"/>
    <constructor-arg value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

以下示例显示了相应的 ExampleBean 类：

```java
public class ExampleBean {

    // a private constructor
    private ExampleBean(...) {
        ...
    }

    // a static factory method; the arguments to this method can be
    // considered the dependencies of the bean that is returned,
    // regardless of how those arguments are actually used.
    public static ExampleBean createInstance (
        AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {

        ExampleBean eb = new ExampleBean (...);
        // some other operations...
        return eb;
    }
}
```

静态工厂方法的参数由 <constructor-arg/> 元素提供，与实际使用构造函数完全相同。 工厂方法返回的类的类型不必与包含静态工厂方法的类的类型相同（尽管在本示例中是）。 实例（非静态）工厂方法可以以基本相同的方式使用（除了使用 factory-bean 属性而不是 class 属性），因此我们不在这里讨论这些细节。

### 依赖和配置细节

如上一节所述，您可以将 bean 属性和构造函数参数定义为对其他托管 bean（协作者）的引用或作为内联定义的值。 为此，Spring 的基于 XML 的配置元数据支持其 <property/> 和 <constructor-arg/> 元素中的子元素类型。

#### 直接值（原语、字符串等）

<property/> 元素的 value 属性将属性或构造函数参数指定为人类可读的字符串表示形式。 Spring 的转换服务用于将这些值从 String 转换为属性或参数的实际类型。 以下示例显示了正在设置的各种值：

```xml
<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <!-- results in a setDriverClassName(String) call -->
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
    <property name="username" value="root"/>
    <property name="password" value="misterkaoli"/>
</bean>
```

以下示例使用 p-namespace 进行更简洁的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource"
        destroy-method="close"
        p:driverClassName="com.mysql.jdbc.Driver"
        p:url="jdbc:mysql://localhost:3306/mydb"
        p:username="root"
        p:password="misterkaoli"/>

</beans>
```

前面的 XML 更简洁。 但是，拼写错误是在运行时而不是设计时发现的，除非您在创建 bean 定义时使用支持自动属性完成的 IDE（例如 IntelliJ IDEA 或 Spring Tools for Eclipse）。 强烈建议使用此类 IDE 帮助。

您还可以配置一个 java.util.Properties 实例，如下所示：

```xml
<bean id="mappings"
    class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">

    <!-- typed as a java.util.Properties -->
    <property name="properties">
        <value>
            jdbc.driver.className=com.mysql.jdbc.Driver
            jdbc.url=jdbc:mysql://localhost:3306/mydb
        </value>
    </property>
</bean>
```

Spring 容器使用 JavaBeans PropertyEditor 机制将 <value/> 元素内的文本转换为 java.util.Properties 实例。 这是一个很好的捷径，并且是 Spring 团队支持使用嵌套 <value/> 元素而不是 value 属性样式的少数几个地方之一。

**idref 元素**

idref 元素只是一种将容器中另一个 bean 的 id（字符串值 - 而不是引用）传递给 <constructor-arg/> 或 <property/> 元素的防错方式。 以下示例显示了如何使用它：

```xml
<bean id="theTargetBean" class="..."/>

<bean id="theClientBean" class="...">
    <property name="targetName">
        <idref bean="theTargetBean"/>
    </property>
</bean>
```

前面的 bean 定义片段与以下片段完全等效（在运行时）：

```xml
<bean id="theTargetBean" class="..." />

<bean id="client" class="...">
    <property name="targetName" value="theTargetBean"/>
</bean>
```

第一种形式比第二种形式更可取，因为使用 idref 标记让容器在部署时验证引用的命名 bean 实际存在。 在第二个变体中，不对传递给客户端 bean 的 targetName 属性的值执行验证。 只有在实际实例化客户端 bean 时才会发现拼写错误（最有可能是致命的结果）。 如果客户端 bean 是prototype  bean，则可能只有在部署容器很久之后才能发现此错误和由此产生的异常。

<idref/> 元素带来价值的一个常见地方（至少在 Spring 2.0 之前的版本中）是在 ProxyFactoryBean bean 定义中的 AOP 拦截器的配置中。 在指定拦截器名称时使用 <idref/> 元素可防止您拼错拦截器 ID。

#### 引用其他bean

ref 元素是 <constructor-arg/> 或 <property/> 定义元素中的最后一个元素。 在这里，您将 bean 的指定属性的值设置为对容器管理的另一个 bean（协作者）的引用。 被引用的 bean 是要设置其属性的 bean 的依赖项，在设置属性之前根据需要对其进行初始化。 （如果协作者是一个单例 bean，它可能已经被容器初始化。）所有引用最终都是对另一个对象的引用。 范围和验证取决于您是否通过 bean 或 parent 属性指定其他对象的 ID 或名称。

通过 <ref/> 标记的 bean 属性指定目标 bean 是最通用的形式，它允许创建对同一容器或父容器中的任何 bean 的引用，无论它是否在同一 XML 文件中。 bean 属性的值可能与目标bean 的id 属性相同，也可能与目标bean 的name 属性中的值之一相同。 以下示例显示了如何使用 ref 元素：

```xml
<ref bean="someBean"/>
```

通过 parent 属性指定目标 bean 会创建对当前容器的父容器中的 bean 的引用。 parent 属性的值可能与目标 bean 的 id 属性或目标 bean 的 name 属性中的值之一相同。 目标 bean 必须在当前容器的父容器中。 您应该主要在具有容器层次结构并且希望使用与父 bean 同名的代理将现有 bean 包装在父容器中时使用此 bean 引用变体。 以下清单显示了如何使用 parent 属性：

```xml
<!-- in the parent context -->
<bean id="accountService" class="com.something.SimpleAccountService">
    <!-- insert dependencies as required here -->
</bean>
```

```xml
<!-- in the child (descendant) context -->
<bean id="accountService" <!-- bean name is the same as the parent bean -->
    class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target">
        <ref parent="accountService"/> <!-- notice how we refer to the parent bean -->
    </property>
    <!-- insert other configuration and dependencies as required here -->
</bean>
```

#### 内部bean

<property/> 或 <constructor-arg/> 元素内的 <bean/> 元素定义了一个内部 bean，如以下示例所示：

```xml
<bean id="outer" class="...">
    <!-- instead of using a reference to a target bean, simply define the target bean inline -->
    <property name="target">
        <bean class="com.example.Person"> <!-- this is the inner bean -->
            <property name="name" value="Fiona Apple"/>
            <property name="age" value="25"/>
        </bean>
    </property>
</bean>
```

内部 bean 定义不需要定义的 ID 或name。 如果指定，容器不会使用这样的值作为标识符。 容器在创建时也会忽略范围标志，因为内部 bean 始终是匿名的，并且始终与外部 bean 一起创建。 不可能独立访问内部 bean 或将它们注入除封闭 bean 之外的协作 bean 中。

#### 集合

<list/>、<set/>、<map/> 和 <props/> 元素分别设置 Java 集合类型 List、Set、Map 和 Properties 的属性和参数。 以下示例显示了如何使用它们：

```xml
<bean id="moreComplexObject" class="example.ComplexObject">
    <!-- results in a setAdminEmails(java.util.Properties) call -->
    <property name="adminEmails">
        <props>
            <prop key="administrator">administrator@example.org</prop>
            <prop key="support">support@example.org</prop>
            <prop key="development">development@example.org</prop>
        </props>
    </property>
    <!-- results in a setSomeList(java.util.List) call -->
    <property name="someList">
        <list>
            <value>a list element followed by a reference</value>
            <ref bean="myDataSource" />
        </list>
    </property>
    <!-- results in a setSomeMap(java.util.Map) call -->
    <property name="someMap">
        <map>
            <entry key="an entry" value="just some string"/>
            <entry key="a ref" value-ref="myDataSource"/>
        </map>
    </property>
    <!-- results in a setSomeSet(java.util.Set) call -->
    <property name="someSet">
        <set>
            <value>just some string</value>
            <ref bean="myDataSource" />
        </set>
    </property>
</bean>
```

##### 集合合并

Spring 容器还支持合并集合。 应用程序开发人员可以定义父 <list/>、<map/>、<set/> 或 <props/> 元素并拥有子 <list/>、<map/>、<set/> 或 <props/> 元素 从父集合继承和覆盖值。 也就是说，子集合的值是合并父集合和子集合的元素的结果，子集合元素覆盖父集合中指定的值。

关于合并的这一节讨论了父子 bean 机制。 不熟悉父和子 bean 定义的读者可能希望在继续之前阅读相关部分。

以下示例演示了集合合并：

```xml
<beans>
    <bean id="parent" abstract="true" class="example.ComplexObject">
        <property name="adminEmails">
            <props>
                <prop key="administrator">administrator@example.com</prop>
                <prop key="support">support@example.com</prop>
            </props>
        </property>
    </bean>
    <bean id="child" parent="parent">
        <property name="adminEmails">
            <!-- the merge is specified on the child collection definition -->
            <props merge="true">
                <prop key="sales">sales@example.com</prop>
                <prop key="support">support@example.co.uk</prop>
            </props>
        </property>
    </bean>
<beans>
```

请注意在子 bean 定义的 adminEmails 属性的 <props/> 元素上使用了 merge=true 属性。 当容器解析并实例化子 bean 时，生成的实例具有一个 adminEmails Properties 集合，该集合包含将子 bean 的 adminEmails 集合与父级的 adminEmails 集合合并的结果。 以下清单显示了结果：

```properties
administrator=administrator@example.com
sales=sales@example.com
support=support@example.co.uk
```

子 Properties 集合的值集继承了父 <props/> 的所有属性元素，支持值的子值覆盖了父集合中的值。

这种合并行为同样适用于 <list/>、<map/> 和 <set/> 集合类型。 在 <list/> 元素的特定情况下，与 List 集合类型（即值的有序集合的概念）相关联的语义得到维护。 父级的值在所有子级列表的值之前。 对于 Map、Set 和 Properties 集合类型，不存在排序。 因此，对于作为容器内部使用的关联 Map、Set 和 Properties 实现类型基础的集合类型，没有任何有效排序语义。

##### 强类型集合

随着 Java 5 中泛型类型的引入，您可以使用强类型集合。 也就是说，可以声明一个 Collection 类型，使其只能包含（例如）String 元素。 如果您使用 Spring 将强类型 Collection 依赖注入到 bean 中，则可以利用 Spring 的类型转换支持。 以下 Java 类和 bean 定义显示了如何执行此操作：

```java
public class SomeClass {

    private Map<String, Float> accounts;

    public void setAccounts(Map<String, Float> accounts) {
        this.accounts = accounts;
    }
}
```

```xml
<beans>
    <bean id="something" class="x.y.SomeClass">
        <property name="accounts">
            <map>
                <entry key="one" value="9.99"/>
                <entry key="two" value="2.75"/>
                <entry key="six" value="3.99"/>
            </map>
        </property>
    </bean>
</beans>
```

当something bean 的accounts 属性准备注入时，关于强类型Map<String, Float> 元素类型的泛型信息可通过反射获得。 因此，Spring 的类型转换基础结构将各种值元素识别为 Float 类型，并将字符串值（9.99、2.75 和 3.99）转换为实际的 Float 类型。

#### 空字符串

Spring 将属性等的空参数视为空字符串。 以下基于 XML 的配置元数据片段将电子邮件属性设置为空字符串值 ("")。

```xml
<bean class="ExampleBean">
    <property name="email" value=""/>
</bean>
```

前面的示例等效于以下 Java 代码：

```java
exampleBean.setEmail("");
```

<null/> 元素处理空值。 以下清单显示了一个示例：

```xml
<bean class="ExampleBean">
    <property name="email">
        <null/>
    </property>
</bean>
```

上面的配置相当于下面的Java代码：

```java
exampleBean.setEmail(null);
```

##### 带有 p 命名空间的 XML 快捷方式

p-namespace 允许您使用 bean 元素的属性（而不是嵌套的 <property/> 元素）来描述协作 bean 的属性值，或两者兼而有之。

Spring 支持具有命名空间的可扩展配置格式，这些格式基于 XML 模式定义。 本章讨论的 bean 配置格式是在 XML Schema 文档中定义的。 但是，p 命名空间并未在 XSD 文件中定义，仅存在于 Spring 的核心中。

以下示例显示了两个解析为相同结果的 XML 片段（第一个使用标准 XML 格式，第二个使用 p 命名空间）：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="classic" class="com.example.ExampleBean">
        <property name="email" value="someone@somewhere.com"/>
    </bean>

    <bean name="p-namespace" class="com.example.ExampleBean"
        p:email="someone@somewhere.com"/>
</beans>
```

该示例显示了在 bean 定义中名为 email 的 p 命名空间中的一个属性。 这告诉 Spring 包含一个属性声明。 如前所述，p 命名空间没有模式定义，因此您可以将属性的名称设置为属性名称。

下一个示例包括另外两个 bean 定义，它们都引用了另一个 bean：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="john-classic" class="com.example.Person">
        <property name="name" value="John Doe"/>
        <property name="spouse" ref="jane"/>
    </bean>

    <bean name="john-modern"
        class="com.example.Person"
        p:name="John Doe"
        p:spouse-ref="jane"/>

    <bean name="jane" class="com.example.Person">
        <property name="name" value="Jane Doe"/>
    </bean>
</beans>
```

此示例不仅包括使用 p 命名空间的属性值，而且还使用特殊格式来声明属性引用。 第一个 bean 定义使用 <property name="spouse" ref="jane"/> 创建从 bean john 到 bean jane 的引用，而第二个 bean 定义使用 p:spouse-ref="jane" 作为属性来做 完全一样的东西。 在这种情况下，spouse是属性名称，而 -ref 部分表示这不是一个直接值，而是对另一个 bean 的引用。

#### 带有 c 命名空间的 XML 快捷方式

与带有 p-namespace 的 XML Shortcut 类似，Spring 3.1 中引入的 c-namespace 允许内联属性来配置构造函数参数，而不是嵌套的 constructor-arg 元素。

下面的例子使用 c: namespace来做与 构造函数注入 相同的事情：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:c="http://www.springframework.org/schema/c"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="beanTwo" class="x.y.ThingTwo"/>
    <bean id="beanThree" class="x.y.ThingThree"/>

    <!-- traditional declaration with optional argument names -->
    <bean id="beanOne" class="x.y.ThingOne">
        <constructor-arg name="thingTwo" ref="beanTwo"/>
        <constructor-arg name="thingThree" ref="beanThree"/>
        <constructor-arg name="email" value="something@somewhere.com"/>
    </bean>

    <!-- c-namespace declaration with argument names -->
    <bean id="beanOne" class="x.y.ThingOne" c:thingTwo-ref="beanTwo"
        c:thingThree-ref="beanThree" c:email="something@somewhere.com"/>

</beans>
```

c: namespace使用与 p: one（用于 bean 引用的尾随 -ref）相同的约定，用于通过名称设置构造函数参数。 同样，它需要在 XML 文件中声明，即使它没有在 XSD 模式中定义（它存在于 Spring 核心中）。

对于构造函数参数名称不可用的极少数情况（通常如果字节码是在没有调试信息的情况下编译的），您可以使用参数索引的回退，如下所示：

```xml
<!-- c-namespace index declaration -->
<bean id="beanOne" class="x.y.ThingOne" c:_0-ref="beanTwo" c:_1-ref="beanThree"
    c:_2="something@somewhere.com"/>
```

> 由于 XML 语法的原因，索引符号需要存在前导 _，因为 XML 属性名称不能以数字开头（即使某些 IDE 允许）。 相应的索引符号也可用于 <constructor-arg> 元素，但不常用，因为声明的简单顺序通常就足够了。

实际上，构造函数解析机制在匹配参数方面非常有效，因此除非您确实需要，否则我们建议在整个配置中使用名称表示法。

#### 复合属性名称

您可以在设置 bean 属性时使用复合或嵌套属性名称，只要路径中除最终属性名称之外的所有组件都不为空即可。 考虑以下 bean 定义：

```xml
<bean id="something" class="things.ThingOne">
    <property name="fred.bob.sammy" value="123" />
</bean>
```

something bean 有一个 fred 属性，它有一个 bob 属性，它有一个 sammy 属性，而最终的 sammy 属性被设置为 123。为了使其工作，something 的 fred 属性和 bob 属性 构造bean 后，fred 不能为空。 否则，抛出 NullPointerException。

### 使用`depends-on`

如果一个 bean 是另一个 bean 的依赖项，这通常意味着一个 bean 被设置为另一个 bean 的属性。 通常，您使用基于 XML 的配置元数据中的 <ref/> 元素来完成此操作。 但是，有时 bean 之间的依赖关系不那么直接。 例如，当需要触发类中的静态初始化程序时，例如数据库驱动程序注册。 在初始化使用此元素的 bean 之前，depends-on 属性可以显式地强制初始化一个或多个 bean。 以下示例使用depends-on 属性来表达对单个bean 的依赖：

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager"/>
<bean id="manager" class="ManagerBean" />
```

要表达对多个 bean 的依赖，请提供 bean 名称列表作为依赖属性的值（逗号、空格和分号是有效的分隔符）：

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
    <property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```

### 懒加载bean

默认情况下，ApplicationContext 实现会在初始化过程中急切地创建和配置所有单例 bean。 通常，这种预实例化是可取的，因为可以立即发现配置或周围环境中的错误，而不是在几小时甚至几天之后。 当这种行为不可取时，您可以通过将 bean 定义标记为延迟初始化来防止单例 bean 的预实例化。 一个延迟初始化的 bean 告诉 IoC 容器在它第一次被请求时创建一个 bean 实例，而不是在启动时。

在 XML 中，此行为由 <bean/> 元素上的 lazy-init 属性控制，如以下示例所示：

```xml
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```

当 ApplicationContext 使用前面的配置时，当 ApplicationContext 启动时，惰性 bean 不会被预先实例化，而 not.lazy bean 会被预先实例化。

但是，当延迟初始化的 bean 是未延迟初始化的单例 bean 的依赖项时，ApplicationContext 在启动时创建延迟初始化的 bean，因为它必须满足单例的依赖项。 延迟初始化的 bean 被注入到其他没有延迟初始化的单例 bean 中。

您还可以通过使用 <beans/> 元素上的 default-lazy-init 属性在容器级别控制延迟初始化，如以下示例所示：

```xml
<beans default-lazy-init="true">
    <!-- no beans will be pre-instantiated... -->
</beans>
```

### 自动装配

Spring 容器可以自动装配协作 bean 之间的关系。 通过检查 ApplicationContext 的内容，您可以让 Spring 自动为您的 bean 解析协作者（其他 bean）。 自动装配具有以下优点：

* 自动装配可以显着减少指定属性或构造函数参数的需要。 
* 自动装配可以随着对象的发展更新配置。 例如，如果您需要向类添加依赖项，则无需修改配置即可自动满足该依赖项。 因此，自动装配在开发过程中特别有用，当代码库变得更稳定时，不会否定切换到显式装配的选项。

使用基于 XML 的配置元数据时（请参阅依赖注入），您可以使用 <bean/> 元素的 autowire 属性为 bean 定义指定自动装配模式。 自动装配功能有四种模式。 您可以为每个 bean 指定自动装配，因此可以选择要自动装配的那些。 下表描述了四种自动装配模式：

* no:（默认）没有自动装配。 Bean 引用必须由 ref 元素定义。 对于较大的部署，不建议更改默认设置，因为明确指定协作者可以提供更好的控制和清晰度。 在某种程度上，它记录了系统的结构。
* byName:按属性名称自动装配。 Spring 查找与需要自动装配的属性同名的 bean。 例如，如果一个 bean 定义被设置为按名称自动装配并且它包含一个master属性（即它有一个 setMaster(..) 方法），Spring 会查找一个名为 master 的 bean 定义并使用它来设置属性。
* byType:如果容器中只存在一个属性类型的 bean，则让属性自动装配。 如果存在多个，则会引发致命异常，这表明您不能为该 bean 使用 byType 自动装配。 如果没有匹配的 bean，则不会发生任何事情（未设置属性）。
* constructor:类似于 byType 但适用于构造函数参数。 如果容器中没有一个构造函数参数类型的 bean，则会引发致命错误。

使用 byType 或构造函数自动装配模式，您可以连接数组和类型化集合。 在这种情况下，提供容器内与预期类型匹配的所有自动装配候选者以满足依赖关系。 如果预期的键类型是 String，您可以自动装配强类型 Map 实例。 自动装配的 Map 实例的值由所有与预期类型匹配的 bean 实例组成，并且 Map 实例的键包含相应的 bean 名称。

#### 自动装配的局限性和缺点

* 属性和构造函数参数设置中的显式依赖项始终自动装配。 您不能自动装配简单的属性，例如primitives、字符串和类（以及此类简单属性的数组）。 此限制是有意设计的。
* 自动装配不如显式装配精确。 虽然，如前面的表中所述，Spring 小心避免在可能产生意外结果的歧义的情况下进行猜测。 不再明确记录 Spring 管理的对象之间的关系。
* 可能无法从 Spring 容器生成文档的工具中使用装配信息。
* 容器内的多个 bean 定义可能与要自动装配的 setter 方法或构造函数参数指定的类型相匹配。 对于数组、集合或 Map 实例，这不一定是问题。 但是，对于期望单个值的依赖项，这种歧义不会被任意解决。 如果没有唯一的 bean 定义可用，则抛出异常。您有多种选择：
  * 放弃自动装配以支持显式装配。
  * 通过将 bean 定义的 autowire-candidate 属性设置为 false 来避免自动装配 bean 定义。
  * 通过将其 <bean/> 元素的primary属性设置为 true，将单个 bean 定义指定为主要候选者。
  * 使用基于注解的配置实现更细粒度的控制。

#### 从自动装配中排除 Bean

在每个 bean 的基础上，您可以从自动装配中排除一个 bean。 在 Spring 的 XML 格式中，将 <bean/> 元素的 autowire-candidate 属性设置为 false。 容器使该特定 bean 定义对自动装配基础设施不可用（包括注释样式配置，例如 @Autowired）。

> autowire-candidate 属性旨在仅影响基于类型的自动装配。 它不会影响按名称的显式引用，即使指定的 bean 未标记为自动装配候选者，也会解析。 因此，如果名称匹配，按名称自动装配仍然会注入一个 bean。

您还可以根据对 bean 名称的模式匹配来限制自动装配候选者。 顶级 <beans/> 元素在其 default-autowire-candidates 属性中接受一个或多个模式。 例如，要将自动装配候选状态限制为名称以 Repository 结尾的任何 bean，请提供值 *Repository。 要提供多个模式，请在逗号分隔的列表中定义它们。 bean 定义的 autowire-candidate 属性的显式 true 或 false 值始终优先。 对于此类 bean，模式匹配规则不适用。

### 方法注入

在大多数应用场景中，容器中的大多数bean都是单例的。 当单例 bean 需要与另一个单例 bean 协作或非单例 bean 需要与另一个非单例 bean 协作时，您通常通过将一个 bean 定义为另一个 bean 的属性来处理依赖关系。 当 bean 生命周期不同时就会出现问题。 假设单例 bean A 需要使用非单例（原型）bean B，可能在 A 上的每次方法调用上。容器只创建单例 bean A 一次，因此只有一次设置属性的机会。 容器无法在每次需要时为 bean A 提供 bean B 的新实例。

一个解决方案是放弃一些控制反转。 您可以通过实现 ApplicationContextAware 接口使 bean A 了解容器，并在每次 bean A 需要时通过对容器进行 getBean("B") 调用来请求（通常是新的）bean B 实例。 以下示例显示了这种方法：

```java
// a class that uses a stateful Command-style class to perform some processing
package fiona.apple;

// Spring-API imports
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class CommandManager implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public Object process(Map commandState) {
        // grab a new instance of the appropriate Command
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    protected Command createCommand() {
        // notice the Spring API dependency!
        return this.applicationContext.getBean("command", Command.class);
    }

    public void setApplicationContext(
            ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

前面是不可取的，因为业务代码知道并耦合到 Spring Framework。 方法注入是 Spring IoC 容器的一个有点高级的特性，可以让你干净地处理这个用例。

#### 指定方法注入

查找方法注入是容器覆盖容器管理 bean 上的方法并返回容器中另一个命名 bean 的查找结果的能力。 查找通常涉及原型 bean，如上一节中描述的场景。 Spring Framework 通过使用来自 CGLIB 库的字节码生成来动态生成覆盖该方法的子类来实现此方法注入。

> * 要使这种动态子类化工作，Spring bean 容器子类化的类不能是 final，要覆盖的方法也不能是 final。
> * 对具有抽象方法的类进行单元测试需要您自己对类进行子类化并提供抽象方法的存根实现。
> * 组件扫描也需要具体的方法，这需要具体的类来获取。
> * 另一个关键限制是查找方法不适用于工厂方法，尤其不适用于配置类中的 @Bean 方法，因为在这种情况下，容器不负责创建实例，因此无法创建运行时生成的 动态子类。

对于前面代码片段中的 CommandManager 类，Spring 容器动态覆盖了 createCommand() 方法的实现。 CommandManager 类没有任何 Spring 依赖项，如重新设计的示例所示：

```java
package fiona.apple;

// no more Spring imports!

public abstract class CommandManager {

    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}
```

在包含要注入的方法（在本例中为 CommandManager）的客户端类中，要注入的方法需要以下形式的签名：

```xml
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
```

如果该方法是抽象的，则动态生成的子类将实现该方法。 否则，动态生成的子类会覆盖原始类中定义的具体方法。 考虑以下示例：

```xml
<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
    <!-- inject dependencies here as required -->
</bean>

<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager">
    <lookup-method name="createCommand" bean="myCommand"/>
</bean>
```

标识为 commandManager 的 bean 在需要 myCommand bean 的新实例时调用它自己的 createCommand() 方法。 如果实际上需要，您必须小心地将 myCommand bean 部署为原型。 如果是单例，则每次都返回相同的 myCommand bean 实例。

或者，在基于注解的组件模型中，您可以通过@Lookup 注解声明一个查找方法，如下例所示：

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup("myCommand")
    protected abstract Command createCommand();
}
```

或者，更惯用的是，您可以依靠目标 bean 根据查找方法的声明返回类型进行解析：

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup
    protected abstract Command createCommand();
}
```

请注意，您通常应该使用具体的存根实现声明此类带注释的查找方法，以便它们与 Spring 的组件扫描规则兼容，其中默认情况下会忽略抽象类。 此限制不适用于显式注册或显式导入的 bean 类。

> 访问不同范围的目标 bean 的另一种方法是 ObjectFactory/Provider 注入点。 将 Scoped Beans 视为依赖项。
>
> 您可能还会发现 ServiceLocatorFactoryBean（在 org.springframework.beans.factory.config 包中）很有用。

#### 任意方法替换

与查找方法注入相比，一种不太有用的方法注入形式是能够用另一种方法实现替换托管 bean 中的任意方法。 您可以安全地跳过本节的其余部分，直到您真正需要此功能。

使用基于 XML 的配置元数据，您可以使用replaced-method 元素将现有的方法实现替换为另一个，用于部署的bean。 考虑下面的类，它有一个我们想要覆盖的名为 computeValue 的方法：

```java
public class MyValueCalculator {

    public String computeValue(String input) {
        // some real code...
    }

    // some other methods...
}
```

实现 org.springframework.beans.factory.support.MethodReplacer 接口的类提供了新的方法定义，如以下示例所示：

```java
/**
 * meant to be used to override the existing computeValue(String)
 * implementation in MyValueCalculator
 */
public class ReplacementComputeValue implements MethodReplacer {

    public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
        // get the input value, work with it, and return a computed result
        String input = (String) args[0];
        ...
        return ...;
    }
}
```

用于部署原始类并指定方法覆盖的 bean 定义类似于以下示例：

```xml
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
    <!-- arbitrary method replacement -->
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>String</arg-type>
    </replaced-method>
</bean>

<bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>
```

您可以在 <replaced-method/> 元素中使用一个或多个 <arg-type/> 元素来指示被覆盖的方法的方法签名。 仅当方法重载并且类中存在多个变体时，才需要参数的签名。 为方便起见，参数的类型字符串可以是完全限定类型名称的子字符串。 例如，以下所有匹配 java.lang.String：

```sh
java.lang.String
String
Str
```

因为参数的数量通常足以区分每个可能的选择，所以这个快捷方式可以节省大量输入，让您只输入与参数类型匹配的最短字符串。

## Bean scopes

Spring Framework 支持六个scope，其中四个仅在您使用 web-aware ApplicationContext 时可用。 您还可以创建自定义范围。

* singleton：（默认）将单个 bean 定义范围限定为每个 Spring IoC 容器的单个对象实例。
* prototype：将单个 bean 定义范围限定为任意数量的对象实例。
* request：将单个 bean 定义范围限定为单个 HTTP 请求的生命周期。 也就是说，每个 HTTP 请求都有自己的 bean 实例，该 bean 实例是在单个 bean 定义的后面创建的。 仅在 web-aware Spring ApplicationContext 的上下文中有效。
* session：将单个 bean 定义范围限定为 HTTP 会话的生命周期。 仅在 web-aware Spring ApplicationContext 的上下文中有效
* application：将单个 bean 定义范围限定为 ServletContext 的生命周期。 仅在 web-aware Spring ApplicationContext 的上下文中有效
* websocket：将单个 bean 定义范围限定为 WebSocket 的生命周期。 仅在 web-aware Spring ApplicationContext 的上下文中有效。

> 从 Spring 3.0 开始，线程作用域可用，但默认情况下未注册。 有关更多信息，请参阅 [SimpleThreadScope ](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/context/support/SimpleThreadScope.html)的文档。

### singleton

Spring IoC 容器会创建该 bean的唯一实例。 该单个实例存储在此类单例 bean 的缓存中，并且对该命名 bean 的所有后续请求和引用都返回缓存对象。 下图显示了单例范围的工作原理：

![singleton](spring-core/singleton.png)

Spring 的单例 bean 概念不同于GoF模式书中定义的单例模式。 GoF 单例对对象的范围进行了硬编码，以便每个 ClassLoader 只创建特定类的一个实例。 Spring 单例的范围最好描述为每个容器和每个 bean。 这意味着，如果您在单个 Spring 容器中为特定类定义一个 bean，则 Spring 容器会创建一个且仅一个实例。 单例作用域是 Spring 中的默认作用域。 要将 bean 定义为 XML 中的单例，您可以定义一个 bean，如下例所示：

```xml
<bean id="accountService" class="com.something.DefaultAccountService"/>

<!-- the following is equivalent, though redundant (singleton scope is the default) -->
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```

### prototype 

原型范围导致每次对特定 bean 发出请求时都会创建一个新 bean 实例。 也就是说，bean 被注入到另一个 bean 中，或者您通过容器上的 getBean() 方法调用来请求它。 通常，您应该对所有有状态 bean 使用原型作用域，对无状态 bean 使用单例作用域。

![prototype](spring-core/prototype.png)

```xml
<bean id="accountService" class="com.something.DefaultAccountService" scope="prototype"/>
```

与其他作用域相比，Spring 不管理原型 bean 的完整生命周期。 容器实例化、配置和以其他方式组装原型对象并将其交给客户端，不会缓存该实例。 因此，尽管在所有对象上调用初始化生命周期回调方法（不管范围如何），但在原型的情况下，不会调用配置的销毁生命周期回调。 客户端代码必须清理原型范围内的对象并释放原型 bean 持有的昂贵资源。 要让 Spring 容器释放原型作用域 bean 持有的资源，请尝试使用自定义 bean 后处理器，它保存对需要清理的 bean 的引用。

### 具有 Prototype-bean 依赖关系的 Singleton Bean

当您使用具有对原型 bean 的依赖的单例作用域 bean 时，请注意在实例化时解析依赖关系。 因此，如果您将原型范围的 bean 依赖注入到单例范围的 bean 中，则会实例化一个新的原型 bean，然后将依赖项注入到单例 bean 中。 原型实例是唯一提供给单例作用域 bean 的实例。

但是，假设您希望单例范围的 bean 在运行时重复获取原型范围的 bean 的新实例。 您不能将原型范围的 bean 依赖注入到您的单例 bean 中，因为该注入仅发生一次，当 Spring 容器实例化单例 bean 并解析并注入其依赖项时。 如果在运行时多次需要原型 bean 的新实例，请参阅方法注入。

### Request, Session, Application, and WebSocket Scopes

请求、会话、应用程序和 websocket 范围仅在您使用 Web 感知 Spring ApplicationContext 实现（例如 XmlWebApplicationContext）时才可用。 如果您将这些作用域与常规 Spring IoC 容器（例如 ClassPathXmlApplicationContext）一起使用，则会抛出 IllegalStateException 抱怨未知的 bean 作用域。

#### 初始化web配置

为了在请求、会话、应用程序和 websocket 级别（网络范围的 bean）支持 bean 的范围，在定义 bean 之前需要一些小的初始配置。 （标准范围（单例和原型）不需要此初始设置。）

您如何完成此初始设置取决于您的特定 Servlet 环境。

如果您在 Spring Web MVC 中访问作用域 bean，实际上是在 Spring DispatcherServlet 处理的请求中，则不需要特殊设置。 DispatcherServlet 已经公开了所有相关的状态。

如果您使用 Servlet 2.5 Web 容器，并且请求在 Spring 的 DispatcherServlet 之外处理（例如，使用 JSF 或 Struts 时），则需要注册 org.springframework.web.context.request.RequestContextListener ServletRequestListener。 对于 Servlet 3.0+，这可以通过使用 WebApplicationInitializer 接口以编程方式完成。 或者，或者对于较旧的容器，请将以下声明添加到您的 Web 应用程序的 web.xml 文件中：

```xml
<web-app>
    ...
    <listener>
        <listener-class>
            org.springframework.web.context.request.RequestContextListener
        </listener-class>
    </listener>
    ...
</web-app>
```

或者，如果您的侦听器设置存在问题，请考虑使用 Spring 的 RequestContextFilter。 过滤器映射取决于周围的 Web 应用程序配置，因此您必须适当地更改它。 以下清单显示了 Web 应用程序的过滤器部分：

```xml
<web-app>
    ...
    <filter>
        <filter-name>requestContextFilter</filter-name>
        <filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>requestContextFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ...
</web-app>
```

DispatcherServlet、RequestContextListener 和 RequestContextFilter 都做完全相同的事情，即将 HTTP 请求对象绑定到为该请求提供服务的线程。 这使得请求和会话范围内的 bean 在调用链的更下游可用。

#### Request scope

```xml
<bean id="loginAction" class="com.something.LoginAction" scope="request"/>
```

Spring 容器通过为每个 HTTP 请求使用 loginAction bean 定义来创建 LoginAction bean 的新实例。 也就是说， loginAction bean 的范围在 HTTP 请求级别。 您可以根据需要更改所创建实例的内部状态，因为从同一 loginAction bean 定义创建的其他实例不会看到这些状态更改。 它们是针对个人要求的。 当请求完成处理时，该请求范围内的 bean 将被丢弃。

当使用注解驱动的组件或 Java 配置时，@RequestScope 注解可用于将组件分配给请求范围。 以下示例显示了如何执行此操作：

```java
@RequestScope
@Component
public class LoginAction {
    // ...
}
```

#### Session Scope

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>
```

Spring 容器通过在单个 HTTP 会话的生命周期中使用 userPreferences bean 定义创建 UserPreferences bean 的新实例。 换句话说，userPreferences bean 有效地限定在 HTTP 会话级别。 与请求范围的 bean 一样，您可以根据需要更改所创建实例的内部状态，知道其他也在使用从相同 userPreferences bean 定义创建的实例的 HTTP Session 实例不会看到这些状态更改 ，因为它们特定于单个 HTTP 会话。 当 HTTP 会话最终被丢弃时，作用域为该特定 HTTP 会话的 bean 也将被丢弃。

使用注解驱动的组件或 Java 配置时，您可以使用 @SessionScope 注解将组件分配给会话范围。

```java
@SessionScope
@Component
public class UserPreferences {
    // ...
}
```

#### Application Scope

```xml
<bean id="appPreferences" class="com.something.AppPreferences" scope="application"/>
```

Spring 容器通过为整个 Web 应用程序使用 appPreferences bean 定义一次来创建 AppPreferences bean 的新实例。 也就是说，appPreferences bean 的作用域是 ServletContext 级别并存储为常规 ServletContext 属性。 这有点类似于 Spring 单例 bean，但在两个重要方面有所不同：它是每个 ServletContext 的单例，而不是每个 Spring ApplicationContext（在任何给定的 Web 应用程序中可能有多个），并且它实际上是公开的，因此可见为 一个 ServletContext 属性。

当使用注解驱动的组件或 Java 配置时，您可以使用 @ApplicationScope 注解将组件分配给应用程序范围。 以下示例显示了如何执行此操作：

```java
@ApplicationScope
@Component
public class AppPreferences {
    // ...
}
```

#### 作用域 Bean 作为依赖项

Spring IoC 容器不仅管理对象（bean）的实例化，还管理协作者（或依赖项）的连接。 如果您想将（例如）一个 HTTP 请求范围的 bean 注入到另一个生命周期更长的 bean 中，您可以选择注入一个 AOP 代理来代替该范围的 bean。 也就是说，您需要注入一个代理对象，该对象公开与作用域对象相同的公共接口，但也可以从相关作用域（例如 HTTP 请求）中检索真实目标对象，并将方法调用委托给真实对象。

下例中的配置只有一行，但了解其背后的“为什么”以及“如何”很重要：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- an HTTP Session-scoped bean exposed as a proxy -->
    <bean id="userPreferences" class="com.something.UserPreferences" scope="session">
        <!-- instructs the container to proxy the surrounding bean -->
        <aop:scoped-proxy/> 
    </bean>

    <!-- a singleton-scoped bean injected with a proxy to the above bean -->
    <bean id="userService" class="com.something.SimpleUserService">
        <!-- a reference to the proxied userPreferences bean -->
        <property name="userPreferences" ref="userPreferences"/>
    </bean>
</beans>
```

### 自定义scope

bean 作用域机制是可扩展的。 您可以定义自己的作用域，甚至重新定义现有作用域，尽管后者被认为是不好的做法，并且您不能覆盖内置的单例和原型作用域。

#### 创建作用域

要将自定义范围集成到 Spring 容器中，您需要实现 org.springframework.beans.factory.config.Scope 接口，这在本节中进行了描述。 有关如何实现您自己的范围的想法，请参阅 Spring Framework 本身提供的 Scope 实现和 Scope javadoc，其中更详细地解释了您需要实现的方法。

Scope 接口有四种方法可以从作用域中获取对象，从作用域中移除它们，以及让它们被销毁。

例如，会话作用域实现返回会话作用域 bean（如果它不存在，则该方法在将它绑定到会话以供将来参考后返回该 bean 的一个新实例）。 以下方法从底层范围返回对象：

```java
Object get(String name, ObjectFactory<?> objectFactory)
```

例如，会话范围实现从底层会话中删除会话范围的 bean。 应该返回该对象，但如果没有找到具有指定名称的对象，您可以返回 null。 以下方法从基础范围中删除对象：

```java
Object remove(String name)
```

以下方法注册了一个回调，当它被销毁或范围中的指定对象被销毁时，该范围应该调用该回调：

```java
void registerDestructionCallback(String name, Runnable destructionCallback)
```

有关销毁回调的更多信息，请参阅 javadoc 或 Spring 范围实现。

以下方法获取基础范围的对话标识符：

```java
String getConversationId()
```

这个标识符对于每个范围都是不同的。 对于会话范围的实现，此标识符可以是会话标识符。

#### 使用自定义scope

在编写并测试一个或多个自定义 Scope 实现后，您需要让 Spring 容器知道您的新范围。 以下方法是向 Spring 容器注册新 Scope 的中心方法

```java
void registerScope(String scopeName, Scope scope);
```

此方法在 ConfigurableBeanFactory 接口上声明，该接口可通过 Spring 附带的大多数具体 ApplicationContext 实现的 BeanFactory 属性获得。

registerScope(..) 方法的第一个参数是与作用域关联的唯一名称。 Spring 容器本身中此类名称的示例是 singleton 和 prototype。 registerScope(..) 方法的第二个参数是您希望注册和使用的自定义 Scope 实现的实际实例。

假设您编写了自定义 Scope 实现，然后按照下一个示例所示进行注册。

```java
Scope threadScope = new SimpleThreadScope();
beanFactory.registerScope("thread", threadScope);
```

然后，您可以创建符合自定义 Scope 的范围规则的 bean 定义，如下所示：

```xml
<bean id="..." class="..." scope="thread">
```

使用自定义 Scope 实现，您不仅限于以编程方式注册范围。 您还可以使用 CustomScopeConfigurer 类以声明方式进行 Scope 注册，如以下示例所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
        <property name="scopes">
            <map>
                <entry key="thread">
                    <bean class="org.springframework.context.support.SimpleThreadScope"/>
                </entry>
            </map>
        </property>
    </bean>

    <bean id="thing2" class="x.y.Thing2" scope="thread">
        <property name="name" value="Rick"/>
        <aop:scoped-proxy/>
    </bean>

    <bean id="thing1" class="x.y.Thing1">
        <property name="thing2" ref="thing2"/>
    </bean>

</beans>
```

## bean自定义

Spring Framework 提供了许多可用于自定义 bean 性质的接口。 本节将它们分组如下：

* 生命周期回调
* [`ApplicationContextAware` and `BeanNameAware`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware)
* 其他Aware

### 生命周期回调

要与容器对 bean 生命周期的管理进行交互，您可以实现 Spring InitializingBean 和 DisposableBean 接口。 容器为前者调用 afterPropertiesSet() 并为后者调用 destroy() 以让 bean 在初始化和销毁 bean 时执行某些操作。

在内部，Spring 框架使用 `BeanPostProcessor` 实现来处理它可以找到的任何回调接口并调用适当的方法。 如果您需要自定义功能或 Spring 默认不提供的其他生命周期行为，您可以自己实现 BeanPostProcessor。 有关详细信息，请参阅容器扩展点。

除了初始化和销毁回调之外，Spring 管理的对象还可以实现 Lifecycle 接口，以便这些对象可以参与启动和关闭过程，由容器自身的生命周期驱动。

本节介绍生命周期回调接口。

#### 初始回调

org.springframework.beans.factory.InitializingBean 接口让容器在 bean 上设置所有必要的属性后执行初始化工作。 InitializingBean 接口指定了一个方法：

```java
void afterPropertiesSet() throws Exception;
```

我们建议您不要使用 InitializingBean 接口，因为它不必要地将代码耦合到 Spring。 或者，我们建议使用 @PostConstruct 注释或指定 POJO 初始化方法。 对于基于 XML 的配置元数据，您可以使用 init-method 属性来指定具有 void 无参数签名的方法的名称。 通过 Java 配置，您可以使用@Bean 的 initMethod 属性。 考虑以下示例：

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
```

```java
public class ExampleBean {

    public void init() {
        // do some initialization work
    }
}
```

前面的示例与下面的示例（由两个清单组成）几乎具有完全相同的效果：

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>
```

```java
public class AnotherExampleBean implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        // do some initialization work
    }
}
```

但是，前面两个示例中的第一个没有将代码耦合到 Spring。

#### 销毁回调

实现 org.springframework.beans.factory.DisposableBean 接口可以让 bean 在其被销毁时获得回调。 DisposableBean 接口指定了一个方法：

```java
void destroy() throws Exception;
```

我们建议您不要使用 DisposableBean 回调接口，因为它不必要地将代码耦合到 Spring。 或者，我们建议使用 @PreDestroy 注释或指定 bean 定义支持的泛型方法。 使用基于 XML 的配置元数据，您可以使用 <bean/> 上的 destroy-method 属性。 通过 Java 配置，您可以使用 @Bean 的 destroyMethod 属性。 考虑以下定义：

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" destroy-method="cleanup"/>
```

```java
public class ExampleBean {

    public void cleanup() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

前面的定义与下面的定义几乎完全相同：

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>
```

```java
public class AnotherExampleBean implements DisposableBean {

    @Override
    public void destroy() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

但是，前面两个定义中的第一个没有将代码耦合到 Spring。

#### 默认的初始化和销毁方法

当您编写不使用特定于 Spring 的 InitializingBean 和 DisposableBean 回调接口的初始化和销毁方法回调时，您通常编写具有诸如 init()、initialize()、dispose() 等名称的方法。 理想情况下，此类生命周期回调方法的名称在整个项目中是标准化的，以便所有开发人员使用相同的方法名称并确保一致性。

您可以将 Spring 容器配置为“查找”命名初始化并销毁每个 bean 上的回调方法名称。 这意味着，作为应用程序开发人员，您可以编写应用程序类并使用称为 init() 的初始化回调，而无需为每个 bean 定义配置 init-method="init" 属性。 Spring IoC 容器在创建 bean 时调用该方法（并根据前面描述的标准生命周期回调协定）。 此功能还为初始化和销毁方法回调强制执行一致的命名约定。

假设您的初始化回调方法名为 init()，而您的销毁回调方法名为 destroy()。 然后，您的类类似于以下示例中的类：

```java
public class DefaultBlogService implements BlogService {

    private BlogDao blogDao;

    public void setBlogDao(BlogDao blogDao) {
        this.blogDao = blogDao;
    }

    // this is (unsurprisingly) the initialization callback method
    public void init() {
        if (this.blogDao == null) {
            throw new IllegalStateException("The [blogDao] property must be set.");
        }
    }
}
```

然后，您可以在类似于以下内容的 bean 中使用该类：

```java
<beans default-init-method="init">

    <bean id="blogService" class="com.something.DefaultBlogService">
        <property name="blogDao" ref="blogDao" />
    </bean>

</beans>
```

顶级 <beans/> 元素属性上的 default-init-method 属性的存在导致 Spring IoC 容器将 bean 类上称为 init 的方法识别为初始化方法回调。 在创建和组装 bean 时，如果 bean 类具有这样的方法，则会在适当的时间调用它。

您可以通过使用顶级 <beans/> 元素上的 default-destroy-method 属性类似地（即在 XML 中）配置销毁方法回调。

如果现有 bean 类已经具有命名与约定不同的回调方法，您可以通过使用 <bean/ 的 init-method 和 destroy-method 属性指定（在 XML 中，即）方法名称来覆盖默认值 > 本身。

Spring 容器保证在为 bean 提供所有依赖项后立即调用配置的初始化回调。 因此，在原始 bean 引用上调用初始化回调，这意味着 AOP 拦截器等尚未应用于 bean。 首先完全创建目标 bean，然后应用带有拦截器链的 AOP 代理（例如）。 如果目标 bean 和代理分别定义，您的代码甚至可以与原始目标 bean 交互，绕过代理。 因此，将拦截器应用于 init 方法是不一致的，因为这样做会将目标 bean 的生命周期耦合到其代理或拦截器，并在您的代码直接与原始目标 bean 交互时留下奇怪的语义。

#### 多个生命周期执行顺序

从 Spring 2.5 开始，您可以通过三个选项来控制 bean 生命周期行为：

* InitializingBean 和 DisposableBean 回调接口

* 自定义 init() 和 destroy() 方法

* @PostConstruct 和 @PreDestroy 注释。 您可以组合这些机制来控制给定的 bean。

如果为一个 bean 配置了多个生命周期机制，并且每个机制都配置了不同的方法名称，那么每个配置的方法将按照本注释后面列出的顺序运行。 但是，如果配置了相同的方法名称 — 例如，init() 用于初始化方法 — 用于多个生命周期机制，则该方法将运行一次，如前一节所述。

为同一个 bean 配置的多个生命周期机制，具有不同的初始化方法，调用如下：

* 用@PostConstruct 注释的方法
* afterPropertiesSet() 由 InitializingBean 回调接口定义
* 自定义配置的 init() 方法

销毁方法以相同的顺序调用：

* 用@PreDestroy 注释的方法
* destroy() 由 DisposableBean 回调接口定义
* 自定义配置的 destroy() 方法

#### Startup 和 Shutdown回调

Lifecycle 接口为任何具有自己生命周期要求的对象定义了基本方法（例如启动和停止某些后台进程）：

```java
public interface Lifecycle {

    void start();

    void stop();

    boolean isRunning();
}
```

任何 Spring 管理的对象都可以实现 Lifecycle 接口。 然后，当 ApplicationContext 本身接收到启动和停止信号时（例如，对于运行时的停止/重启场景），它会将这些调用级联到该上下文中定义的所有 Lifecycle 实现。 它通过委托给 LifecycleProcessor 来做到这一点，如下面的清单所示：

```java
public interface LifecycleProcessor extends Lifecycle {

    void onRefresh();

    void onClose();
}
```

请注意 LifecycleProcessor 本身是 Lifecycle 接口的扩展。 它还添加了另外两种方法来对正在刷新和关闭的上下文做出反应。

> 请注意，常规 org.springframework.context.Lifecycle 接口是用于显式启动和停止通知的简单契约，并不意味着在上下文刷新时自动启动。 为了对特定 bean 的自动启动（包括启动阶段）进行细粒度控制，请考虑实现 org.springframework.context.SmartLifecycle 代替。
>
> 另外，请注意，不能保证在销毁之前发出停止通知。 在常规关闭时，所有 Lifecycle bean 在传播一般销毁回调之前首先收到停止通知。 但是，在上下文生命周期内的热刷新或停止刷新尝试时，只会调用 destroy 方法。

启动和关闭调用的顺序可能很重要。 如果任何两个对象之间存在“依赖”关系，则依赖方在其依赖之后开始，并在其依赖之前停止。 然而，有时，直接依赖是未知的。 您可能只知道某种类型的对象应该在另一种类型的对象之前开始。 在这些情况下，SmartLifecycle 接口定义了另一个选项，即在其超级接口 Phased 上定义的 getPhase() 方法。 以下清单显示了 Phased 接口的定义：

```java
public interface Phased {

    int getPhase();
}
```

以下清单显示了 SmartLifecycle 接口的定义：

```java
public interface SmartLifecycle extends Lifecycle, Phased {

    boolean isAutoStartup();

    void stop(Runnable callback);
}
```

启动时，Phased最低的对象首先启动。 停止时，遵循相反的顺序。 因此，一个实现 SmartLifecycle 并且其 getPhase() 方法返回 Integer.MIN_VALUE 的对象将是最先启动和最后一个停止的对象。 在频谱的另一端，Integer.MAX_VALUE 的阶段值表示该对象应该最后启动并首先停止（可能是因为它取决于正在运行的其他进程）。在考虑Phased值时，重要的是要知道任何未实现 SmartLifecycle 的“正常”生命周期对象的默认Phased是 0。因此，任何负Phased值表示对象应该在这些标准组件之前开始（并停止） 在他们之后）。 对于任何正Phased值，反之亦然。

SmartLifecycle 定义的 stop 方法接受回调。 任何实现都必须在该实现的关闭过程完成后调用该回调的 run() 方法。 这会在必要时启用异步关闭，因为 LifecycleProcessor 接口的默认实现 DefaultLifecycleProcessor 会等待每个阶段中的对象组调用该回调的超时值。 默认的每阶段超时为 30 秒。 您可以通过在上下文中定义一个名为生命周期处理器的 bean 来覆盖默认的生命周期处理器实例。 如果您只想修改超时，定义以下内容就足够了：

```xml
<bean id="lifecycleProcessor" class="org.springframework.context.support.DefaultLifecycleProcessor">
    <!-- timeout value in milliseconds -->
    <property name="timeoutPerShutdownPhase" value="10000"/>
</bean>
```

如前所述，LifecycleProcessor 接口还定义了刷新和关闭上下文的回调方法。 后者驱动关闭过程，就好像 stop() 已被显式调用一样，但它会在上下文关闭时发生。 另一方面，“刷新”回调启用 SmartLifecycle bean 的另一个功能。 当上下文刷新时（在所有对象都被实例化和初始化之后），该回调被调用。 此时，默认生命周期处理器会检查每个 SmartLifecycle 对象的 isAutoStartup() 方法返回的布尔值。 如果为 true，则该对象在该点启动，而不是等待上下文或其自己的 start() 方法的显式调用（与上下文刷新不同，对于标准上下文实现，上下文启动不会自动发生）。 相位值和任何“依赖”关系确定启动顺序，如前所述。

#### 在非 Web 应用程序中优雅地关闭 Spring IoC 容器

如果您在非 Web 应用程序环境（例如，富客户端桌面环境）中使用 Spring 的 IoC 容器，请向 JVM 注册一个关闭钩子。 这样做可确保正常关闭并在单例 bean 上调用相关的 destroy 方法，以便释放所有资源。 您仍然必须正确配置和实现这些销毁回调。

要注册关闭挂钩，请调用在 ConfigurableApplicationContext 接口上声明的 registerShutdownHook() 方法，如以下示例所示：

```java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

        // add a shutdown hook for the above context...
        ctx.registerShutdownHook();

        // app runs here...

        // main method exits, hook is called prior to the app shutting down...
    }
}
```

### `ApplicationContextAware` 和`BeanNameAware`

当 ApplicationContext 创建一个实现 org.springframework.context.ApplicationContextAware 接口的对象实例时，该实例提供了对该 ApplicationContext 的引用。 以下清单显示了 ApplicationContextAware 接口的定义：

```java
public interface ApplicationContextAware {

    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

因此，bean 可以通过 ApplicationContext 接口或通过将引用转换为该接口的已知子类（例如 ConfigurableApplicationContext，它公开附加功能），以编程方式操作创建它们的 ApplicationContext。 一种用途是其他 bean 的程序化检索。 有时，此功能很有用。 但是，一般而言，您应该避免使用它，因为它将代码耦合到 Spring 并且不遵循控制反转样式，其中将协作者作为属性提供给 bean。 ApplicationContext 的其他方法提供对文件资源的访问、发布应用程序事件和访问 MessageSource。 这些附加功能在 ApplicationContext 的附加功能中进行了描述。

自动装配是获取对 ApplicationContext 的引用的另一种选择。 传统的构造函数和 byType 自动装配模式（如 Autowiring Collaborators 中所述）可以分别为构造函数参数或 setter 方法参数提供 ApplicationContext 类型的依赖项。 为了获得更大的灵活性，包括自动装配字段和多参数方法的能力，请使用基于注释的自动装配功能。 如果这样做，如果所讨论的字段、构造函数或方法带有 @Autowired 注释，则 ApplicationContext 将自动装配到需要 ApplicationContext 类型的字段、构造函数参数或方法参数中。 有关更多信息，请参阅使用 @Autowired。

当 ApplicationContext 创建一个实现 org.springframework.beans.factory.BeanNameAware 接口的类时，该类被提供了对在其关联对象定义中定义的名称的引用。 以下清单显示了 BeanNameAware 接口的定义：

```java
public interface BeanNameAware {

    void setBeanName(String name) throws BeansException;
}
```

在填充普通 bean 属性之后但在初始化回调（例如 InitializingBean.afterPropertiesSet() 或自定义 init-method ）之前调用回调。

### 其他`Aware`接口

除了 ApplicationContextAware 和 BeanNameAware（之前讨论过），Spring 还提供了广泛的 Aware 回调接口，让 bean 向容器表明它们需要某种基础设施依赖项。 作为一般规则，名称表示依赖项类型。 下表总结了最重要的 Aware 接口：

| Name                             | Injected Dependency                                          | Explained in…                                                |
| :------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `ApplicationContextAware`        | Declaring `ApplicationContext`.                              | [`ApplicationContextAware` and `BeanNameAware`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware) |
| `ApplicationEventPublisherAware` | Event publisher of the enclosing `ApplicationContext`.       | [Additional Capabilities of the `ApplicationContext`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction) |
| `BeanClassLoaderAware`           | Class loader used to load the bean classes.                  | [Instantiating Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-class) |
| `BeanFactoryAware`               | Declaring `BeanFactory`.                                     | [The `BeanFactory`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beanfactory) |
| `BeanNameAware`                  | Name of the declaring bean.                                  | [`ApplicationContextAware` and `BeanNameAware`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware) |
| `LoadTimeWeaverAware`            | Defined weaver for processing class definition at load time. | [Load-time Weaving with AspectJ in the Spring Framework](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-aj-ltw) |
| `MessageSourceAware`             | Configured strategy for resolving messages (with support for parametrization and internationalization). | [Additional Capabilities of the `ApplicationContext`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction) |
| `NotificationPublisherAware`     | Spring JMX notification publisher.                           | [Notifications](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#jmx-notifications) |
| `ResourceLoaderAware`            | Configured loader for low-level access to resources.         | [Resources](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#resources) |
| `ServletConfigAware`             | Current `ServletConfig` the container runs in. Valid only in a web-aware Spring `ApplicationContext`. | [Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc) |
| `ServletContextAware`            | Current `ServletContext` the container runs in. Valid only in a web-aware Spring `ApplicationContext`. | [Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc) |

再次注意，使用这些接口会将您的代码与 Spring API 联系起来，并且不遵循控制反转风格。 因此，我们建议将它们用于需要以编程方式访问容器的基础设施 bean。



