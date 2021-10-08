# spring 容器

## Bean 定义继承

bean 定义可以包含很多配置信息，包括构造函数参数、属性值和特定于容器的信息，例如初始化方法、静态工厂方法名称等。 子 bean 定义从父定义继承配置数据。 子定义可以根据需要覆盖某些值或添加其他值。 使用父和子 bean 定义可以节省大量输入。 实际上，这是一种模板形式。

如果您以编程方式使用 ApplicationContext 接口，则子 bean 定义由 ChildBeanDefinition 类表示。 大多数用户不会在这个级别上与他们合作。 相反，它们在类（例如 ClassPathXmlApplicationContext）中以声明方式配置 bean 定义。 当您使用基于 XML 的配置元数据时，您可以通过使用 parent 属性来指示子 bean 定义，将父 bean 指定为该属性的值。 以下示例显示了如何执行此操作：

```xml
<bean id="inheritedTestBean" abstract="true"
        class="org.springframework.beans.TestBean">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithDifferentClass"
        class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBean" init-method="initialize">  
    <property name="name" value="override"/>
    <!-- the age property value of 1 will be inherited from parent -->
</bean>
```

如果没有指定，子 bean 定义使用父定义中的 bean 类，但也可以覆盖它。 在后一种情况下，子 bean 类必须与父级兼容（即它必须接受父级的属性值）。

子 bean 定义从父 bean 继承范围、构造函数参数值、属性值和方法覆盖，并可以选择添加新值。 您指定的任何范围、初始化方法、销毁方法或静态工厂方法设置都会覆盖相应的父设置。

其余设置始终取自子定义：依赖、自动装配模式、依赖项检查、单例和惰性初始化。

前面的示例使用抽象属性将父 bean 定义显式标记为抽象。 如果父定义未指定类，则需要将父 bean 定义显式标记为抽象，如以下示例所示：

```xml
<bean id="inheritedTestBeanWithoutClass" abstract="true">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithClass" class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBeanWithoutClass" init-method="initialize">
    <property name="name" value="override"/>
    <!-- age will inherit the value of 1 from the parent bean definition-->
</bean>
```

父 bean 不能单独实例化，因为它是不完整的，并且它也被显式标记为抽象。 当定义是抽象的时，它只能用作纯模板 bean 定义，作为子定义的父定义。 通过将其作为另一个 bean 的 ref 属性引用或使用父 bean ID 执行显式 getBean() 调用，尝试单独使用此类抽象父 bean 会返回错误。 类似地，容器的内部 preInstantiateSingletons() 方法忽略定义为抽象的 bean 定义。

## 容器扩展点

通常，应用程序开发人员不需要子类化 ApplicationContext 实现类。 相反，可以通过插入特殊集成接口的实现来扩展 Spring IoC 容器。 接下来的几节将描述这些集成接口。

### 使用`BeanPostProcessor`自定义bean

BeanPostProcessor 接口定义了回调方法，您可以实现这些方法来提供您自己的（或覆盖容器的默认）实例化逻辑、依赖解析逻辑等。 如果要在 Spring 容器完成对 bean 的实例化、配置和初始化之后实现一些自定义逻辑，可以插入一个或多个自定义 BeanPostProcessor 实现。

您可以配置多个 BeanPostProcessor 实例，并且可以通过设置 order 属性来控制这些 BeanPostProcessor 实例的运行顺序。 仅当 BeanPostProcessor 实现 Ordered 接口时才能设置此属性。 如果您编写自己的 BeanPostProcessor，您也应该考虑实现 Ordered 接口。 有关更多详细信息，请参阅 BeanPostProcessor 和 Ordered 接口的 javadoc。 另请参阅有关 BeanPostProcessor 实例的编程注册的说明。

> BeanPostProcessor 实例对 bean（或对象）实例进行操作。 也就是说，Spring IoC 容器实例化一个 bean 实例，然后 BeanPostProcessor 实例完成它们的工作。
>
> BeanPostProcessor 实例的范围是每个容器的。 仅当您使用容器层次结构时，这才是相关的。 如果您在一个容器中定义了一个 BeanPostProcessor，它只会对该容器中的 bean 进行后处理。 换句话说，在一个容器中定义的 bean 不会被另一个容器中定义的 BeanPostProcessor 进行后处理，即使两个容器都是同一层次结构的一部分。
>
> 要更改实际的 bean 定义（即定义 bean 的蓝图），您需要使用 BeanFactoryPostProcessor，如使用 BeanFactoryPostProcessor 自定义配置元数据中所述。

org.springframework.beans.factory.config.BeanPostProcessor 接口正好包含两个回调方法。 当这样的类注册为容器的后处理器时，对于容器创建的每个 bean 实例，后处理器在容器初始化方法（例如 InitializingBean.afterPropertiesSet() 或 任何声明的 init 方法）被调用，并在任何 bean 初始化回调之后。 后处理器可以对 bean 实例执行任何操作，包括完全忽略后处理回调。 bean 后处理器通常检查回调接口，或者它可以用代理包装 bean。 一些 Spring AOP 基础设施类被实现为 bean 后处理器，以提供代理包装逻辑

ApplicationContext 自动检测在实现 BeanPostProcessor 接口的配置元数据中定义的任何 bean。 ApplicationContext 将这些 bean 注册为后处理器，以便稍后在 bean 创建时调用它们。 Bean 后处理器可以以与任何其他 Bean 相同的方式部署在容器中。

请注意，当在配置类上使用@Bean 工厂方法声明 BeanPostProcessor 时，工厂方法的返回类型应该是实现类本身或至少是 org.springframework.beans.factory.config.BeanPostProcessor 接口，很明显 指示该 bean 的后处理器性质。 否则，ApplicationContext 在完全创建之前无法按类型自动检测它。 由于 BeanPostProcessor 需要尽早实例化以便应用于上下文中其他 bean 的初始化，因此这种早期类型检测至关重要。

**以编程方式注册 BeanPostProcessor 实例**
虽然推荐的 BeanPostProcessor 注册方法是通过 ApplicationContext 自动检测（如前所述），但您可以使用 addBeanPostProcessor 方法以编程方式针对 ConfigurableBeanFactory 注册它们。 当您需要在注册之前评估条件逻辑时，甚至需要在层次结构中的上下文之间复制 bean 后处理器时，这会很有用。 但是请注意，以编程方式添加的 BeanPostProcessor 实例不遵守 Ordered 接口。 在这里，注册的顺序决定了执行的顺序。 另请注意，以编程方式注册的 BeanPostProcessor 实例始终在通过自动检测注册的实例之前处理，而不管任何显式排序。

**BeanPostProcessor 实例和 AOP 自动代理**
实现 BeanPostProcessor 接口的类是特殊的，容器会对其进行不同的处理。 所有 BeanPostProcessor 实例和它们直接引用的 bean 在启动时被实例化，作为 ApplicationContext 的特殊启动阶段的一部分。 接下来，所有 BeanPostProcessor 实例都以排序方式注册并应用于容器中的所有其他 bean。 因为 AOP 自动代理是作为 BeanPostProcessor 本身实现的，所以 BeanPostProcessor 实例和它们直接引用的 bean 都没有资格进行自动代理，因此，它们没有编织方面。

对于任何此类 bean，您应该看到一条信息性日志消息：Bean someBean is not eligible for getting processed by all BeanPostProcessor interfaces (for example: not eligible for auto-proxying)

如果您使用自动装配或 @Resource（可能会回退到自动装配）将 bean 连接到 BeanPostProcessor，则 Spring 在搜索类型匹配依赖项候选者时可能会访问意外的 bean，因此，使它们不符合自动代理或其他类型的条件 bean后处理。 例如，如果您有一个用 @Resource 注释的依赖项，其中字段或 setter 名称不直接对应于 bean 的声明名称，并且没有使用 name 属性，则 Spring 访问其他 bean 以按类型匹配它们。

以下示例展示了如何在 ApplicationContext 中编写、注册和使用 BeanPostProcessor 实例。

#### 示例：Hello World，BeanPostProcessor 风格

第一个示例说明了基本用法。 该示例显示了一个自定义 BeanPostProcessor 实现，它在容器创建时调用每个 bean 的 toString() 方法，并将结果字符串打印到系统控制台。

```java
package scripting;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

    // simply return the instantiated bean as-is
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // we could potentially return any object reference here...
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("Bean '" + beanName + "' created : " + bean.toString());
        return bean;
    }
}
```

以下 beans 元素使用 InstantiationTracingBeanPostProcessor：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:lang="http://www.springframework.org/schema/lang"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/lang
        https://www.springframework.org/schema/lang/spring-lang.xsd">

    <lang:groovy id="messenger"
            script-source="classpath:org/springframework/scripting/groovy/Messenger.groovy">
        <lang:property name="message" value="Fiona Apple Is Just So Dreamy."/>
    </lang:groovy>

    <!--
    when the above bean (messenger) is instantiated, this custom
    BeanPostProcessor implementation will output the fact to the system console
    -->
    <bean class="scripting.InstantiationTracingBeanPostProcessor"/>

</beans>
```

注意 InstantiationTracingBeanPostProcessor 是如何定义的。 它甚至没有名字，而且，因为它是一个 bean，所以它可以像任何其他 bean 一样进行依赖注入。 （前面的配置还定义了一个由 Groovy 脚本支持的 bean。Spring 动态语言支持在标题为动态语言支持的章节中有详细介绍。）

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.scripting.Messenger;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("scripting/beans.xml");
        Messenger messenger = ctx.getBean("messenger", Messenger.class);
        System.out.println(messenger);
    }

}
```

输出信息如下：

```sh
Bean 'messenger' created : org.springframework.scripting.groovy.GroovyMessenger@272961
org.springframework.scripting.groovy.GroovyMessenger@272961
```

#### 示例：AutowiredAnnotationBeanPostProcessor

将回调接口或注解与自定义 BeanPostProcessor 实现结合使用是扩展 Spring IoC 容器的常用方法。 一个例子是 Spring 的 AutowiredAnnotationBeanPostProcessor — 一个 BeanPostProcessor 实现，它与 Spring 发行版一起提供并自动装配带注释的字段、setter 方法和任意配置方法。

### 使用 BeanFactoryPostProcessor 自定义配置元数据

我们要查看的下一个扩展点是 org.springframework.beans.factory.config.BeanFactoryPostProcessor。 此接口的语义与 BeanPostProcessor 的语义相似，但有一个主要区别：BeanFactoryPostProcessor 对 bean 配置元数据进行操作。 也就是说，Spring IoC 容器让 BeanFactoryPostProcessor 读取配置元数据，并可能在容器实例化除 BeanFactoryPostProcessor 实例之外的任何 bean 之前更改它。

您可以配置多个 BeanFactoryPostProcessor 实例，您可以通过设置 order 属性来控制这些 BeanFactoryPostProcessor 实例的运行顺序。 但是，如果 BeanFactoryPostProcessor 实现了 Ordered 接口，则只能设置此属性。 如果您编写自己的 BeanFactoryPostProcessor，您也应该考虑实现 Ordered 接口。 有关更多详细信息，请参阅 BeanFactoryPostProcessor 和 Ordered 接口的 javadoc。

> 如果您想更改实际的 bean 实例（即，从配置元数据创建的对象），那么您需要使用 BeanPostProcessor（在前面通过使用 BeanPostProcessor 自定义 Bean 中进行了描述）。 虽然技术上可以在 BeanFactoryPostProcessor 中使用 bean 实例（例如，通过使用 BeanFactory.getBean()），但这样做会导致过早的 bean 实例化，违反标准容器生命周期。 这可能会导致负面影响，例如绕过 bean 后处理。
>
> 此外， BeanFactoryPostProcessor 实例的范围是每个容器的。 这仅在您使用容器层次结构时才相关。 如果您在一个容器中定义 BeanFactoryPostProcessor，则它仅应用于该容器中的 bean 定义。 一个容器中的 Bean 定义不会由另一个容器中的 BeanFactoryPostProcessor 实例进行后处理，即使两个容器都是同一层次结构的一部分。

bean 工厂后处理器在 ApplicationContext 中声明时会自动运行，以便将更改应用于定义容器的配置元数据。 Spring 包含许多预定义的 bean 工厂后处理器，例如 PropertyOverrideConfigurer 和 PropertySourcesPlaceholderConfigurer。 您还可以使用自定义 BeanFactoryPostProcessor — ，例如，注册自定义属性编辑器。

ApplicationContext 自动检测部署到其中的任何实现 BeanFactoryPostProcessor 接口的 bean。 它在适当的时候使用这些 bean 作为 bean factory 后处理器。 您可以像部署任何其他 bean 一样部署这些后处理器 bean。

与 BeanPostProcessors 一样，您通常不希望为延迟初始化配置 BeanFactoryPostProcessors。 如果没有其他 bean 引用 Bean(Factory)PostProcessor，则该后处理器根本不会被实例化。 因此，将其标记为延迟初始化将被忽略，即使您在 <beans /> 元素的声明中将 default-lazy-init 属性设置为 true ， Bean(Factory)PostProcessor 也将急切地实例化。

#### 示例：类名替换 PropertySourcesPlaceholderConfigurer

您可以使用 PropertySourcesPlaceholderConfigurer 使用标准的 Java 属性格式将属性值从一个单独的文件中的 bean 定义外部化。 这样做使部署应用程序的人员能够自定义特定于环境的属性，例如数据库 URL 和密码，而无需修改容器的主要 XML 定义文件或文件的复杂性或风险。

考虑以下基于 XML 的配置元数据片段，其中定义了具有占位符值的 DataSource：

```xml
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
    <property name="locations" value="classpath:com/something/jdbc.properties"/>
</bean>

<bean id="dataSource" destroy-method="close"
        class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
```

该示例显示了从外部属性文件配置的属性。 在运行时，一个 PropertySourcesPlaceholderConfigurer 被应用到元数据，替换了 DataSource 的一些属性。 要替换的值指定为 ${property-name} 形式的占位符，它遵循 Ant 和 log4j 以及 JSP EL 样式。

实际值来自标准 Java 属性格式的另一个文件：

```
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```

因此，${jdbc.username} 字符串在运行时替换为值“sa”，同样适用于与属性文件中的键匹配的其他占位符值。 PropertySourcesPlaceholderConfigurer 检查 bean 定义的大多数属性和属性中的占位符。 此外，您可以自定义占位符前缀和后缀。

使用 Spring 2.5 中引入的上下文命名空间，您可以使用专用配置元素配置属性占位符。 您可以在 location 属性中以逗号分隔列表的形式提供一个或多个位置，如以下示例所示：

```xml
<context:property-placeholder location="classpath:com/something/jdbc.properties"/>
```

PropertySourcesPlaceholderConfigurer 不仅在您指定的属性文件中查找属性。 默认情况下，如果在指定的属性文件中找不到属性，它会检查 Spring Environment 属性和常规 Java System 属性。

您可以使用 PropertySourcesPlaceholderConfigurer 来替换类名，当您必须在运行时选择特定的实现类时，这有时很有用。 以下示例显示了如何执行此操作：

```xml
<bean class="org.springframework.beans.factory.config.PropertySourcesPlaceholderConfigurer">
    <property name="locations">
        <value>classpath:com/something/strategy.properties</value>
    </property>
    <property name="properties">
        <value>custom.strategy.class=com.something.DefaultStrategy</value>
    </property>
</bean>

<bean id="serviceStrategy" class="${custom.strategy.class}"/>
```

如果该类在运行时无法解析为有效类，则在即将创建该 bean 时解析该 bean 失败，这是在非延迟初始化 bean 的 ApplicationContext 的 preInstantiateSingletons() 阶段期间。

#### `PropertyOverrideConfigurer`

PropertyOverrideConfigurer 是另一个 bean 工厂后处理器，类似于 PropertySourcesPlaceholderConfigurer，但与后者不同的是，原始定义可以为 bean 属性设置默认值或根本没有值。 如果覆盖的属性文件没有特定 bean 属性的条目，则使用默认上下文定义。

请注意，bean 定义不知道被覆盖，因此从 XML 定义文件中不能立即看出正在使用覆盖配置器。 如果多个 PropertyOverrideConfigurer 实例为同一个 bean 属性定义了不同的值，由于覆盖机制，最后一个获胜。

属性文件配置行采用以下格式：

```
beanName.property=value
```

以下清单显示了该格式的示例：

```
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql:mydb
```

此示例文件可与包含名为 dataSource 的 bean 的容器定义一起使用，该 bean 具有驱动程序和 url 属性。

还支持复合属性名称，只要路径的每个组件除了被覆盖的最终属性之外都已经是非空的（大概是由构造函数初始化的）。 在以下示例中，将 tom bean 的 fred 属性的 bob 属性的 sammy 属性设置为标量值 123：

```
tom.fred.bob.sammy=123
```

使用 Spring 2.5 中引入的上下文命名空间，可以使用专用配置元素配置属性覆盖，如以下示例所示：

```xml
<context:property-override location="classpath:override.properties"/>
```

### 使用 FactoryBean 自定义实例化逻辑

您可以为本身是工厂的对象实现 org.springframework.beans.factory.FactoryBean 接口。

FactoryBean 接口是 Spring IoC 容器实例化逻辑的可插入点。 如果您有复杂的初始化代码，可以用 Java 更好地表达，而不是（可能）冗长的 XML，您可以创建自己的 FactoryBean，在该类中编写复杂的初始化，然后将您的自定义 FactoryBean 插入到容器中。

FactoryBean<T> 接口提供了三种方法：

* T getObject()：返回此工厂创建的对象的实例。 实例可能会被共享，这取决于这个工厂是返回单例还是原型。
* boolean isSingleton()： 如果此 FactoryBean 返回单例，则返回 true，否则返回 false。 此方法的默认实现返回 true。
* Class<?> getObjectType()： 返回 getObject() 方法返回的对象类型，如果类型事先未知，则返回 null。

FactoryBean 概念和接口在 Spring Framework 中的许多地方使用。 超过 50 种 FactoryBean 接口的实现与 Spring 本身一起提供。

当您需要向容器请求实际的 FactoryBean 实例本身而不是它生成的 bean 时，请在调用 ApplicationContext 的 getBean() 方法时在 bean 的 id 前面加上与符号 (&)。 因此，对于具有 myBean id 的给定 FactoryBean，在容器上调用 getBean("myBean") 会返回 FactoryBean 的产品，而调用 getBean("&myBean") 会返回 FactoryBean 实例本身。

## 基于注解的方式配置容器

基于注解的配置的引入提出了这种方法是否比 XML“更好”的问题。简短的回答是“视情况而定”。长的答案是每种方法都有其优点和缺点，通常由开发人员决定哪种策略更适合他们。由于它们的定义方式，注解在它们的声明中提供了很多上下文，从而导致更短和更简洁的配置。然而，XML 擅长连接组件，而无需接触它们的源代码或重新编译它们。一些开发人员更喜欢将布线靠近源，而其他人则认为带注释的类不再是 POJO，而且配置变得分散且更难控制。

无论选择哪种，Spring 都可以容纳这两种风格，甚至可以将它们混合在一起。值得指出的是，通过其 JavaConfig 选项，Spring 允许以非侵入性方式使用注解，而无需触及目标组件源代码，并且在工具方面，Spring Tools for Eclipse 支持所有配置样式。

XML 设置的替代方案由基于注释的配置提供，它依赖于字节码元数据来连接组件而不是尖括号声明。 开发人员不使用 XML 来描述 bean 连接，而是通过在相关类、方法或字段声明上使用注释将配置移动到组件类本身中。 如示例：AutowiredAnnotationBeanPostProcessor 中所述，将 BeanPostProcessor 与注释结合使用是扩展 Spring IoC 容器的常用方法。

例如，Spring 2.0 引入了使用 @Required 注释强制执行必需属性的可能性。 Spring 2.5 使得遵循相同的通用方法来驱动 Spring 的依赖注入成为可能。 从本质上讲，@Autowired 注释提供了与 Autowiring Collaborators 中描述的相同的功能，但具有更细粒度的控制和更广泛的适用性。 Spring 2.5 还添加了对 JSR-250 注释的支持，例如 @PostConstruct 和 @PreDestroy。 Spring 3.0 添加了对包含在 javax.inject 包中的 JSR-330（Java 依赖注入）注解的支持，例如 @Inject 和 @Named。 有关这些注释的详细信息可以在相关部分中找到。

与往常一样，您可以将后处理器注册为单独的 bean 定义，但也可以通过在基于 XML 的 Spring 配置中包含以下标记来隐式注册它们（注意包含上下文命名空间）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

</beans>
```

<context:annotation-config/> 元素隐式注册以下后处理器：

- [`ConfigurationClassPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/context/annotation/ConfigurationClassPostProcessor.html)
- [`AutowiredAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html)
- [`CommonAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/context/annotation/CommonAnnotationBeanPostProcessor.html)
- [`PersistenceAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/orm/jpa/support/PersistenceAnnotationBeanPostProcessor.html)
- [`EventListenerMethodProcessor`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/context/event/EventListenerMethodProcessor.html)

<context:annotation-config/> 仅在定义它的同一应用程序上下文中查找 bean 上的注释。 这意味着，如果您将 <context:annotation-config/> 放在 DispatcherServlet 的 WebApplicationContext 中，它只会检查控制器中的 @Autowired bean，而不检查您的服务。 有关更多信息，请参阅 DispatcherServlet。

### @Required

@Required 注释适用于 bean 属性 setter 方法，如下例所示：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Required
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

此注释表明必须在配置时通过 bean 定义中的显式属性值或通过自动装配来填充受影响的 bean 属性。 如果尚未填充受影响的 bean 属性，则容器将引发异常。 这允许急切和显式失败，避免以后出现 NullPointerException 实例等。 我们仍然建议您将断言放入 bean 类本身（例如，放入 init 方法）。 这样做会强制执行那些必需的引用和值，即使您在容器外部使用该类。

RequiredAnnotationBeanPostProcessor 必须注册为 bean 才能启用对 @Required 注释的支持。

@Required annotation 和 RequiredAnnotationBeanPostProcessor 从 Spring Framework 5.1 开始正式弃用，赞成使用构造函数注入进行所需设置（或 InitializingBean.afterPropertiesSet() 的自定义实现或自定义 @PostConstruct 方法以及 bean 属性 setter 方法）。

### `@Autowired`

您可以将 @Autowired 注释应用于构造函数，如以下示例所示：

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

从 Spring Framework 4.3 开始，如果目标 bean 只定义了一个构造函数，则不再需要在此类构造函数上添加 @Autowired 注释。 但是，如果有多个构造函数可用并且没有主/默认构造函数，则必须至少用 @Autowired 注释构造函数之一，以便指示容器使用哪个构造函数。 有关详细信息，请参阅有关构造函数解析的讨论。

您还可以将 @Autowired 注释应用于传统的 setter 方法，如以下示例所示：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

您还可以将注释应用于具有任意名称和多个参数的方法，如以下示例所示：

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

您也可以将 @Autowired 应用于字段，甚至将其与构造函数混合使用，如下例所示：

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    private MovieCatalog movieCatalog;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

您还可以通过将 @Autowired 注释添加到需要该类型数组的字段或方法来指示 Spring 从 ApplicationContext 提供特定类型的所有 bean，如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    private MovieCatalog[] movieCatalogs;

    // ...
}
```

这同样适用于类型化集合，如以下示例所示：

```java
public class MovieRecommender {

    private Set<MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
}
```

> 如果您希望数组或列表中的项目按特定顺序排序，您的目标 bean 可以实现 org.springframework.core.Ordered 接口或使用 @Order 或标准 @Priority 注释。 否则，它们的顺序遵循容器中相应目标 bean 定义的注册顺序。
>
> 您可以在目标类级别和 @Bean 方法上声明 @Order 注释，可能用于单个 bean 定义（在多个定义使用相同 bean 类的情况下）。 @Order 值可能会影响注入点的优先级，但请注意，它们不会影响单例启动顺序，这是由依赖关系和 @DependsOn 声明决定的正交问题。
>
> 请注意，标准 javax.annotation.Priority 注释在 @Bean 级别不可用，因为它不能在方法上声明。 它的语义可以通过@Order 值结合@Primary 在每个类型的单个bean 上建模。

只要预期的键类型是字符串，即使是类型化的 Map 实例也可以自动装配。 映射值包含预期类型的所有 bean，键包含相应的 bean 名称，如以下示例所示：

```java
public class MovieRecommender {

    private Map<String, MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Map<String, MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
}
```

默认情况下，当给定注入点没有匹配的候选 bean 时，自动装配失败。 对于声明的数组、集合或映射，至少需要一个匹配元素。

默认行为是将带注释的方法和字段视为指示所需的依赖项。 您可以更改此行为，如下例所示，通过将不可满足的注入点标记为非必需（即，通过将 @Autowired 中的 required 属性设置为 false），使框架能够跳过不可满足的注入点：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired(required = false)
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

如果非必需方法的依赖项（或其依赖项之一，如果有多个参数）不可用，则根本不会调用它。 在这种情况下，根本不会填充非必填字段，而是保留其默认值。

注入的构造函数和工厂方法参数是一种特殊情况，因为由于 Spring 的构造函数解析算法可能潜在地处理多个构造函数，@Autowired 中的 required 属性具有一些不同的含义。 构造函数和工厂方法参数在默认情况下是有效的，但在单构造函数场景中有一些特殊规则，例如多元素注入点（数组、集合、映射）在没有匹配的 bean 可用时解析为空实例。 这允许一种通用的实现模式，其中所有依赖项都可以在唯一的多参数构造函数中声明——例如，声明为没有 @Autowired 注释的单个公共构造函数。

或者，您可以通过 Java 8 的 java.util.Optional 表达特定依赖项的非必需性质，如以下示例所示：

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(Optional<MovieFinder> movieFinder) {
        ...
    }
}
```

从 Spring Framework 5.0 开始，您还可以使用 @Nullable 注释（任何包中的任何类型 — 例如，来自 JSR-305 的 javax.annotation.Nullable）或仅利用 Kotlin 内置的空安全支持：

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        ...
    }
}
```

您还可以将 @Autowired 用于众所周知的可解析依赖项的接口：BeanFactory、ApplicationContext、Environment、ResourceLoader、ApplicationEventPublisher 和 MessageSource。 这些接口及其扩展接口（例如 ConfigurableApplicationContext 或 ResourcePatternResolver）会自动解析，无需特殊设置。 以下示例自动装配 ApplicationContext 对象：

```java
public class MovieRecommender {

    @Autowired
    private ApplicationContext context;

    public MovieRecommender() {
    }

    // ...
}
```



@Autowired、@Inject、@Value 和 @Resource 注释由 Spring BeanPostProcessor 实现处理。 这意味着您不能在自己的 BeanPostProcessor 或 BeanFactoryPostProcessor 类型（如果有）中应用这些注释。 这些类型必须使用 XML 或 Spring @Bean 方法显式“连接”。

### 使用@Primary 微调基于注解的自动装配

由于按类型自动装配可能会导致多个候选对象，因此通常需要对选择过程进行更多控制。 实现此目的的一种方法是使用 Spring 的 @Primary 注释。 @Primary 表示当多个 bean 是自动装配到单值依赖项的候选者时，应优先考虑特定 bean。 如果候选中恰好存在一个主要 bean，则它成为自动装配的值。

考虑以下将 firstMovieCatalog 定义为主要 MovieCatalog 的配置：

```java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }

    @Bean
    public MovieCatalog secondMovieCatalog() { ... }

    // ...
}
```

使用上述配置，以下 MovieRecommender 自动装配到 firstMovieCatalog：

```java
public class MovieRecommender {

    @Autowired
    private MovieCatalog movieCatalog;

    // ...
}
```

相应的bean定义如下：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog" primary="true">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

### 使用Qualifiers微调基于注释的自动装配

@Primary 是一种有效的方法，当可以确定一个主要候选对象时，可以通过多个实例按类型使用自动装配。 当您需要对选择过程进行更多控制时，可以使用 Spring 的 @Qualifier 注解。 您可以将限定符值与特定参数相关联，缩小类型匹配的范围，以便为每个参数选择一个特定的 bean。 在最简单的情况下，这可以是一个简单的描述性值，如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog;

    // ...
}
```

您还可以在单个构造函数参数或方法参数上指定 @Qualifier 注释，如以下示例所示：

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main") MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

以下示例显示了相应的 bean 定义。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/> 

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/> 

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

对于回退匹配，bean 名称被视为默认限定符值。 因此，您可以使用 main 的 id 而不是嵌套的 qualifier 元素来定义 bean，从而导致相同的匹配结果。 但是，尽管您可以使用此约定按名称引用特定 bean，但 @Autowired 从根本上讲是关于带有可选语义限定符的类型驱动注入。 这意味着限定符值，即使使用 bean 名称回退，在类型匹配集中始终具有缩小语义。 它们不会在语义上表达对唯一 bean id 的引用。 好的限定符值是 main 或 EMEA 或persistent，表达独立于 bean id 的特定组件的特征，在匿名 bean 定义（例如前面示例中的定义）的情况下可能会自动生成

限定符也适用于类型化集合，如前所述 — 例如，适用于 Set<MovieCatalog>。 在这种情况下，根据声明的限定符，所有匹配的 bean 作为集合注入。 这意味着限定符不必是唯一的。 相反，它们构成过滤标准。 例如，您可以使用相同的限定符值“action”定义多个 MovieCatalog bean，所有这些 bean 都被注入到一个用 @Qualifier("action") 注释的 Set<MovieCatalog> 中。

在类型匹配的候选中，让限定符值根据目标 bean 名称进行选择，不需要在注入点使用 @Qualifier 注释。 如果没有其他解析指示符（例如限定符或主标记），对于非唯一依赖的情况，Spring 将注入点名称（即字段名称或参数名称）与目标 bean 名称进行匹配并选择 同名候选人，如果有的话。

也就是说，如果您打算按名称表达注解驱动的注入，请不要主要使用 @Autowired，即使它能够在类型匹配的候选中按 bean 名称进行选择。 相反，请使用 JSR-250 @Resource 注释，该注释在语义上定义为通过其唯一名称标识特定目标组件，声明的类型与匹配过程无关。 @Autowired 有相当不同的语义：按类型选择候选 bean 后，指定的 String 限定符值仅在那些类型选择的候选中考虑（例如，将帐户限定符与标记有相同限定符标签的 bean 匹配）。

对于本身定义为集合、映射或数组类型的 bean，@Resource 是一个很好的解决方案，通过唯一名称引用特定的集合或数组 bean。 也就是说，从 4.3 开始，您也可以通过 Spring 的 @Autowired 类型匹配算法匹配集合、Map 和数组类型，只要元素类型信息保留在 @Bean 返回类型签名或集合继承层次结构中即可。 在这种情况下，您可以使用限定符值在相同类型的集合中进行选择，如上一段所述。

从 4.3 开始，@Autowired 还考虑注入的自引用（即对当前注入的 bean 的引用）。 请注意，自注入是一种后备方法。 对其他组件的常规依赖始终具有优先级。 从这个意义上说，自我推荐不参与常规的候选人选择，因此特别是从不主要。 相反，它们总是以最低的优先级结束。 在实践中，您应该仅将自引用用作最后的手段（例如，通过 bean 的事务代理调用同一实例上的其他方法）。 在这种情况下，考虑将受影响的方法分解为单独的委托 bean。 或者，您可以使用@Resource，它可以通过其唯一名称获取返回当前 bean 的代理。

尝试将 @Bean 方法的结果注入同一配置类也是一种有效的自引用场景。 要么在实际需要的方法签名中延迟解析此类引用（而不是配置类中的自动装配字段），要么将受影响的@Bean 方法声明为静态方法，将它们与包含的配置类实例及其生命周期分离。 否则，仅在回退阶段考虑此类 bean，而将其他配置类上的匹配 bean 选为主要候选对象（如果可用）。

@Autowired 适用于字段、构造函数和多参数方法，允许通过参数级别的限定符注释来缩小范围。 相比之下，@Resource 仅支持带有单个参数的字段和 bean 属性 setter 方法。 因此，如果您的注入目标是构造函数或多参数方法，您应该坚持使用限定符。

您可以创建自己的自定义限定符注释。 为此，请定义注释并在定义中提供 @Qualifier 注释，如以下示例所示：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

    String value();
}
```

然后，您可以在自动装配的字段和参数上提供自定义限定符，如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    @Genre("Action")
    private MovieCatalog actionCatalog;

    private MovieCatalog comedyCatalog;

    @Autowired
    public void setComedyCatalog(@Genre("Comedy") MovieCatalog comedyCatalog) {
        this.comedyCatalog = comedyCatalog;
    }

    // ...
}
```

接下来，您可以提供候选 bean 定义的信息。 您可以添加 <qualifier/> 标签作为 <bean/> 标签的子元素，然后指定类型和值以匹配您的自定义限定符注释。 该类型与注释的完全限定类名匹配。 或者，如果不存在名称冲突的风险，为了方便起见，您可以使用短类名称。 以下示例演示了这两种方法：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="Genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="example.Genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

在某些情况下，使用没有值的注释可能就足够了。 当注释用于更通用的目的并且可以应用于多种不同类型的依赖项时，这可能很有用。 例如，您可以提供一个离线目录，当没有可用的 Internet 连接时可以搜索该目录。 首先，定义简单的注解，如下例所示：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Offline {

}
```

然后将注解添加到要自动装配的字段或属性中，如下例所示：

```java
public class MovieRecommender {

    @Autowired
    @Offline 
    private MovieCatalog offlineCatalog;

    // ...
}
```

现在 bean 定义只需要一个限定符类型，如以下示例所示：

```xml
<bean class="example.SimpleMovieCatalog">
    <qualifier type="Offline"/> 
    <!-- inject any dependencies required by this bean -->
</bean>
```

除了简单值属性之外，您还可以定义自定义限定符注释，这些注释接受命名属性。 如果随后在要自动装配的字段或参数上指定了多个属性值，则 bean 定义必须匹配所有此类属性值才能被视为自动装配候选者。 例如，请考虑以下注释定义：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MovieQualifier {

    String genre();

    Format format();
}
```

在这种情况下 Format 是一个枚举，定义如下：

```java
public enum Format {
    VHS, DVD, BLURAY
}
```

要自动装配的字段使用自定义限定符进行注释，并包含两个属性的值：流派和格式，如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Action")
    private MovieCatalog actionVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Comedy")
    private MovieCatalog comedyVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.DVD, genre="Action")
    private MovieCatalog actionDvdCatalog;

    @Autowired
    @MovieQualifier(format=Format.BLURAY, genre="Comedy")
    private MovieCatalog comedyBluRayCatalog;

    // ...
}
```

最后，bean 定义应该包含匹配的限定符值。 此示例还演示了您可以使用 bean 元属性代替 <qualifier/> 元素。 如果可用， <qualifier/> 元素及其属性优先，但如果不存在这样的限定符，自动装配机制将回退到 <meta/> 标签中提供的值，如以下示例中的最后两个 bean 定义 ：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Action"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Comedy"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="DVD"/>
        <meta key="genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="BLURAY"/>
        <meta key="genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

</beans>
```

### 使用泛型作为自动装配限定符

除了@Qualifier 注释之外，您还可以使用 Java 泛型类型作为限定的隐式形式。 例如，假设您有以下配置：

```java
@Configuration
public class MyConfiguration {

    @Bean
    public StringStore stringStore() {
        return new StringStore();
    }

    @Bean
    public IntegerStore integerStore() {
        return new IntegerStore();
    }
}
```

假设前面的bean实现了一个泛型接口，（即Store<String>和Store<Integer>），你可以@Autowire Store接口，并使用泛型作为限定符，如下例所示：

```java
@Autowired
private Store<String> s1; // <String> qualifier, injects the stringStore bean

@Autowired
private Store<Integer> s2; // <Integer> qualifier, injects the integerStore bean
```

通用限定符也适用于自动装配列表、Map 实例和数组。 以下示例自动装配通用列表：

```java
// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```

### `CustomAutowireConfigurer`

CustomAutowireConfigurer 是一个 BeanFactoryPostProcessor，它允许您注册自己的自定义限定符注解类型，即使它们没有使用 Spring 的 @Qualifier 注解进行注解。 以下示例显示了如何使用 CustomAutowireConfigurer：

```xml
<bean id="customAutowireConfigurer"
        class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
    <property name="customQualifierTypes">
        <set>
            <value>example.CustomQualifier</value>
        </set>
    </property>
</bean>
```

AutowireCandidateResolver 通过以下方式确定自动装配候选者：

* 每个 bean 定义的autowire-candidate值
* <beans/> 元素上可用的任何 default-autowire-candidates 模式
* @Qualifier 注释和向 CustomAutowireConfigurer 注册的任何自定义注释的存在

当多个 bean 有资格作为自动装配候选时，“主要”的确定如下：如果候选中恰好有一个 bean 定义的主要属性设置为 true，则选择它。

### `@Resource`

Spring 还通过在字段或 bean 属性 setter 方法上使用 JSR-250 @Resource 注释 (javax.annotation.Resource) 来支持注入。 这是 Java EE 中的常见模式：例如，在 JSF 管理的 bean 和 JAX-WS 端点中。 对于 Spring 管理的对象，Spring 也支持这种模式。

@Resource 采用 name 属性。 默认情况下，Spring 将该值解释为要注入的 bean 名称。 换句话说，它遵循按名称语义，如以下示例所示：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource(name="myMovieFinder") 
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

如果未明确指定名称，则默认名称源自字段名称或 setter 方法。 如果是字段，则采用字段名称。 在 setter 方法的情况下，它采用 bean 属性名称。 下面的例子将把名为 movieFinder 的 bean 注入到它的 setter 方法中：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

随注释提供的名称由 CommonAnnotationBeanPostProcessor 知道的 ApplicationContext 解析为 bean 名称。 如果显式配置 Spring 的 SimpleJndiBeanFactory，则可以通过 JNDI 解析名称。 但是，我们建议您依赖默认行为并使用 Spring 的 JNDI 查找功能来保留间接级别。

在没有指定显式名称的 @Resource 用法的唯一情况下，与 @Autowired 类似，@Resource 查找主要类型匹配而不是特定的命名 bean 并解析众所周知的可解析依赖项：BeanFactory、ApplicationContext、ResourceLoader、ApplicationEventPublisher 和 消息源接口。

因此，在以下示例中，customerPreferenceDao 字段首先查找名为“customerPreferenceDao”的 bean，然后回退到 CustomerPreferenceDao 类型的主要类型匹配：

```java
public class MovieRecommender {

    @Resource
    private CustomerPreferenceDao customerPreferenceDao;

    @Resource
    private ApplicationContext context; 

    public MovieRecommender() {
    }

    // ...
}
```

### `@Value`

@Value 通常用于注入外化属性：

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name}") String catalog) {
        this.catalog = catalog;
    }
}
```

使用以下配置：

```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig { }
```

以及以下 application.properties 文件：

```java
catalog.name=MovieCatalog
```

在这种情况下，catalog 参数和字段将等于 MovieCatalog 值。

Spring 提供了一个默认的宽松嵌入值解析器。 它将尝试解析属性值，如果无法解析，则属性名称（例如 ${catalog.name}）将作为值注入。 如果你想对不存在的值保持严格的控制，你应该声明一个 PropertySourcesPlaceholderConfigurer bean，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

> 使用 JavaConfig 配置 PropertySourcesPlaceholderConfigurer 时，@Bean 方法必须是静态的。

如果无法解析任何 ${} 占位符，则使用上述配置可确保 Spring 初始化失败。 也可以使用 setPlaceholderPrefix、setPlaceholderSuffix 或 setValueSeparator 等方法来自定义占位符。

Spring Boot 默认配置一个 PropertySourcesPlaceholderConfigurer bean，它将从 application.properties 和 application.yml 文件中获取属性。

Spring 提供的内置转换器支持允许自动处理简单的类型转换（例如到 Integer 或 int）。 多个逗号分隔的值可以自动转换为 String 数组，无需额外的努力。可以提供如下默认值：

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name:defaultCatalog}") String catalog) {
        this.catalog = catalog;
    }
}
```

Spring BeanPostProcessor 在幕后使用 ConversionService 来处理将 @Value 中的 String 值转换为目标类型的过程。 如果您想为您自己的自定义类型提供转换支持，您可以提供您自己的 ConversionService bean 实例，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
        conversionService.addConverter(new MyCustomConverter());
        return conversionService;
    }
}
```

当 @Value 包含 SpEL 表达式时，该值将在运行时动态计算，如下例所示：

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("#{systemProperties['user.catalog'] + 'Catalog' }") String catalog) {
        this.catalog = catalog;
    }
}
```

SpEL 还支持使用更复杂的数据结构：

```java
@Component
public class MovieRecommender {

    private final Map<String, Integer> countOfMoviesPerCatalog;

    public MovieRecommender(
            @Value("#{{'Thriller': 100, 'Comedy': 300}}") Map<String, Integer> countOfMoviesPerCatalog) {
        this.countOfMoviesPerCatalog = countOfMoviesPerCatalog;
    }
}
```

### 使用`@PostConstruct`和`@PreDestroy`

CommonAnnotationBeanPostProcessor 不仅可以识别 @Resource 注释，还可以识别 JSR-250 生命周期注释：javax.annotation.PostConstruct 和 javax.annotation.PreDestroy。 在 Spring 2.5 中引入，对这些注解的支持提供了初始化回调和销毁回调中描述的生命周期回调机制的替代方案。 假设 CommonAnnotationBeanPostProcessor 已在 Spring ApplicationContext 中注册，则在生命周期中与相应的 Spring 生命周期接口方法或显式声明的回调方法相同的点调用带有这些注释之一的方法。 在以下示例中，缓存在初始化时预填充并在销毁时清除：

```java
public class CachingMovieLister {

    @PostConstruct
    public void populateMovieCache() {
        // populates the movie cache upon initialization...
    }

    @PreDestroy
    public void clearMovieCache() {
        // clears the movie cache upon destruction...
    }
}
```

### 类路径扫描和托管组件

本节描述了通过扫描类路径隐式检测候选组件的选项。 候选组件是与过滤条件匹配的类，并在容器中注册了相应的 bean 定义。 这消除了使用 XML 执行 bean 注册的需要。 相反，您可以使用注释（例如，@Component）、AspectJ 类型表达式或您自己的自定义过滤条件来选择哪些类具有向容器注册的 bean 定义。

#### `@Component`

@Repository 注释是任何满足存储库角色或构造型（也称为数据访问对象或 DAO）的类的标记。 此标记的用途之一是异常的自动转换，如异常转换中所述。

Spring 提供了更多的构造型注解：@Component、@Service 和 @Controller。 @Component 是任何 Spring 管理的组件的通用构造型。 @Repository、@Service 和@Controller 是@Component 的特化，用于更具体的用例（分别在持久层、服务层和表示层）。因此，您可以使用@Component 注释您的组件类，但是，通过使用@Repository、@Service 或@Controller 来注释它们，您的类更适合由工具处理或与方面相关联。例如，这些构造型注释是切入点的理想目标。 @Repository、@Service 和 @Controller 还可以在 Spring Framework 的未来版本中携带额外的语义。因此，如果您在服务层使用 @Component 或 @Service 之间做出选择，@Service 显然是更好的选择。同样，如前所述，@Repository 已经被支持作为持久层中自动异常转换的标记。

#### 使用元注释和组合注释

Spring 提供的许多注解都可以在您自己的代码中用作元注解。 元注释是可以应用于另一个注释的注释。 比如前面提到的@Service注解就是用@Component元注解的，如下例所示：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component 
public @interface Service {

    // ...
}
```

您还可以组合元注释来创建“组合注释”。 例如，Spring MVC 中的@RestController 注解就是由@Controller 和@ResponseBody 组成的。

此外，组合注释可以选择性地从元注释重新声明属性以允许自定义。 当您只想公开元注释属性的子集时，这尤其有用。 例如，Spring 的 @SessionScope 注释将作用域名称硬编码为会话，但仍允许自定义 proxyMode。 以下清单显示了 SessionScope 注释的定义：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

    /**
     * Alias for {@link Scope#proxyMode}.
     * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
     */
    @AliasFor(annotation = Scope.class)
    ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

然后您可以使用 @SessionScope 而不声明 proxyMode 如下：

```java
@Service
@SessionScope
public class SessionScopedService {
    // ...
}
```

您还可以覆盖 proxyMode 的值，如以下示例所示：

```java
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
    // ...
}
```

For further details, see the [Spring Annotation Programming Model](https://github.com/spring-projects/spring-framework/wiki/Spring-Annotation-Programming-Model) wiki page.

#### 自动检测类并注册 Bean 定义

Spring 可以自动检测构造型类并将相应的 BeanDefinition 实例注册到 ApplicationContext。 例如，以下两个类有资格进行此类自动检测：

```java
@Service
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

```java
@Repository
public class JpaMovieFinder implements MovieFinder {
    // implementation elided for clarity
}
```

要自动检测这些类并注册相应的bean，您需要将@ComponentScan 添加到@Configuration 类中，其中basePackages 属性是两个类的公共父包。 （或者，您可以指定包含每个类的父包的逗号或分号或空格分隔的列表。）

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    // ...
}
```

以下替代方案使用 XML：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example"/>

</beans>
```

> <context:component-scan> 的使用隐式启用了 <context:annotation-config> 的功能。 使用 <context:component-scan> 时，通常不需要包含 <context:annotation-config> 元素。

此外，当您使用组件扫描元素时， AutowiredAnnotationBeanPostProcessor 和 CommonAnnotationBeanPostProcessor 都隐式包含在内。 这意味着这两个组件被自动检测并连接在一起 — 都没有在 XML 中提供任何 bean 配置元数据。

您可以通过包含值为 false 的 annotation-config 属性来禁用 AutowiredAnnotationBeanPostProcessor 和 CommonAnnotationBeanPostProcessor 的注册。

#### 使用过滤器自定义扫描

默认情况下，用@Component、@Repository、@Service、@Controller、@Configuration 注释的类或本身用@Component 注释的自定义注释是唯一检测到的候选组件。 但是，您可以通过应用自定义过滤器来修改和扩展此行为。 添加它们作为 @ComponentScan 注释的 includeFilters 或 excludeFilters 属性（或作为 XML 配置中 <context:component-scan> 元素的 <context:include-filter /> 或 <context:exclude-filter /> 子元素）。 每个过滤器元素都需要 type 和 expression 属性。 下表描述了过滤选项：

| Filter Type          | Example Expression           | Description                                                  |
| :------------------- | :--------------------------- | :----------------------------------------------------------- |
| annotation (default) | `org.example.SomeAnnotation` | An annotation to be *present* or *meta-present* at the type level in target components. |
| assignable           | `org.example.SomeClass`      | A class (or interface) that the target components are assignable to (extend or implement). |
| aspectj              | `org.example..*Service+`     | An AspectJ type expression to be matched by the target components. |
| regex                | `org\.example\.Default.*`    | A regex expression to be matched by the target components' class names. |
| custom               | `org.example.MyTypeFilter`   | A custom implementation of the `org.springframework.core.type.TypeFilter` interface. |

以下示例显示了忽略所有 @Repository 注释并使用“存根”存储库的配置：

```java
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    // ...
}
```

等同的xml：

```xml
<beans>
    <context:component-scan base-package="org.example">
        <context:include-filter type="regex"
                expression=".*Stub.*Repository"/>
        <context:exclude-filter type="annotation"
                expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
</beans>
```

您还可以通过在注释上设置 useDefaultFilters=false 或通过提供 use-default-filters="false" 作为 <component-scan/> 元素的属性来禁用默认过滤器。 这有效地禁用了对使用 @Component、@Repository、@Service、@Controller、@RestController 或 @Configuration 进行注释或元注释的类的自动检测。

#### 在组件中定义 Bean 元数据

Spring 组件还可以将 bean 定义元数据提供给容器。 您可以使用用于在 @Configuration 注释类中定义 bean 元数据的相同 @Bean 注释来执行此操作。 以下示例显示了如何执行此操作：

```java
@Component
public class FactoryMethodComponent {

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    public void doWork() {
        // Component method implementation omitted
    }
}
```

前面的类是一个 Spring 组件，它的 doWork() 方法中有特定于应用程序的代码。 但是，它还提供了一个 bean 定义，该定义具有一个引用方法 publicInstance() 的工厂方法。 @Bean 批注通过@Qualifier 批注标识工厂方法和其他bean 定义属性，例如限定符值。 其他可以指定的方法级注解是@Scope、@Lazy 和自定义限定符注解。

除了用于组件初始化的作用外，您还可以在标记为@Autowired 或@Inject 的注入点上放置@Lazy 注解。 在这种情况下，它导致了惰性解析代理的注入。

如前所述，支持自动装配字段和方法，并额外支持 @Bean 方法的自动装配。 以下示例显示了如何执行此操作：

```java
@Component
public class FactoryMethodComponent {

    private static int i;

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    // use of a custom qualifier and autowiring of method parameters
    @Bean
    protected TestBean protectedInstance(
            @Qualifier("public") TestBean spouse,
            @Value("#{privateInstance.age}") String country) {
        TestBean tb = new TestBean("protectedInstance", 1);
        tb.setSpouse(spouse);
        tb.setCountry(country);
        return tb;
    }

    @Bean
    private TestBean privateInstance() {
        return new TestBean("privateInstance", i++);
    }

    @Bean
    @RequestScope
    public TestBean requestScopedInstance() {
        return new TestBean("requestScopedInstance", 3);
    }
}
```

该示例将 String 方法参数 country 自动装配到另一个名为 privateInstance 的 bean 上的 age 属性值。 Spring 表达式语言元素通过符号 #{ <expression> } 定义属性的值。 对于@Value 注释，表达式解析器已预先配置为在解析表达式文本时查找 bean 名称。

从 Spring Framework 4.3 开始，您还可以声明类型为 InjectionPoint（或其更具体的子类：DependencyDescriptor）的工厂方法参数来访问触发当前 bean 创建的请求注入点。 请注意，这仅适用于 bean 实例的实际创建，不适用于现有实例的注入。 因此，此功能对于原型范围的 bean 最有意义。 对于其他范围，工厂方法只会看到在给定范围内触发创建新 bean 实例的注入点（例如，触发创建惰性单例 bean 的依赖项）。 在这种情况下，您可以使用提供的注入点元数据并注意语义。 以下示例显示了如何使用 InjectionPoint：

```java
@Component
public class FactoryMethodComponent {

    @Bean @Scope("prototype")
    public TestBean prototypeInstance(InjectionPoint injectionPoint) {
        return new TestBean("prototypeInstance for " + injectionPoint.getMember());
    }
}
```

> 您可以将@Bean 方法声明为静态方法，允许在不将其包含的配置类创建为实例的情况下调用它们。 这在定义后处理器 bean（例如，BeanFactoryPostProcessor 或 BeanPostProcessor 类型）时特别有意义，因为此类 bean 在容器生命周期的早期被初始化，并且应避免在那时触发配置的其他部分。
>
> 由于技术限制，对静态 @Bean 方法的调用永远不会被容器拦截，即使在 @Configuration 类中也不行（如本节前面所述）：CGLIB 子类化只能覆盖非静态方法。 因此，对另一个@Bean 方法的直接调用具有标准的 Java 语义，导致直接从工厂方法本身返回一个独立的实例。
>
> @Bean 方法的 Java 语言可见性不会对 Spring 容器中生成的 bean 定义产生直接影响。 您可以在非@Configuration 类以及任何地方的静态方法中自由地声明您认为合适的工厂方法。 但是，@Configuration 类中的常规@Bean 方法需要是可覆盖的 — 也就是说，它们不能被声明为private 或final。
>
> @Bean 方法也可以在给定组件或配置类的基类上发现，以及在组件或配置类实现的接口中声明的 Java 8 默认方法上。 这为组合复杂的配置安排提供了很大的灵活性，甚至可以通过 Java 8 默认方法（从 Spring 4.2 开始）进行多重继承。
>
> 最后，单个类可以为同一个 bean 保存多个 @Bean 方法，作为多个工厂方法的安排，根据运行时的可用依赖项使用。 这与在其他配置场景中选择“最贪婪”的构造函数或工厂方法的算法相同：在构造时选择具有最多可满足依赖项的变体，类似于容器如何在多个 @Autowired 构造函数之间进行选择。

#### 命名自动检测组件

当一个组件作为扫描过程的一部分被自动检测时，它的 bean 名称由该扫描器已知的 BeanNameGenerator 策略生成。 默认情况下，任何包含名称值的 Spring 构造型批注（@Component、@Repository、@Service 和 @Controller）从而将该名称提供给相应的 bean 定义。

如果此类注释不包含名称值或任何其他检测到的组件（例如自定义过滤器发现的组件），则默认 bean 名称生成器返回未大写的非限定类名称。 例如，如果检测到以下组件类，则名称将是 myMovieLister 和 movieFinderImpl：

```java
@Service("myMovieLister")
public class SimpleMovieLister {
    // ...
}
```

```java
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

如果不想依赖默认的 bean 命名策略，可以提供自定义 bean 命名策略。 首先，实现 BeanNameGenerator 接口，并确保包含一个默认的无参数构造函数。 然后，在配置扫描器时提供完全限定的类名，如以下示例注释和 bean 定义所示。

```java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example"
        name-generator="org.example.MyNameGenerator" />
</beans>
```

> 如果由于多个自动检测到的组件具有相同的非限定类名（即，具有相同名称但驻留在不同包中的类）而遇到命名冲突，您可能需要配置一个 BeanNameGenerator，该 BeanNameGenerator 默认为该类的完全限定类名。 生成的 bean 名称。 从 Spring Framework 5.2.3 开始，位于 org.springframework.context.annotation 包中的fullyQualifiedAnnotationBeanNameGenerator 可用于此类目的。

作为一般规则，当其他组件可能对其进行显式引用时，请考虑使用注释指定名称。 另一方面，只要容器负责接线，自动生成的名称就足够了。

#### 为自动检测组件提供范围

与 Spring 管理的组件一样，自动检测组件的默认和最常见的范围是单例。 但是，有时您需要一个可以由 @Scope 批注指定的不同范围。 您可以在注释中提供范围的名称，如以下示例所示：

```java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

@Scope 注释仅在具体 bean 类（对于带注释的组件）或工厂方法（对于 @Bean 方法）上进行内省。 与 XML bean 定义相反，没有 bean 定义继承的概念，并且类级别的继承层次结构与元数据目的无关。

要为范围解析提供自定义策略而不是依赖基于注释的方法，您可以实现 ScopeMetadataResolver 接口。 确保包含默认的无参数构造函数。 然后您可以在配置扫描器时提供完全限定的类名，如下面的注释和 bean 定义示例所示：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example" scope-resolver="org.example.MyScopeResolver"/>
</beans>
```

当使用某些非单一作用域时，可能需要为作用域对象生成代理。 为此，可以在 component-scan 元素上使用 scoped-proxy 属性。 三个可能的值是：no、interfaces 和 targetClass。 例如，以下配置会产生标准的 JDK 动态代理：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopedProxy = ScopedProxyMode.INTERFACES)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example" scoped-proxy="interfaces"/>
</beans>
```

#### 提供带有注释的限定符元数据

@Qualifier 注释在使用限定符微调基于注释的自动装配中讨论。 该部分中的示例演示了使用 @Qualifier 注释和自定义限定符注释在解析自动装配候选时提供细粒度控制。 因为这些示例基于 XML bean 定义，所以通过使用 XML 中 bean 元素的限定符或元子元素，在候选 bean 定义上提供限定符元数据。 当依靠类路径扫描来自动检测组件时，您可以提供带有候选类的类型级别注释的限定符元数据。 以下三个示例演示了此技术：

```java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}
```

```java
@Component
@Genre("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}
```

```java
@Component
@Offline
public class CachingMovieCatalog implements MovieCatalog {
    // ...
}
```

#### 生成候选组件的索引

虽然类路径扫描非常快，但可以通过在编译时创建一个静态候选列表来提高大型应用程序的启动性能。 在此模式下，所有作为组件扫描目标的模块都必须使用此机制。

您现有的 @ComponentScan 或 <context:component-scan/> 指令必须保持不变才能请求上下文扫描某些包中的候选对象。 当 ApplicationContext 检测到这样的索引时，它会自动使用它而不是扫描类路径。

要生成索引，请向包含作为组件扫描指令目标的组件的每个模块添加额外的依赖项。 以下示例显示了如何使用 Maven 执行此操作：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-indexer</artifactId>
        <version>5.3.10</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

```sh
dependencies {
    annotationProcessor "org.springframework:spring-context-indexer:5.3.10"
}
```

spring-context-indexer 工件生成一个 META-INF/spring.components 文件，该文件包含在 jar 文件中。

在 IDE 中使用此模式时，spring-context-indexer 必须注册为注释处理器，以确保在候选组件更新时索引是最新的。

当在类路径上找到 META-INF/spring.components 文件时，索引会自动启用。 如果索引对某些库（或用例）部分可用但无法为整个应用程序构建，则可以通过设置 spring.index.ignore 回退到常规类路径安排（好像根本不存在索引） 为真，无论是作为 JVM 系统属性还是通过 SpringProperties 机制。

## 使用 JSR 330 标准注解

从 Spring 3.0 开始，Spring 提供对 JSR-330 标准注解（依赖注入）的支持。 这些注释的扫描方式与 Spring 注释相同。 要使用它们，您需要在类路径中包含相关的 jar。

```xml
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```

您可以使用@javax.inject.Inject 代替@Autowired

您可以使用 @javax.inject.Named 或 @javax.annotation.ManagedBean 代替 @Component

## 基于java的容器配置

### `@Bean` and `@Configuration`

Spring 新的 Java 配置支持中的核心工件是 @Configuration-annotated 类和 @Bean-annotated 方法。

@Bean 注解用于指示一个方法实例化、配置和初始化一个由 Spring IoC 容器管理的新对象。 对于那些熟悉 Spring 的 <beans/> XML 配置的人来说，@Bean 注释与 <bean/> 元素扮演着相同的角色。 您可以将带有 @Bean 注释的方法与任何 Spring @Component 一起使用。 但是，它们最常与 @Configuration bean 一起使用。

用 @Configuration 注释一个类表明它的主要目的是作为 bean 定义的来源。 此外，@Configuration 类允许通过调用同一类中的其他 @Bean 方法来定义 bean 间的依赖关系。 最简单的@Configuration 类如下所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

前面的 AppConfig 类相当于下面的 Spring <beans/> XML：

```xml
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

完整的@Configuration 与“精简版”@Bean 模式？

当 @Bean 方法在没有用 @Configuration 注释的类中声明时，它们被称为在“精简”模式下处理。 在@Component 或什至在一个普通的旧类中声明的 Bean 方法被认为是“轻量级”的，包含类的主要目的不同，@Bean 方法在那里是一种奖励。 例如，服务组件可以通过每个适用组件类上的附加 @Bean 方法向容器公开管理视图。 在这种情况下，@Bean 方法是一种通用的工厂方法机制。

与完整的@Configuration 不同，lite @Bean 方法不能声明 bean 间的依赖关系。 相反，它们对包含组件的内部状态进行操作，并且可以选择对它们可能声明的参数进行操作。 因此，这样的 @Bean 方法不应调用其他 @Bean 方法。 每个这样的方法实际上只是特定 bean 引用的工厂方法，没有任何特殊的运行时语义。 这里的积极副作用是不需要在运行时应用 CGLIB 子类化，因此在类设计方面没有限制（即包含类可能是 final 等等）。

在常见情况下，@Bean 方法将在 @Configuration 类中声明，确保始终使用“完整”模式，因此跨方法引用被重定向到容器的生命周期管理。 这可以防止通过常规 Java 调用意外调用相同的 @Bean 方法，这有助于减少在“精简”模式下操作时难以追踪的细微错误。

@Bean 和 @Configuration 注释将在以下部分进行深入讨论。 然而，首先，我们介绍了使用基于 Java 的配置创建 spring 容器的各种方法。

### 使用 AnnotationConfigApplicationContext 实例化 Spring 容器

以下部分记录了 Spring 3.0 中引入的 Spring 的 AnnotationConfigApplicationContext。 这个通用的 ApplicationContext 实现不仅能够接受 @Configuration 类作为输入，还能够接受普通的 @Component 类和用 JSR-330 元数据注释的类。

当@Configuration 类作为输入提供时，@Configuration 类本身被注册为一个 bean 定义，并且该类中所有声明的 @Bean 方法也被注册为 bean 定义。

当提供@Component 和JSR-330 类时，它们被注册为bean 定义，并且假定在这些类中必要时使用DI 元数据，例如@Autowired 或@Inject。

#### 简单的构造

与在实例化 ClassPathXmlApplicationContext 时使用 Spring XML 文件作为输入的方式大致相同，您可以在实例化 AnnotationConfigApplicationContext 时使用 @Configuration 类作为输入。 这允许完全无 XML 使用 Spring 容器，如以下示例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

如前所述，AnnotationConfigApplicationContext 不仅限于使用 @Configuration 类。 任何 @Component 或 JSR-330 注释类都可以作为构造函数的输入提供，如以下示例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

前面的示例假设 MyServiceImpl、Dependency1 和 Dependency2 使用 Spring 依赖注入注解，例如 @Autowired。

#### 使用 register(Class<?>… ) 以编程方式构建容器

您可以使用无参数构造函数实例化 AnnotationConfigApplicationContext，然后使用 register() 方法对其进行配置。 这种方法在以编程方式构建 AnnotationConfigApplicationContext 时特别有用。 以下示例显示了如何执行此操作：

````java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.register(AppConfig.class, OtherConfig.class);
    ctx.register(AdditionalConfig.class);
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
````

#### 使用 scan(String… ) 启用组件扫描

要启用组件扫描，您可以按如下方式注释 @Configuration 类：

```java
@Configuration
@ComponentScan(basePackages = "com.acme") 
public class AppConfig  {
    // ...
}
```

在前面的示例中，扫描 com.acme 包以查找任何带有 @Component 注释的类，并将这些类注册为容器内的 Spring bean 定义。 AnnotationConfigApplicationContext 公开了 scan(String… ) 方法以允许相同的组件扫描功能，如以下示例所示：

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.scan("com.acme");
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
}
```

#### 使用 AnnotationConfigWebApplicationContext 支持 Web 应用程序

AnnotationConfigApplicationContext 的 WebApplicationContext 变体可用于 AnnotationConfigWebApplicationContext。 您可以在配置 Spring ContextLoaderListener servlet 侦听器、Spring MVC DispatcherServlet 等时使用此实现。 以下 web.xml 片段配置了一个典型的 Spring MVC Web 应用程序（注意 contextClass 上下文参数和 init-param 的使用）：

```xml
<web-app>
    <!-- Configure ContextLoaderListener to use AnnotationConfigWebApplicationContext
        instead of the default XmlWebApplicationContext -->
    <context-param>
        <param-name>contextClass</param-name>
        <param-value>
            org.springframework.web.context.support.AnnotationConfigWebApplicationContext
        </param-value>
    </context-param>

    <!-- Configuration locations must consist of one or more comma- or space-delimited
        fully-qualified @Configuration classes. Fully-qualified packages may also be
        specified for component-scanning -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>com.acme.AppConfig</param-value>
    </context-param>

    <!-- Bootstrap the root application context as usual using ContextLoaderListener -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- Declare a Spring MVC DispatcherServlet as usual -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- Configure DispatcherServlet to use AnnotationConfigWebApplicationContext
            instead of the default XmlWebApplicationContext -->
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>
                org.springframework.web.context.support.AnnotationConfigWebApplicationContext
            </param-value>
        </init-param>
        <!-- Again, config locations must consist of one or more comma- or space-delimited
            and fully-qualified @Configuration classes -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>com.acme.web.MvcConfig</param-value>
        </init-param>
    </servlet>

    <!-- map all requests for /app/* to the dispatcher servlet -->
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>
</web-app>
```

### 使用`@Bean`注解

要声明一个 bean，您可以使用 @Bean 注释来注释一个方法。 您可以使用此方法在指定为方法返回值的类型的 ApplicationContext 中注册 bean 定义。 默认情况下，bean 名称与方法名称相同。 以下示例显示了 @Bean 方法声明：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}
```

您还可以使用接口（或基类）返回类型声明 @Bean 方法，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

但是，这将高级类型预测的可见性限制为指定的接口类型 (TransferService)。 然后，只有在实例化受影响的单例 bean 后，容器才知道完整类型 (TransferServiceImpl)。 非惰性单例 bean 根据它们的声明顺序进行实例化，因此您可能会看到不同的类型匹配结果，具体取决于另一个组件何时尝试通过非声明类型进行匹配（例如 @Autowired TransferServiceImpl，它仅在 transferService bean 具有 被实例化）。

#### bean 依赖

@Bean-annotated 方法可以有任意数量的参数来描述构建该 bean 所需的依赖项。 例如，如果我们的 TransferService 需要 AccountRepository，我们可以使用方法参数具体化该依赖项，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}
```

解析机制与基于构造函数的依赖注入几乎相同

#### 接收生命周期回调

任何用@Bean 注释定义的类都支持常规生命周期回调，并且可以使用 JSR-250 中的 @PostConstruct 和 @PreDestroy 注释。 有关更多详细信息，请参阅 JSR-250 注释。

也完全支持常规的 Spring 生命周期回调。 如果 bean 实现了 InitializingBean、DisposableBean 或 Lifecycle，则容器会调用它们各自的方法。

也完全支持标准的 *Aware 接口集（例如 BeanFactoryAware、BeanNameAware、MessageSourceAware、ApplicationContextAware 等）。

@Bean 注解支持指定任意的初始化和销毁回调方法，很像 Spring XML 的 bean 元素上的 init-method 和 destroy-method 属性，如下例所示：

```java
public class BeanOne {

    public void init() {
        // initialization logic
    }
}

public class BeanTwo {

    public void cleanup() {
        // destruction logic
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "init")
    public BeanOne beanOne() {
        return new BeanOne();
    }

    @Bean(destroyMethod = "cleanup")
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

对于上述示例中的 BeanOne，在构造期间直接调用 init() 方法同样有效，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        BeanOne beanOne = new BeanOne();
        beanOne.init();
        return beanOne;
    }

    // ...
}
```

有时，提供 bean 的更详细的文本描述会很有帮助。 当 bean 被公开（可能通过 JMX）用于监视目的时，这可能特别有用。

要向 @Bean 添加描述，您可以使用 @Description 注释，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    @Description("Provides a basic example of a bean")
    public Thing thing() {
        return new Thing();
    }
}
```

### `@Configuration`

@Configuration 是一个类级别的注解，表明一个对象是 bean 定义的来源。 @Configuration 类通过@Bean 注释的方法声明bean。 对@Configuration 类上的@Bean 方法的调用也可用于定义bean 间的依赖关系。 有关一般介绍，请参阅基本概念：@Bean 和 @Configuration。

#### 注入bean间依赖

当 bean 相互依赖时，表达这种依赖就像让一个 bean 方法调用另一个方法一样简单，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        return new BeanOne(beanTwo());
    }

    @Bean
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

这种声明 bean 间依赖关系的方法只有在 @Configuration 类中声明了 @Bean 方法时才有效。 您不能使用普通的 @Component 类来声明 bean 间的依赖关系。

如前所述，查找方法注入是一项您应该很少使用的高级功能。 在单例范围的 bean 依赖于原型范围的 bean 的情况下，它很有用。 将 Java 用于这种类型的配置为实现这种模式提供了一种自然的方法。 下面的例子展示了如何使用查找方法注入：

```java
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

通过使用 Java 配置，您可以创建 CommandManager 的子类，其中抽象的 createCommand() 方法以查找新（原型）命令对象的方式被覆盖。 以下示例显示了如何执行此操作：

```java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
    AsyncCommand command = new AsyncCommand();
    // inject dependencies here as required
    return command;
}

@Bean
public CommandManager commandManager() {
    // return new anonymous implementation of CommandManager with createCommand()
    // overridden to return a new prototype Command object
    return new CommandManager() {
        protected Command createCommand() {
            return asyncCommand();
        }
    }
}
```

#### 关于基于 Java 的配置如何在内部工作的更多信息

考虑以下示例，它显示了一个被 @Bean 注释的方法被调用两次：

```java
@Configuration
public class AppConfig {

    @Bean
    public ClientService clientService1() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientService clientService2() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientDao clientDao() {
        return new ClientDaoImpl();
    }
}
```

clientDao() 已在 clientService1() 和 clientService2() 中调用一次。 由于此方法会创建一个 ClientDaoImpl 的新实例并返回它，因此您通常希望有两个实例（每个服务一个）。 这肯定会有问题：在 Spring 中，实例化的 bean 默认有一个单例作用域。 这就是神奇之处：所有@Configuration 类在启动时使用 CGLIB 进行子类化。 在子类中，子方法在调用父方法并创建新实例之前，首先检查容器中是否有任何缓存的（作用域）bean。

### 编写基于 Java 的配置

Spring 的基于 Java 的配置功能允许您编写注解，这可以降低配置的复杂性。

#### 使用@Import 注释

就像在 Spring XML 文件中使用 <import/> 元素来帮助模块化配置一样，@Import 注释允许从另一个配置类加载 @Bean 定义，如以下示例所示：

```java
@Configuration
public class ConfigA {

    @Bean
    public A a() {
        return new A();
    }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```

现在，无需在实例化上下文时同时指定 ConfigA.class 和 ConfigB.class，只需显式提供 ConfigB，如下例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

    // now both beans A and B will be available...
    A a = ctx.getBean(A.class);
    B b = ctx.getBean(B.class);
}
```

从 Spring Framework 4.2 开始，@Import 还支持对常规组件类的引用，类似于 AnnotationConfigApplicationContext.register 方法。 如果您想避免组件扫描，这将特别有用，通过使用一些配置类作为入口点来显式定义所有组件。

#### 注入对导入的 @Bean 定义的依赖

前面的示例有效但很简单。 在大多数实际场景中，bean 跨配置类相互依赖。 使用 XML 时，这不是问题，因为不涉及编译器，您可以声明 ref="someBean" 并信任 Spring 在容器初始化期间解决它。 使用@Configuration 类时，Java 编译器会对配置模型施加约束，因为对其他 bean 的引用必须是有效的 Java 语法。

幸运的是，解决这个问题很简单。 正如我们已经讨论过的，@Bean 方法可以有任意数量的参数来描述 bean 的依赖关系。 考虑以下更真实的场景，其中包含多个 @Configuration 类，每个类都依赖于其他类中声明的 bean：

```java
@Configuration
public class ServiceConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    @Bean
    public AccountRepository accountRepository(DataSource dataSource) {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

还有另一种方法可以达到相同的结果。 请记住，@Configuration 类最终只是容器中的另一个 bean：这意味着它们可以利用 @Autowired 和 @Value 注入以及与任何其他 bean 相同的其他功能。

确保您以这种方式注入的依赖项只是最简单的类型。 @Configuration 类在上下文的初始化过程中被处理得很早，并且强制以这种方式注入依赖项可能会导致意外的早期初始化。 尽可能使用基于参数的注入，如前面的示例所示。

另外，通过@Bean 定义 BeanPostProcessor 和 BeanFactoryPostProcessor 时要特别小心。 这些通常应该声明为静态@Bean 方法，而不是触发它们包含的配置类的实例化。 否则，@Autowired 和@Value 可能不适用于配置类本身，因为可以在 AutowiredAnnotationBeanPostProcessor 之前将其创建为 bean 实例。

以下示例显示了如何将一个 bean 自动装配到另一个 bean：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private AccountRepository accountRepository;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    private final DataSource dataSource;

    public RepositoryConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

在前面的场景中，使用 @Autowired 效果很好并提供了所需的模块化，但是确定自动装配的 bean 定义的确切位置仍然有些模棱两可。 例如，作为查看 ServiceConfig 的开发人员，您如何确切知道@Autowired AccountRepository bean 的声明位置？ 它在代码中没有明确，这可能就好了。 请记住，用于 Eclipse 的 Spring 工具提供的工具可以呈现显示所有内容如何连接的图形，这可能就是您所需要的。 此外，您的 Java IDE 可以轻松找到 AccountRepository 类型的所有声明和使用，并快速向您显示返回该类型的 @Bean 方法的位置。

如果这种歧义是不可接受的，并且您希望在 IDE 内从一个 @Configuration 类直接导航到另一个，请考虑自动装配配置类本身。 以下示例显示了如何执行此操作：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        // navigate 'through' the config class to the @Bean method!
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}
```

在前面的情况下，AccountRepository 的定义是完全明确的。 但是，ServiceConfig 现在与 RepositoryConfig 紧密耦合。 这就是权衡。 通过使用基于接口或基于抽象类的 @Configuration 类，可以在一定程度上缓解这种紧密耦合。 考虑以下示例：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}

@Configuration
public interface RepositoryConfig {

    @Bean
    AccountRepository accountRepository();
}

@Configuration
public class DefaultRepositoryConfig implements RepositoryConfig {

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(...);
    }
}

@Configuration
@Import({ServiceConfig.class, DefaultRepositoryConfig.class})  // import the concrete config!
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return DataSource
    }

}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

现在 ServiceConfig 与具体的 DefaultRepositoryConfig 松散耦合，内置的 IDE 工具仍然有用：您可以轻松获得 RepositoryConfig 实现的类型层次结构。 通过这种方式，导航@Configuration 类及其依赖项与导航基于接口的代码的通常过程没有什么不同。

#### 有条件地包含@Configuration 类或@Bean 方法

基于某些任意系统状态，有条件地启用或禁用完整的 @Configuration 类甚至单个 @Bean 方法通常很有用。 一个常见的例子是，只有在 Spring 环境中启用了特定配置文件时，才使用 @Profile 注释激活 bean（有关详细信息，请参阅 Bean 定义配置文件）。

@Profile 注解实际上是通过使用一个更灵活的注解@Conditional 来实现的。 @Conditional 注释指示在注册 @Bean 之前应该咨询的特定 org.springframework.context.annotation.Condition 实现。

Condition 接口的实现提供了一个matches(… ) 方法，该方法返回true 或false。 例如，以下清单显示了用于 @Profile 的实际 Condition 实现：

```java
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    // Read the @Profile annotation attributes
    MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
    if (attrs != null) {
        for (Object value : attrs.get("value")) {
            if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
                return true;
            }
        }
        return false;
    }
    return true;
}
```

See the [`@Conditional`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/context/annotation/Conditional.html) javadoc for more detail.

#### 结合 Java 和 XML 配置

Spring 的 @Configuration 类支持并不旨在 100% 完全替代 Spring XML。 一些工具，例如 Spring XML 命名空间，仍然是配置容器的理想方式。 在 XML 方便或必要的情况下，您有一个选择：使用例如 ClassPathXmlApplicationContext 以“以 XML 为中心”的方式实例化容器，或使用 AnnotationConfigApplicationContext 以“以 Java 为中心”的方式实例化容器 @ImportResource 注释以根据需要导入 XML。

##### @Configuration 类的以 XML 为中心的使用

最好从 XML 引导 Spring 容器并以特别方式包含 @Configuration 类。 例如，在使用 Spring XML 的大型现有代码库中，根据需要创建 @Configuration 类并从现有 XML 文件中包含它们会更容易。 在本节的后面，我们将介绍在这种“以 XML 为中心”的情况下使用 @Configuration 类的选项。

将@Configuration 类声明为普通的 Spring <bean/> 元素

请记住，@Configuration 类最终是容器中的 bean 定义。 在本系列示例中，我们创建了一个名为 AppConfig 的 @Configuration 类，并将其作为 <bean/> 定义包含在 system-test-config.xml 中。 因为<context:annotation-config/>开启了，容器识别@Configuration注解，正确处理AppConfig中声明的@Bean方法。

```java
@Configuration
public class AppConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }

    @Bean
    public TransferService transferService() {
        return new TransferService(accountRepository());
    }
}
```

```xml
<beans>
    <!-- enable processing of annotations such as @Autowired and @Configuration -->
    <context:annotation-config/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="com.acme.AppConfig"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

```java
public static void main(String[] args) {
    ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:/com/acme/system-test-config.xml");
    TransferService transferService = ctx.getBean(TransferService.class);
    // ...
}
```

使用 <context:component-scan/> 来获取@Configuration 类

因为@Configuration 是用@Component 元注释的，所以@Configuration 注释的类自动成为组件扫描的候选者。 使用与前面示例中描述的相同的场景，我们可以重新定义 system-test-config.xml 以利用组件扫描。 请注意，在这种情况下，我们不需要显式声明 <context:annotation-config/>，因为 <context:component-scan/> 启用了相同的功能。

```xml
<beans>
    <!-- picks up and registers AppConfig as a bean definition -->
    <context:component-scan base-package="com.acme"/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

@Configuration 以类为中心的 XML 与 @ImportResource 的使用

在@Configuration 类是配置容器的主要机制的应用程序中，仍然可能需要至少使用一些 XML。 在这些场景中，您可以使用 @ImportResource 并根据需要定义尽可能多的 XML。 这样做可以实现“以 Java 为中心”的方法来配置容器，并将 XML 保持在最低限度。 下面的示例（包括一个配置类、一个定义 bean 的 XML 文件、一个属性文件和主类）展示了如何使用 @ImportResource 注释来实现根据需要使用 XML 的“以 Java 为中心”的配置：

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}
```

properties-config.xml 

```xml
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    // ...
}
```

### 环境抽象

Environment 接口是集成在容器中的抽象，它对应用程序环境的两个关键方面进行建模：配置文件和属性。

配置文件是一个命名的、逻辑的 bean 定义组，仅当给定的配置文件处于活动状态时才向容器注册。 Bean 可以分配给配置文件，无论是在 XML 中定义还是使用注释。 与配置文件相关的环境对象的作用是确定哪些配置文件（如果有）当前是活动的，以及默认情况下哪些配置文件（如果有）应该是活动的。

属性在几乎所有应用程序中都扮演着重要的角色，并且可能来自各种来源：属性文件、JVM 系统属性、系统环境变量、JNDI、servlet 上下文参数、ad-hoc Properties 对象、Map 对象等等。 与属性相关的 Environment 对象的作用是为用户提供方便的服务接口，用于配置属性源并从中解析属性。

#### Bean 定义配置文件

Bean 定义配置文件在核心容器中提供了一种机制，允许在不同环境中注册不同的 Bean。 “环境”这个词对不同的用户可能有不同的含义，此功能可以帮助解决许多用例，包括：

* 在开发中使用内存数据源，而不是在 QA 或生产中从 JNDI 查找相同的数据源。
* 仅在将应用程序部署到性能环境中时才注册监控基础设施。
* 为客户 A 和客户 B 部署注册 bean 的自定义实现。

考虑需要数据源的实际应用程序中的第一个用例。 在测试环境中，配置可能类似于以下内容：

```java
@Bean
public DataSource dataSource() {
    return new EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.HSQL)
        .addScript("my-schema.sql")
        .addScript("my-test-data.sql")
        .build();
}
```

现在考虑如何将此应用程序部署到 QA 或生产环境中，假设应用程序的数据源已注册到生产应用程序服务器的 JNDI 目录。 我们的 dataSource bean 现在看起来像下面的清单：

```java
@Bean(destroyMethod="")
public DataSource dataSource() throws Exception {
    Context ctx = new InitialContext();
    return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
}
```

问题是如何根据当前环境在使用这两种变体之间进行切换。 随着时间的推移，Spring 用户设计了多种方法来完成这项工作，通常依赖于系统环境变量和包含 ${placeholder} 标记的 XML <import/> 语句的组合，这些标记根据值解析为正确的配置文件路径 一个环境变量。 Bean 定义配置文件是一个核心容器特性，它为这个问题提供了解决方案。

如果我们概括前面环境特定 bean 定义示例中显示的用例，我们最终需要在某些上下文中注册某些 bean 定义，而不是在其他上下文中。 你可以说你想在情况 A 中注册一个 bean 定义的特定配置文件，在情况 B 中注册一个不同的配置文件。我们首先更新我们的配置来反映这种需求。

当一个或多个指定的配置文件处于活动状态时，@Profile 批注让您可以指示组件有资格注册。 使用前面的示例，我们可以按如下方式重写 dataSource 配置：

````java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}
````

```java
@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

配置文件字符串可能包含一个简单的配置文件名称（例如，生产）或配置文件表达式。 配置文件表达式允许表达更复杂的配置文件逻辑（例如，生产和 us-east）。 配置文件表达式支持以下运算符：

- `!`: A logical “not” of the profile
- `&`: A logical “and” of the profiles
- `|`: A logical “or” of the profiles

您可以使用 @Profile 作为元注释来创建自定义组合注释。 以下示例定义了一个自定义 @Production 注释，您可以将其用作 @Profile("production") 的替代品：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

如果@Configuration 类用@Profile 标记，则所有与该类关联的@Bean 方法和@Import 注释都将被绕过，除非一个或多个指定的配置文件处于活动状态。 如果@Component 或@Configuration 类用@Profile({"p1", "p2"}) 标记，则除非已激活配置文件“p1”或“p2”，否则不会注册或处理该类。 如果给定的配置文件以 NOT 运算符 (!) 为前缀，则仅当配置文件未处于活动状态时才注册带注释的元素。 例如，给定@Profile({"p1", "!p2"})，如果配置文件“p1”处于活动状态或配置文件“p2”未处于活动状态，则会发生注册。

@Profile 也可以在方法级别声明以仅包含配置类的一个特定 bean（例如，对于特定 bean 的替代变体），如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Profile("development") 
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Profile("production") 
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

现在我们已经更新了我们的配置，我们仍然需要指示 Spring 哪个配置文件是活动的。 如果我们现在启动示例应用程序，我们会看到抛出 NoSuchBeanDefinitionException，因为容器找不到名为 dataSource 的 Spring bean。

可以通过多种方式激活配置文件，但最直接的是针对环境 API 以编程方式进行，该 API 可通过 ApplicationContext 获得。 以下示例显示了如何执行此操作：

```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```

此外，您还可以通过 spring.profiles.active 属性声明性地激活配置文件，该属性可以通过系统环境变量、JVM 系统属性、web.xml 中的 servlet 上下文参数指定，甚至作为 JNDI 中的条目（请参阅 PropertySource Abstraction ）。 在集成测试中，可以使用 spring-test 模块中的 @ActiveProfiles 注释来声明活动配置文件（请参阅带有环境配置文件的上下文配置）。

请注意，配置文件不是“非此即彼”的命题。 您可以一次激活多个配置文件。 以编程方式，您可以为 setActiveProfiles() 方法提供多个配置文件名称，该方法接受 String… varargs。 以下示例激活多个配置文件：

```java
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```

声明性地， spring.profiles.active 可以接受以逗号分隔的配置文件名称列表，如以下示例所示：

```
    -Dspring.profiles.active="profile1,profile2"
```

默认配置文件代表默认启用的配置文件。 考虑以下示例：

```java
@Configuration
@Profile("default")
public class DefaultDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .build();
    }
}
```

如果没有配置文件处于活动状态，则创建数据源。 您可以将其视为为一个或多个 bean 提供默认定义的一种方式。 如果启用了任何配置文件，则默认配置文件不适用。

您可以通过在 Environment 上使用 setDefaultProfiles() 或声明性地使用 spring.profiles.default 属性来更改默认配置文件的名称。

#### `PropertySource` 抽象

Spring 的 Environment 抽象在属性源的可配置层次结构上提供搜索操作。 考虑以下清单：

```java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```

在前面的片段中，我们看到了一种询问 Spring 是否为当前环境定义了 my-property 属性的高级方法。 为了回答这个问题，Environment 对象对一组 PropertySource 对象执行搜索。 PropertySource 是对任何键值对源的简单抽象，Spring 的 StandardEnvironment 配置了两个 PropertySource 对象 — 一个表示 JVM 系统属性集（System.getProperties()）和一个表示系统环境变量集（ System.getenv())。

这些默认属性源适用于 StandardEnvironment，用于独立应用程序。 StandardServletEnvironment 填充了额外的默认属性源，包括 servlet 配置和 servlet 上下文参数。 它可以选择启用 JndiPropertySource。 有关详细信息，请参阅 javadoc。

具体来说，当您使用 StandardEnvironment 时，如果运行时存在 my-property 系统属性或 my-property 环境变量，则对 env.containsProperty("my-property") 的调用将返回 true。

执行的搜索是分层的。 默认情况下，系统属性优先于环境变量。 因此，如果在调用 env.getProperty("my-property") 期间碰巧在两个地方都设置了 my-property 属性，则系统属性值“获胜”并返回。 请注意，属性值不会合并，而是被前面的条目完全覆盖。

对于常见的 StandardServletEnvironment，完整的层次结构如下，最高优先级的条目位于顶部：

ServletConfig 参数（如果适用 — 例如，在 DispatcherServlet 上下文的情况下）

ServletContext 参数（web.xml 上下文参数条目）

JNDI 环境变量（java:comp/env/ 条目）

JVM 系统属性（-D 命令行参数）

JVM系统环境（操作系统环境变量）

最重要的是，整个机制是可配置的。 也许您有想要集成到此搜索中的自定义属性源。 为此，请实现并实例化您自己的 PropertySource，并将其添加到当前环境的 PropertySource 集中。 以下示例显示了如何执行此操作：

```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```

在前面的代码中，MyPropertySource 已在搜索中以最高优先级添加。 如果它包含 my-property 属性，则会检测并返回该属性，以支持任何其他 PropertySource 中的任何 my-property 属性。 MutablePropertySources API 公开了许多允许精确操作属性源集的方法。

#### `@PropertySource`

@PropertySource 注解为将 PropertySource 添加到 Spring 的 Environment 提供了一种方便的声明机制。

给定一个名为 app.properties 的文件，其中包含键值对 testbean.name=myTestBean，以下 @Configuration 类使用 @PropertySource 的方式是调用 testBean.getName() 返回 myTestBean：

```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

@PropertySource 资源位置中存在的任何 ${… } 占位符都根据已针对环境注册的一组属性源进行解析，如以下示例所示：

```java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

假设 my.placeholder 存在于已注册的属性源之一（例如，系统属性或环境变量）中，占位符将解析为相应的值。 如果不是，则默认/路径用作默认值。 如果未指定默认值且无法解析属性，则会引发 IllegalArgumentException。

根据 Java 8 约定，@PropertySource 注释是可重复的。 但是，所有此类 @PropertySource 注释都需要在同一级别声明，或者直接在配置类上声明，或者作为同一自定义注释中的元注释。 不推荐混合直接注释和元注释，因为直接注释有效地覆盖了元注释。

#### 语句中的占位符解析

从历史上看，元素中占位符的值只能根据 JVM 系统属性或环境变量进行解析。 这已不再是这种情况。 因为 Environment 抽象集成在整个容器中，所以很容易通过它路由占位符的解析。 这意味着您可以按照自己喜欢的任何方式配置解析过程。 您可以更改搜索系统属性和环境变量的优先级或完全删除它们。 您还可以根据需要将自己的属性源添加到组合中。

具体来说，无论客户属性在哪里定义，只要它在环境中可用，以下语句都有效：

```xml
<beans>
    <import resource="com/bank/service/${customer}-config.xml"/>
</beans>
```

### 注册`LoadTimeWeaver`

Spring 使用 LoadTimeWeaver 在类加载到 Java 虚拟机 (JVM) 时对其进行动态转换。

要启用加载时编织，您可以将 @EnableLoadTimeWeaving 添加到您的 @Configuration 类之一，如以下示例所示：

```java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

或者，对于 XML 配置，您可以使用 context:load-time-weaver 元素：

```xml
<beans>
    <context:load-time-weaver/>
</beans>
```

为 ApplicationContext 配置后，该 ApplicationContext 中的任何 bean 都可以实现 LoadTimeWeaverAware，从而接收对加载时编织器实例的引用。 这在结合 Spring 的 JPA 支持时特别有用，其中 JPA 类转换可能需要加载时编织。 有关更多详细信息，请参阅 LocalContainerEntityManagerFactoryBean javadoc。 有关 AspectJ 加载时编织的更多信息，请参阅 Spring 框架中的 AspectJ 加载时编织。

[Load-time Weaving with AspectJ in the Spring Framework](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-aj-ltw).

[`LocalContainerEntityManagerFactoryBean`](https://docs.spring.io/spring-framework/docs/5.3.10/javadoc-api/org/springframework/orm/jpa/LocalContainerEntityManagerFactoryBean.html)

### `ApplicationContext`其他功能

正如章节介绍中所讨论的， org.springframework.beans.factory 包提供了管理和操作 bean 的基本功能，包括以编程方式。 org.springframework.context 包添加了 ApplicationContext 接口，它扩展了 BeanFactory 接口，此外还扩展了其他接口，以更面向应用程序框架的风格提供附加功能。 许多人以完全声明式的方式使用 ApplicationContext，甚至不是以编程方式创建它，而是依赖于诸如 ContextLoader 之类的支持类来自动实例化 ApplicationContext 作为 Java EE Web 应用程序正常启动过程的一部分。

#### 使用 MessageSource 进行国际化

ApplicationContext 接口扩展了一个名为 MessageSource 的接口，因此提供了国际化（“i18n”）功能。 Spring 还提供了 HierarchicalMessageSource 接口，可以分层解析消息。 这些接口一起提供了 Spring 影响消息解析的基础。 在这些接口上定义的方法包括：

* String getMessage(String code, Object[] args, String default, Locale loc)：用于从 MessageSource 检索消息的基本方法。 如果没有找到指定语言环境的消息，则使用默认消息。 使用标准库提供的 MessageFormat 功能，传入的任何参数都成为替换值。
* String getMessage(String code, Object[] args, Locale loc)：与前面的方法基本相同，但有一个区别：不能指定默认消息。 如果找不到消息，则抛出 NoSuchMessageException。
* String getMessage(MessageSourceResolvable resolvable, Locale locale)：上述方法中使用的所有属性也都封装在一个名为 MessageSourceResolvable 的类中，您可以在此方法中使用该类。

加载 ApplicationContext 时，它会自动搜索上下文中定义的 MessageSource bean。 bean 必须具有名称 messageSource。 如果找到这样的 bean，则对上述方法的所有调用都将委托给消息源。 如果未找到消息源，则 ApplicationContext 会尝试查找包含同名 bean 的父级。 如果是，则使用该 bean 作为 MessageSource。 如果 ApplicationContext 找不到任何消息源，则会实例化一个空的 DelegatingMessageSource，以便能够接受对上面定义的方法的调用。

Spring 提供了三个 MessageSource 实现，ResourceBundleMessageSource、ReloadableResourceBundleMessageSource 和 StaticMessageSource。 它们都实现 HierarchicalMessageSource 以进行嵌套消息传递。 StaticMessageSource 很少使用，但提供了向源添加消息的编程方式。 以下示例显示了 ResourceBundleMessageSource：

```xml
<beans>
    <bean id="messageSource"
            class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>
                <value>exceptions</value>
                <value>windows</value>
            </list>
        </property>
    </bean>
</beans>
```

该示例假设您在类路径中定义了三个名为 format、exceptions 和 windows 的资源包。 任何解析消息的请求都以 JDK 标准的方式通过 ResourceBundle 对象解析消息来处理。 出于示例的目的，假设上述两个资源包文件的内容如下：

```
    # in format.properties
    message=Alligators rock!
```

```
    # in exceptions.properties
    argument.required=The {0} argument is required.
```

下一个示例显示了一个运行 MessageSource 功能的程序。 请记住，所有 ApplicationContext 实现也是 MessageSource 实现，因此可以转换为 MessageSource 接口。

```java
public static void main(String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("message", null, "Default", Locale.ENGLISH);
    System.out.println(message);
}
```

总而言之，MessageSource 是在名为 beans.xml 的文件中定义的，该文件位于类路径的根目录中。 messageSource bean 定义通过其 basenames 属性引用了许多资源包。 在列表中传递给 basenames 属性的三个文件作为文件存在于类路径的根目录中，分别称为 format.properties、exceptions.properties 和 windows.properties。

下一个示例显示传递给消息查找的参数。 这些参数被转换为 String 对象并插入到查找消息中的占位符中。

```xml
<beans>

    <!-- this MessageSource is being used in a web application -->
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basename" value="exceptions"/>
    </bean>

    <!-- lets inject the above MessageSource into this POJO -->
    <bean id="example" class="com.something.Example">
        <property name="messages" ref="messageSource"/>
    </bean>

</beans>
```

```java
public class Example {

    private MessageSource messages;

    public void setMessages(MessageSource messages) {
        this.messages = messages;
    }

    public void execute() {
        String message = this.messages.getMessage("argument.required",
            new Object [] {"userDao"}, "Required", Locale.ENGLISH);
        System.out.println(message);
    }
}
```

调用 execute() 方法的结果输出如下：

```
The userDao argument is required.
```

关于国际化（“i18n”），Spring 的各种 MessageSource 实现遵循与标准 JDK ResourceBundle 相同的区域设置解析和回退规则。 简而言之，并继续之前定义的示例 messageSource，如果您想根据英国 (en-GB) 语言环境解析消息，您将分别创建名为 format_en_GB.properties、exceptions_en_GB.properties 和 windows_en_GB.properties 的文件。

通常，区域设置解析由应用程序的周围环境管理。 在以下示例中，手动指定解析（英国）消息所针对的区域设置：

```java
# in exceptions_en_GB.properties
argument.required=Ebagum lad, the ''{0}'' argument is required, I say, required.
```

```java
public static void main(final String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("argument.required",
        new Object [] {"userDao"}, "Required", Locale.UK);
    System.out.println(message);
}
```

运行上述程序的结果如下：

```
Ebagum lad, the 'userDao' argument is required, I say, required.
```

您还可以使用 MessageSourceAware 接口获取对已定义的任何 MessageSource 的引用。 创建和配置 bean 时，在实现 MessageSourceAware 接口的 ApplicationContext 中定义的任何 bean 都会注入应用程序上下文的 MessageSource。

因为 Spring 的 MessageSource 基于 Java 的 ResourceBundle，所以它不会合并具有相同基名的 bundle，而只会使用找到的第一个 bundle。 具有相同基本名称的后续消息包将被忽略。

作为 ResourceBundleMessageSource 的替代方案，Spring 提供了一个 ReloadableResourceBundleMessageSource 类。 此变体支持相同的包文件格式，但比基于标准 JDK 的 ResourceBundleMessageSource 实现更灵活。 特别是，它允许从任何 Spring 资源位置（不仅从类路径）读取文件，并支持包属性文件的热重载（同时在它们之间有效地缓存它们）。 有关详细信息，请参阅 ReloadableResourceBundleMessageSource javadoc。

#### 标准和自定义事件

ApplicationContext 中的事件处理是通过 ApplicationEvent 类和 ApplicationListener 接口提供的。 如果将实现 ApplicationListener 接口的 bean 部署到上下文中，则每次将 ApplicationEvent 发布到 ApplicationContext 时，都会通知该 bean。 本质上，这是标准的观察者设计模式。

| Event                        | Explanation                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| `ContextRefreshedEvent`      | Published when the `ApplicationContext` is initialized or refreshed (for example, by using the `refresh()` method on the `ConfigurableApplicationContext` interface). Here, “initialized” means that all beans are loaded, post-processor beans are detected and activated, singletons are pre-instantiated, and the `ApplicationContext` object is ready for use. As long as the context has not been closed, a refresh can be triggered multiple times, provided that the chosen `ApplicationContext` actually supports such “hot” refreshes. For example, `XmlWebApplicationContext` supports hot refreshes, but `GenericApplicationContext` does not. |
| `ContextStartedEvent`        | Published when the `ApplicationContext` is started by using the `start()` method on the `ConfigurableApplicationContext` interface. Here, “started” means that all `Lifecycle` beans receive an explicit start signal. Typically, this signal is used to restart beans after an explicit stop, but it may also be used to start components that have not been configured for autostart (for example, components that have not already started on initialization). |
| `ContextStoppedEvent`        | Published when the `ApplicationContext` is stopped by using the `stop()` method on the `ConfigurableApplicationContext` interface. Here, “stopped” means that all `Lifecycle` beans receive an explicit stop signal. A stopped context may be restarted through a `start()` call. |
| `ContextClosedEvent`         | Published when the `ApplicationContext` is being closed by using the `close()` method on the `ConfigurableApplicationContext` interface or via a JVM shutdown hook. Here, "closed" means that all singleton beans will be destroyed. Once the context is closed, it reaches its end of life and cannot be refreshed or restarted. |
| `RequestHandledEvent`        | A web-specific event telling all beans that an HTTP request has been serviced. This event is published after the request is complete. This event is only applicable to web applications that use Spring’s `DispatcherServlet`. |
| `ServletRequestHandledEvent` | A subclass of `RequestHandledEvent` that adds Servlet-specific context information. |

您还可以创建和发布自己的自定义事件。 以下示例显示了一个扩展 Spring 的 ApplicationEvent 基类的简单类：

```java
public class BlockedListEvent extends ApplicationEvent {

    private final String address;
    private final String content;

    public BlockedListEvent(Object source, String address, String content) {
        super(source);
        this.address = address;
        this.content = content;
    }

    // accessor and other methods...
}
```

要发布自定义 ApplicationEvent，请在 ApplicationEventPublisher 上调用 publishEvent() 方法。 通常，这是通过创建一个实现 ApplicationEventPublisherAware 的类并将其注册为 Spring bean 来完成的。 以下示例显示了这样一个类：

```java
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blockedList;
    private ApplicationEventPublisher publisher;

    public void setBlockedList(List<String> blockedList) {
        this.blockedList = blockedList;
    }

    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String content) {
        if (blockedList.contains(address)) {
            publisher.publishEvent(new BlockedListEvent(this, address, content));
            return;
        }
        // send email...
    }
}
```

配置时，Spring容器检测到EmailService实现了ApplicationEventPublisherAware，并自动调用setApplicationEventPublisher()。 实际上，传入的参数是Spring容器本身。 您正在通过其 ApplicationEventPublisher 接口与应用程序上下文进行交互。

要接收自定义 ApplicationEvent，您可以创建一个实现 ApplicationListener 的类并将其注册为 Spring bean。 以下示例显示了这样一个类：

```java
public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

请注意，ApplicationListener 通常使用自定义事件的类型（在前面的示例中为 BlockedListEvent）进行参数化。 这意味着 onApplicationEvent() 方法可以保持类型安全，避免任何向下转换的需要。 您可以根据需要注册任意数量的事件侦听器，但请注意，默认情况下，事件侦听器会同步接收事件。 这意味着 publishEvent() 方法会阻塞，直到所有侦听器都完成对事件的处理。 这种同步和单线程方法的一个优点是，当侦听器接收到事件时，如果事务上下文可用，它就会在发布者的事务上下文中运行。 如果需要另一种事件发布策略，请参阅 Spring 的 ApplicationEventMulticaster 接口和 SimpleApplicationEventMulticaster 实现的 javadoc 以了解配置选项。

以下示例显示了用于注册和配置上述每个类的 bean 定义：

```xml
<bean id="emailService" class="example.EmailService">
    <property name="blockedList">
        <list>
            <value>known.spammer@example.org</value>
            <value>known.hacker@example.org</value>
            <value>john.doe@example.org</value>
        </list>
    </property>
</bean>

<bean id="blockedListNotifier" class="example.BlockedListNotifier">
    <property name="notificationAddress" value="blockedlist@example.org"/>
</bean>
```

总而言之，当调用 emailService bean 的 sendEmail() 方法时，如果有任何电子邮件消息应该被阻止，则会发布一个 BlockedListEvent 类型的自定义事件。 BlockedListNotifier bean 注册为 ApplicationListener 并接收 BlockedListEvent，此时它可以通知适当的各方。

Spring 的事件机制是为同一应用程序上下文中 Spring bean 之间的简单通信而设计的。 然而，对于更复杂的企业集成需求，单独维护的 Spring Integration 项目为构建基于众所周知的 Spring 编程模型的轻量级、面向模式、事件驱动的体系结构提供了完整的支持。

#### 基于注解的事件监听器

您可以使用 @EventListener 注释在托管 bean 的任何方法上注册事件侦听器。 BlockedListNotifier 可以重写如下：



```java
public class BlockedListNotifier {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    @EventListener
    public void processBlockedListEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

方法签名再次声明了它侦听的事件类型，但这次使用了灵活的名称并且没有实现特定的侦听器接口。 只要实际事件类型在其实现层次结构中解析您的泛型参数，也可以通过泛型缩小事件类型。

如果您的方法应该侦听多个事件，或者您想定义它时根本不带参数，则还可以在注释本身上指定事件类型。 以下示例显示了如何执行此操作：

```java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
    // ...
}
```

还可以通过使用定义 SpEL 表达式的注释的条件属性添加额外的运行时过滤，该表达式应该匹配以实际调用特定事件的方法。以下示例显示了如何重写我们的通知程序以仅在事件的内容属性等于 my-event 时才被调用：

```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlockedListEvent(BlockedListEvent blEvent) {
    // notify appropriate parties via notificationAddress...
}
```

每个 SpEL 表达式都针对专用上下文进行评估。 下表列出了可用于上下文的项目，以便您可以将它们用于条件事件处理：

| Name            | Location           | Description                                                  | Example                                                      |
| :-------------- | :----------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Event           | root object        | The actual `ApplicationEvent`.                               | `#root.event` or `event`                                     |
| Arguments array | root object        | The arguments (as an object array) used to invoke the method. | `#root.args` or `args`; `args[0]` to access the first argument, etc. |
| *Argument name* | evaluation context | The name of any of the method arguments. If, for some reason, the names are not available (for example, because there is no debug information in the compiled byte code), individual arguments are also available using the `#a<#arg>` syntax where `<#arg>` stands for the argument index (starting from 0). | `#blEvent` or `#a0` (you can also use `#p0` or `#p<#arg>` parameter notation as |

请注意，#root.event 可让您访问底层事件，即使您的方法签名实际上是指已发布的任意对象。

如果您需要发布一个事件作为处理另一个事件的结果，您可以更改方法签名以返回应该发布的事件，如以下示例所示：

```java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```

handleBlockedListEvent() 方法为它处理的每个 BlockedListEvent 发布一个新的 ListUpdateEvent。 如果您需要发布多个事件，则可以返回一个集合或事件数组。

#### 异步监听

如果您希望特定的侦听器异步处理事件，则可以重用常规的 @Async 支持。 以下示例显示了如何执行此操作：

```java
@EventListener
@Async
public void processBlockedListEvent(BlockedListEvent event) {
    // BlockedListEvent is processed in a separate thread
}
```

使用异步事件时请注意以下限制：

如果异步事件侦听器抛出异常，则不会将其传播给调用者。 有关更多详细信息，请参阅 AsyncUncaughtExceptionHandler。

异步事件侦听器方法无法通过返回值来发布后续事件。 如果您需要发布另一个事件作为处理结果，请注入 ApplicationEventPublisher 以手动发布事件。

#### 监听器的顺序

如果您需要在另一个侦听器之前调用一个侦听器，则可以在方法声明中添加 @Order 注释，如下例所示：

```java
@EventListener
@Order(42)
public void processBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress...
}
```

#### 泛型事件

您还可以使用泛型进一步定义事件的结构。 考虑使用 EntityCreatedEvent<T> ，其中 T 是创建的实际实体的类型。 例如，您可以创建以下侦听器定义以仅接收 Person 的 EntityCreatedEvent：

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    // ...
}
```

由于类型擦除，这仅在被触发的事件解析事件侦听器过滤的通用参数时才有效（即类 PersonCreatedEvent extends EntityCreatedEvent<Person> { ... }）。

在某些情况下，如果所有事件都遵循相同的结构（如前面示例中的事件应该是这种情况），这可能会变得非常乏味。 在这种情况下，您可以实现 ResolvableTypeProvider 来指导框架超出运行时环境提供的范围。 以下事件显示了如何执行此操作：

```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

    public EntityCreatedEvent(T entity) {
        super(entity);
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
    }

```

### 便捷访问底层资源

为了最佳地使用和理解应用程序上下文，您应该熟悉 Spring 的 Resource 抽象，如参考资料中所述。

应用程序上下文是一个 ResourceLoader，可用于加载 Resource 对象。 资源本质上是 JDK java.net.URL 类的功能更丰富的版本。 事实上，Resource 的实现在适当的地方包装了一个 java.net.URL 的实例。 Resource 可以以透明的方式从几乎任何位置获取低级资源，包括从类路径、文件系统位置、用标准 URL 描述的任何位置以及其他一些变体。 如果资源位置字符串是一个没有任何特殊前缀的简单路径，那么这些资源的来源是特定的并且适合实际的应用程序上下文类型。

您可以将部署到应用程序上下文中的 bean 配置为实现特殊的回调接口 ResourceLoaderAware，以在初始化时自动回调应用程序上下文本身作为 ResourceLoader 传入。 您还可以公开 Resource 类型的属性，用于访问静态资源。 它们像任何其他属性一样被注入其中。 您可以将这些 Resource 属性指定为简单的 String 路径，并在部署 bean 时依赖于从这些文本字符串到实际 Resource 对象的自动转换。

提供给 ApplicationContext 构造函数的位置路径或路径实际上是资源字符串，并且以简单的形式根据特定的上下文实现进行适当处理。 例如 ClassPathXmlApplicationContext 将简单的位置路径视为类路径位置。 您还可以使用带有特殊前缀的位置路径（资源字符串）来强制从类路径或 URL 加载定义，而不管实际的上下文类型。

### 应用程序启动跟踪

ApplicationContext 管理 Spring 应用程序的生命周期并围绕组件提供丰富的编程模型。 因此，复杂的应用程序可能具有同样复杂的组件图和启动阶段。

使用特定指标跟踪应用程序启动步骤有助于了解启动阶段的时间花在何处，但也可以用作更好地了解整个上下文生命周期的方法。

AbstractApplicationContext（及其子类）使用 ApplicationStartup 进行检测，该 ApplicationStartup 收集有关各个启动阶段的 StartupStep 数据：

* 应用程序上下文生命周期（基本包扫描、配置类管理）

* bean 生命周期（实例化、智能初始化、后处理）

* 应用事件处理

这是 AnnotationConfigApplicationContext 中的检测示例：

```java
// create a startup step and start recording
StartupStep scanPackages = this.getApplicationStartup().start("spring.context.base-packages.scan");
// add tagging information to the current step
scanPackages.tag("packages", () -> Arrays.toString(basePackages));
// perform the actual phase we're instrumenting
this.scanner.scan(basePackages);
// end the current step
scanPackages.end();
```

应用程序上下文已经通过多个步骤进行了检测。 记录后，可以使用特定工具收集、显示和分析这些启动步骤。 有关现有启动步骤的完整列表，您可以查看专用附录部分。

默认的 ApplicationStartup 实现是无操作变体，以实现最小的开销。 这意味着默认情况下不会在应用程序启动期间收集任何指标。 Spring Framework 附带了一个使用 Java Flight Recorder 跟踪启动步骤的实现：FlightRecorderApplicationStartup。 要使用此变体，您必须在创建后立即将其实例配置到 ApplicationContext。

如果开发人员提供他们自己的 AbstractApplicationContext 子类，或者如果他们希望收集更精确的数据，也可以使用 ApplicationStartup 基础结构。

要开始收集自定义的 StartupStep，组件可以直接从应用程序上下文中获取 ApplicationStartup 实例，使其组件实现 ApplicationStartupAware，或者在任何注入点请求 ApplicationStartup 类型。

### Web 应用程序的便捷 ApplicationContext 实例化

例如，您可以使用 ContextLoader 以声明方式创建 ApplicationContext 实例。 当然，您也可以使用 ApplicationContext 实现之一以编程方式创建 ApplicationContext 实例。

您可以使用 ContextLoaderListener 注册 ApplicationContext，如以下示例所示：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

侦听器检查 contextConfigLocation 参数。 如果该参数不存在，则侦听器使用 /WEB-INF/applicationContext.xml 作为默认值。 当参数确实存在时，侦听器使用预定义的分隔符（逗号、分号和空格）分隔字符串，并将这些值用作搜索应用程序上下文的位置。 也支持 Ant 风格的路径模式。 示例是 /WEB-INF/*Context.xml（对于名称以 Context.xml 结尾并驻留在 WEB-INF 目录中的所有文件）和 /WEB-INF/**/*Context.xml（对于所有此类 WEB-INF 的任何子目录中的文件）。

### 将 Spring ApplicationContext 部署为 Java EE RAR 文件

可以将 Spring ApplicationContext 部署为 RAR 文件，将上下文及其所有必需的 bean 类和库 JAR 封装在 Java EE RAR 部署单元中。 这相当于引导独立的 ApplicationContext（仅托管在 Java EE 环境中）能够访问 Java EE 服务器设施。 RAR 部署是部署无头 WAR 文件场景的更自然的替代方案 — 实际上，一个没有任何 HTTP 入口点的 WAR 文件仅用于在 Java EE 环境中引导 Spring ApplicationContext。

RAR 部署非常适合不需要 HTTP 入口点而仅包含消息端点和计划作业的应用程序上下文。 此类上下文中的 Bean 可以使用应用程序服务器资源，例如 JTA 事务管理器和 JNDI 绑定的 JDBC 数据源实例和 JMS ConnectionFactory 实例，还可以注册到平台的 JMX 服务器 — 全部通过 Spring 的标准事务管理以及 JNDI 和 JMX 支持设施。 应用程序组件还可以通过 Spring 的 TaskExecutor 抽象与应用程序服务器的 JCA WorkManager 交互。

有关 RAR 部署中涉及的配置详细信息，请参阅 SpringContextResourceAdapter 类的 javadoc。

将 Spring ApplicationContext 简单部署为 Java EE RAR 文件：

1. 将所有应用程序类打包成一个 RAR 文件（这是一个具有不同文件扩展名的标准 JAR 文件）。

2. 将所有需要的库 JAR 添加到 RAR 存档的根目录中。

3. 添加 META-INF/ra.xml 部署描述符（如 SpringContextResourceAdapter 的 javadoc 中所示）和相应的 Spring XML bean 定义文件（通常是 META-INF/applicationContext.xml）。

4. 将生成的 RAR 文件放入应用程序服务器的部署目录中。

这种 RAR 部署单元通常是独立的。 它们不会将组件暴露给外部世界，甚至不会暴露给同一应用程序的其他模块。 与基于 RAR 的 ApplicationContext 的交互通常通过它与其他模块共享的 JMS 目标发生。 例如，基于 RAR 的 ApplicationContext 还可以调度一些作业或对文件系统中的新文件做出反应（或类似的）。 如果它需要允许来自外部的同步访问，它可以（例如）导出 RMI 端点，这些端点可能被同一台机器上的其他应用程序模块使用。

### `BeanFactory`

BeanFactory API 为 Spring 的 IoC 功能提供了底层基础。 它的特定契约主要用于与 Spring 的其他部分和相关第三方框架的集成，它的 DefaultListableBeanFactory 实现是更高级别的 GenericApplicationContext 容器中的关键委托。

BeanFactory 和相关接口（如 BeanFactoryAware、InitializingBean、DisposableBean）是其他框架组件的重要集成点。 由于不需要任何注释甚至反射，它们允许容器与其组件之间非常有效的交互。 应用程序级 bean 可能使用相同的回调接口，但通常更喜欢声明式依赖注入，无论是通过注释还是通过编程配置。

请注意，核心 BeanFactory API 级别及其 DefaultListableBeanFactory 实现不会对要使用的配置格式或任何组件注释进行假设。 所有这些风格都通过扩展（例如 XmlBeanDefinitionReader 和 AutowiredAnnotationBeanPostProcessor）进入，并在共享 BeanDefinition 对象上作为核心元数据表示进行操作。 这就是使 Spring 的容器如此灵活和可扩展的本质。

#### `BeanFactory` or `ApplicationContext`

本节解释 BeanFactory 和 ApplicationContext 容器级别之间的差异以及对引导的影响。

您应该使用 ApplicationContext ，除非您有充分的理由不这样做，将 GenericApplicationContext 及其子类 AnnotationConfigApplicationContext 作为自定义引导的常见实现。 这些是用于所有常见目的的 Spring 核心容器的主要入口点：加载配置文件、触发类路径扫描、以编程方式注册 bean 定义和带注释的类，以及（从 5.0 开始）注册功能 bean 定义。

由于 ApplicationContext 包含 BeanFactory 的所有功能，因此通常建议使用普通 BeanFactory，除非需要完全控制 bean 处理的场景。 在 ApplicationContext（例如 GenericApplicationContext 实现）中，根据约定（即通过 bean 名称或 bean 类型 — 特别是后处理器）检测几种 bean，而普通的 DefaultListableBeanFactory 与任何特殊 bean 无关。

对于许多扩展的容器功能，例如注释处理和 AOP 代理，BeanPostProcessor 扩展点是必不可少的。 如果您仅使用普通的 DefaultListableBeanFactory，则默认情况下不会检测和激活此类后处理器。 这种情况可能会令人困惑，因为您的 bean 配置实际上没有任何问题。 相反，在这种情况下，容器需要通过额外的设置来完全引导。

下表列出了 BeanFactory 和 ApplicationContext 接口和实现提供的功能。

| Feature                                                      | `BeanFactory` | `ApplicationContext` |
| :----------------------------------------------------------- | :------------ | :------------------- |
| Bean instantiation/wiring                                    | Yes           | Yes                  |
| Integrated lifecycle management                              | No            | Yes                  |
| Automatic `BeanPostProcessor` registration                   | No            | Yes                  |
| Automatic `BeanFactoryPostProcessor` registration            | No            | Yes                  |
| Convenient `MessageSource` access (for internationalization) | No            | Yes                  |
| Built-in `ApplicationEvent` publication mechanism            | No            | Yes                  |

要使用 DefaultListableBeanFactory 显式注册 bean 后处理器，您需要以编程方式调用 addBeanPostProcessor，如以下示例所示：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// populate the factory with bean definitions

// now register any needed BeanPostProcessor instances
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// now start using the factory
```

要将 BeanFactoryPostProcessor 应用于普通的 DefaultListableBeanFactory，您需要调用其 postProcessBeanFactory 方法，如下例所示：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// bring in some property values from a Properties file
PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// now actually do the replacement
cfg.postProcessBeanFactory(factory);
```

在这两种情况下，显式注册步骤都不方便，这就是为什么在 Spring 支持的应用程序中，各种 ApplicationContext 变体比普通的 DefaultListableBeanFactory 更受欢迎，尤其是在典型企业设置中依赖 BeanFactoryPostProcessor 和 BeanPostProcessor 实例来扩展容器功能时。

AnnotationConfigApplicationContext 注册了所有常见的注释后处理器，并且可以通过配置注释（例如@EnableTransactionManagement）在幕后引入额外的处理器。 在 Spring 基于注解的配置模型的抽象层，bean 后处理器的概念变成了一个纯粹的容器内部细节。