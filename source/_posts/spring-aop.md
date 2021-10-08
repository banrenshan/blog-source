# AOP概念

* Aspect：跨越多个类的关注点的模块化。`aspect` 由 `pointcount` 和 `advice` 组成, 它既包含了横切逻辑的定义, 也包括了连接点的定义. Spring AOP就是负责实施切面的框架, 它将切面所定义的横切逻辑织入到切面所指定的连接点中. AOP的工作重心在于如何将增强织入目标对象的连接点上, 这里包含两个工作:

  1. 如何通过 pointcut 和 advice 定位到特定的 joinpoint 上
  2. 如何在 advice 中编写切面代码.

  **可以简单地认为, 使用 @Aspect 注解的类就是切面.**

* Join point（连接点）：程序执行过程中的一个点，例如方法的执行或异常的处理。 在 Spring AOP 中，一个连接点总是代表一个方法的执行。

* Advice：在特定连接点采取的行动。

* Pointcut：匹配连接点的谓词。 Advice 与Pointcut表达式相关联，并在与Pointcut匹配的任何连接点处运行（例如，执行具有特定名称的方法）。 由Pointcut表达式匹配的连接点的概念是 AOP 的核心，Spring 默认使用 AspectJ 切入点表达式语言。

* Introduction：为一个类型添加额外的方法或字段. Spring AOP 允许我们为 `目标对象` 引入新的接口(和对应的实现). 例如我们可以使用 introduction 来为一个 bean 实现 IsModified 接口, 并以此来简化 caching 的实现.

* Target object：织入 advice 的目标对象. 目标对象也被称为 `advised object`.`因为 Spring AOP 使用运行时代理的方式来实现 aspect, 因此 adviced object 总是一个代理对象(proxied object)`注意, adviced object 指的不是原来的类, 而是织入 advice 后所产生的代理类.

* AOP proxy：一个类被 AOP 织入 advice, 就会产生一个结果类, 它是融合了原类和增强逻辑的代理类. 在 Spring AOP 中, 一个 AOP 代理是一个 JDK 动态代理对象或 CGLIB 代理对象.

* Weaving：将 aspect 和其他对象连接起来, 并创建 adviced object 的过程. 根据不同的实现技术, AOP织入有三种方式:

  - 编译器织入, 这要求有特殊的Java编译器.
  - 类装载期织入, 这需要有特殊的类装载器.
  - 动态代理织入, 在运行期为目标类添加增强(Advice)生成子类的方式. Spring 采用动态代理织入, 而AspectJ采用编译器织入和类装载期织入.

  事务来说明这几个概念？？？

## advice 的类型

- before advice, 在 join point 前被执行的 advice. 虽然 before advice 是在 join point 前被执行, 但是它并不能够阻止 join point 的执行, 除非发生了异常(即我们在 before advice 代码中, 不能人为地决定是否继续执行 join point 中的代码)
- after return advice, 在一个 join point 正常返回后执行的 advice
- after throwing advice, 当一个 join point 抛出异常后执行的 advice
- after(final) advice, 无论一个 join point 是正常退出还是发生了异常, 都会被执行的 advice.
- around advice, 在 join point 前和 joint point 退出后都执行的 advice. 这个是最常用的 advice.

## 代理方式

Spring AOP 默认为 AOP 代理使用标准的 JDK 动态代理。 被代理的类必须实现某个接口。

Spring AOP 也可以使用 CGLIB 代理。 不需要被代理类实现接口。 默认情况下，如果业务对象未实现接口，则使用 CGLIB。

可以使用注解或xml的方式配置切面。

# @AspectJ方式

## 开启@AspectJ支持

引入aspectjweaver.jar类库，开启@EnableAspectJAutoProxy注解：

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

xml方式：

```xml
<aop:aspectj-autoproxy/>
```

## 声明切面

启用@AspectJ 支持后，Spring 会自动检测@AspectJ 注解的类：

```java
package org.xyz;
import org.aspectj.lang.annotation.Aspect;

@Aspect
public class NotVeryUsefulAspect {

}
```

Aspects （用@Aspect 注释的类）可以有方法和字段，与任何其他类相同。 它们还可以包含pointcut, advice 和 introduction。

## 声明切点

切点确定感兴趣的连接点，从而使我们能够控制通知何时运行。 Spring AOP 仅支持 Spring bean 的方法执行连接点，因此您可以将切入点视为匹配 Spring bean 上方法的执行。 一个切入点声明有两个部分：完整的方法签名和切入点表达式，它确定我们对哪些方法执行感兴趣。在 AOP 的@AspectJ 注释样式中，一个切入点签名由常规方法定义提供 ，切入点表达式使用@Pointcut注解表示（作为切入点签名的方法必须有一个void返回类型）。

一个例子可能有助于明确切入点签名和切入点表达式之间的区别。 以下示例定义了一个名为 anyOldTransfer 的切入点，该切入点与名为 transfer 的任何方法的执行相匹配：

```java
@Pointcut("execution(* transfer(..))") // the pointcut expression
private void anyOldTransfer() {} // the pointcut signature
```

支持的切入点指示符:

* execution: 用于匹配方法执行连接点。 例如 "execution(* hello(..))" 表示匹配所有目标类中的 hello() 方法. 这个是最基本的 pointcut 标志符.
* within:匹配特定包下的所有 join point, 例如 `within(com.xys.*)` 表示 com.xys 包中的所有连接点, 即包中的所有类的所有方法. 而 `within(com.xys.service.*Service)` 表示在 com.xys.service 包中所有以 Service 结尾的类的所有的连接点.
* this: 限制匹配连接点（使用 Spring AOP 时的方法执行），其中 bean 引用（Spring AOP 代理）是给定类型的实例。
* target: 限制匹配连接点（使用 Spring AOP 时的方法执行），其中目标对象（被代理的应用程序对象）是给定类型的实例。
* args：限制匹配连接点（使用 Spring AOP 时方法的执行），其中参数是给定类型的实例。
* @target：限制匹配连接点（使用 Spring AOP 时的方法执行），其中执行对象的类具有给定类型的注释。
* @args：限制匹配连接点（使用 Spring AOP 时的方法执行），其中传递的实际参数的运行时类型具有给定类型的注释。
* @within：限制匹配连接点，within具有给定的注解
* @annotation：限制匹配连接点，匹配由指定注解所标注的方法

> 由于 Spring 的 AOP 框架基于代理的特性，根据定义，目标对象内的调用不会被拦截（即直接调用）。对于 JDK 代理，只能拦截代理上的公共接口方法。使用 CGLIB，可以拦截代理上的公共和受保护方法
>

您可以使用 `&&`、`||` 和 `！`来组合切入点表达式 。 您还可以按名称引用切入点表达式。 以下示例显示了三个切入点表达式：

```java
@Pointcut("execution(public * *(..))")
private void anyPublicOperation() {} 

@Pointcut("within(com.xyz.myapp.trading..*)")
private void inTrading() {} 

@Pointcut("anyPublicOperation() && inTrading()")
private void tradingOperation() {} 
```

* 匹配所有的public方法
* 匹配 trading模块下的所有方法

举例：

```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern)
                throws-pattern?)
```

```
// 匹配指定包中的所有的方法
execution(* com.xys.service.*(..))

// 匹配当前包中的指定类的所有方法
execution(* UserService.*(..))

// 匹配指定包中的所有 public 方法
execution(public * com.xys.service.*(..))

// 匹配指定包中的所有 public 方法, 并且返回值是 int 类型的方法
execution(public int com.xys.service.*(..))

// 匹配指定包中的所有 public 方法, 并且第一个参数是 String, 返回值是 int 类型的方法
execution(public int com.xys.service.*(String name, ..))
```

```sh
// 匹配指定包中的所有的方法, 但不包括子包
within(com.xys.service.*)

// 匹配指定包中的所有的方法, 包括子包
within(com.xys.service..*)

// 匹配当前包中的指定类中的方法
within(UserService)

// 匹配一个接口的所有实现类中的实现的方法
within(UserDao+)
```

```sh
// 目标对象实现 AccountService 接口的任何连接点（仅在 Spring AOP 中执行方法）：
target(com.xyz.service.AccountService)

//任何连接点（仅在 Spring AOP 中的方法执行）接受单个参数并且在运行时传递的参数是可序列化的：
args(java.io.Serializable)

//目标对象具有 @Transactional 注释的任何连接点（仅在 Spring AOP 中执行方法）：
@target(org.springframework.transaction.annotation.Transactional)

// 任何连接点（方法仅在 Spring AOP 中执行），其中目标对象的声明类型具有 @Transactional 注释：
@within(org.springframework.transaction.annotation.Transactional)

//任何连接点（方法仅在 Spring AOP 中执行），其中执行方法具有 @Transactional 注释：
@annotation(org.springframework.transaction.annotation.Transactional)
```

## 声明通知

前置通知：

```java
@Aspect
public class BeforeExample {

    @Before("com.xyz.myapp.CommonPointcuts.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }
}
```

如果我们就地使用切入点表达式，我们可以将前面的示例重写为以下示例：

```java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class BeforeExample {

    @Before("execution(* com.xyz.myapp.dao.*.*(..))")
    public void doAccessCheck() {
        // ...
    }
}
```

return advice:

```java
@Aspect
public class AfterReturningExample {

    @AfterReturning("com.xyz.myapp.CommonPointcuts.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }
}
```

有时，您需要在通知正文中访问返回的实际值。 如以下示例所示：

```java
@Aspect
public class AfterReturningExample {

    @AfterReturning(
        pointcut="com.xyz.myapp.CommonPointcuts.dataAccessOperation()",
        returning="retVal")
    public void doAccessCheck(Object retVal) {
        // ...
    }
}
```

返回属性中使用的名称必须与通知方法中的参数名称相对应。 当方法执行返回时，返回值作为相应的参数值传递给通知方法。

Throwing Advice:

```java
@Aspect
public class AfterThrowingExample {

    @AfterThrowing("com.xyz.myapp.CommonPointcuts.dataAccessOperation()")
    public void doRecoveryActions() {
        // ...
    }
}
```

如果要访问异常信息：

```java
@Aspect
public class AfterThrowingExample {

    @AfterThrowing(
        pointcut="com.xyz.myapp.CommonPointcuts.dataAccessOperation()",
        throwing="ex")
    public void doRecoveryActions(DataAccessException ex) {
        // ...
    }
}
```

throwing 属性中使用的名称必须与通知方法中的参数名称相对应。 当方法执行通过抛出异常退出时，异常将作为相应的参数值传递给通知方法。 throwing 子句还将匹配仅限于那些抛出指定类型异常（在本例中为 DataAccessException）的方法执行。

Finally Advice:

```java
@Aspect
public class AfterFinallyExample {

    @After("com.xyz.myapp.CommonPointcuts.dataAccessOperation()")
    public void doReleaseLock() {
        // ...
    }
}
```

环绕通知：

环绕通知是通过使用@Around 注释来声明的。 方法的第一个参数必须是 ProceedingJoinPoint 类型。 在通知正文中，对 ProceedingJoinPoint 调用proceed() 会导致底层方法运行。 proceed 方法也可以传入一个 Object[]。 数组中的值在方法执行时用作方法执行的参数。

```java
@Aspect
public class AroundExample {

    @Around("com.xyz.myapp.CommonPointcuts.businessService()")
    public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
        // start stopwatch
        Object retVal = pjp.proceed();
        // stop stopwatch
        return retVal;
    }
}
```

环绕通知返回的值是方法调用者看到的返回值。 例如，一个简单的缓存切面可以从缓存返回一个值，如果它有一个值，如果没有，则调用proceed()。 请注意，在 around 建议的主体内，proceed 可能会被调用一次、多次或根本不调用。 所有这些都是合法的。

任何通知方法都可以声明一个 org.aspectj.lang.JoinPoint 类型的参数作为它的第一个参数（注意，环绕通知需要声明一个 ProceedingJoinPoint 类型的第一个参数，它是 JoinPoint 的子类。JoinPoint 接口提供了一些有用的方法：

* getArgs()：返回方法参数
* getThis()：返回代理对象
* getTarget()：返回目标对象
* getSignature()：返回方法签名
* toString()：打印方法信息

在advice获取方法参数：

```java
@Before("com.xyz.myapp.CommonPointcuts.dataAccessOperation() && args(account,..)")
public void validateAccount(Account account) {
    // ...
}
```

切入点表达式的 args(account,..) 部分有两个目的。 首先，它将匹配限制为仅那些方法执行，其中该方法至少接受一个参数，并且传递给该参数的参数是 Account 的一个实例。 其次，它通过 account 参数使实际的 Account 对象可用于Advice。

另一种写法是声明一个切入点，当它匹配一个连接点时“提供”Account 对象值，然后从通知中引用命名的切入点。 这将如下所示：

```java
@Pointcut("com.xyz.myapp.CommonPointcuts.dataAccessOperation() && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
    // ...
}
```

审计相关绑定：

```java
@Before("com.xyz.lib.Pointcuts.anyPublicMethod() && @annotation(auditable)")
public void audit(Auditable auditable) {
    AuditCode code = auditable.value();
    // ...
}
```

Spring AOP 可以处理类声明和方法参数中使用的泛型。 假设你有一个像下面这样的泛型类型：

```java
public interface Sample<T> {
    void sampleGenericMethod(T param);
    void sampleGenericCollectionMethod(Collection<T> param);
}
```

```java
@Before("execution(* ..Sample+.sampleGenericMethod(*)) && args(param)")
public void beforeSampleMethod(MyType param) {
    // Advice implementation
}
```

### 通知的顺序

当多个通知都想在同一个连接点运行时会发生什么？ Spring AOP 遵循与 AspectJ 相同的优先级规则来确定通知执行的顺序。 最高优先级的建议首先在“in the way in”运行（因此，给定两条 before 建议，具有最高优先级的建议最先运行）。 从连接点“出路”，最高优先级通知最后运行（因此，给定两条后通知，具有最高优先级的将第二运行）。

当在不同切面定义的两条通知都需要在同一个连接点运行时，除非您另外指定，否则执行顺序是未定义的。 您可以通过指定优先级来控制执行顺序。 在方面类中实现 org.springframework.core.Ordered 接口或使用 @Order 注释对其进行注释。 给定两个方面，从 Ordered.getOrder() 返回较低值的方面（或注释值）具有更高的优先级。

## Introductions

使被通知的对象实现给定的接口，并代表这些对象提供该接口的实现。

您可以使用@DeclareParents 注释进行introduction 。 此注释用于声明匹配类型具有新的父级。 例如：

```java
@Aspect
public class UsageTracking {

    @DeclareParents(value="com.xzy.myapp.service.*+", defaultImpl=DefaultUsageTracked.class)
    public static UsageTracked mixin;

    @Before("com.xyz.myapp.CommonPointcuts.businessService() && this(usageTracked)")
    public void recordUsage(UsageTracked usageTracked) {
        usageTracked.incrementUseCount();
    }

}
```

要实现的接口由注释字段的类型决定。 @DeclareParents 注释的 value 属性是一个 AspectJ 类型模式。 任何匹配类型的 bean 都实现 UsageTracked 接口。 请注意，在示例的 before 通知中，服务 bean 可以直接用作 UsageTracked 接口的实现。 如果以编程方式访问 bean，您将编写以下内容：

```java
UsageTracked usageTracked = (UsageTracked) context.getBean("myService");
```

## 代理模式

Spring AOP 使用 JDK 动态代理或 CGLIB 为给定的目标对象创建代理。 JDK 中内置了 JDK 动态代理，而 CGLIB 是一个常见的开源类定义库（重新打包到 spring-core 中）。

如果要代理的目标对象至少实现了一个接口，则使用 JDK 动态代理。 目标类型实现的所有接口都被代理。 如果目标对象没有实现任何接口，则创建一个 CGLIB 代理。

如果您想强制使用 CGLIB 代理（例如，代理为目标对象定义的每个方法，而不仅仅是由其接口实现的方法），您可以这样做。 但是，您应该考虑以下问题：

* 使用 CGLIB，不能建议 final 方法，因为它们不能在运行时生成的子类中被覆盖。

* 从 Spring 4.0 开始，代理对象的构造函数不再被调用两次，因为 CGLIB 代理实例是通过 Objenesis 创建的。 仅当您的 JVM 不允许构造函数绕过时，您可能会看到来自 Spring 的 AOP 支持的双重调用和相应的调试日志条目。

要强制使用 CGLIB 代理，请将 <aop:config> 元素的 proxy-target-class 属性值设置为 true，如下所示：

```java
<aop:config proxy-target-class="true">
    <!-- other beans defined here... -->
</aop:config>
```

要在使用 @AspectJ 自动代理支持时强制使用 CGLIB 代理，请将 <aop:aspectj-autoproxy> 元素的 proxy-target-class 属性设置为 true，如下所示：

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

