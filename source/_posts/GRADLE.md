# 构建环境

了解 Gradle 分三个阶段评估和执行构建脚本很重要：

1. Initialization：为构建设置环境并确定哪些项目将参与其中。
2. Configuration：构建和配置任务图，然后根据用户想要运行的任务确定需要运行哪些任务以及以何种顺序运行。
3. Execution：运行在配置阶段结束时选择的任务。

这些阶段构成了 Gradle 的构建生命周期。



Gradle 提供了多种机制来配置 Gradle 本身和特定项目：

* 命令行参数，这种方式的优先级最高
* 系统属性：例如gradle.properties文件中配置的systemProp开头的属性
* gradle属性：项目root目录或GRADLE_USER_HOME环境变量配置目录的gradle.properties文件
* 环境变量：环境变量GRADLE_OPTS

除了配置构建环境之外，您还可以使用诸如 -PreleaseType=final 之类的项目属性来配置给定的项目构建。

## gradle属性

Gradle 提供了多个选项，可以轻松配置将用于执行构建的 Java 进程。 虽然可以通过 GRADLE_OPTS 或 JAVA_OPTS 在本地环境中配置这些，但能够在版本控制中存储某些设置（如 JVM 内存配置和 Java 主目录位置）非常有用，以便整个团队可以在一致的环境中工作。 为此，请将这些设置放入提交给版本控制系统的 gradle.properties 文件中。

gradle根据命令行参数和`gradle.properties`文件的属性启动jvm进程。如果某个属性配置在多个文件中，先发现的生效：

* 命令行参数，使用 -P或--project-prop 选项设置。
* GRADLE_USER_HOME 目录下的gradle.properties
* root 目录下的gradle.properties
* gradle安装目录下gradle.properties

> GRADLE_USER_HOME 的位置可通过命令行上传递的 `-Dgradle.user.home `系统属性更改。

示例：理解属性类型

**gradle.properties**

```properties
gradlePropertiesProp=gradlePropertiesValue
sysProp=shouldBeOverWrittenBySysProp
systemProp.system=systemValue
```

**build.gradle**

```groovy
tasks.register('printProps') {
    doLast {
        println commandLineProjectProp
        println gradlePropertiesProp
        println systemProjectProp
        println System.properties['system']
    }
}
```

结果执行：

```sh
$ gradle -q -PcommandLineProjectProp=commandLineProjectPropValue -Dorg.gradle.project.systemProjectProp=systemPropertyValue printProps
commandLineProjectPropValue
gradlePropertiesValue
systemPropertyValue
systemValue
```

以下属性可用于配置 Gradle 构建环境：

| 属性                                                       | 说明                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| **org.gradle.caching=(true,false)**                        | 当设置为 true 时，Gradle 将在可能的情况下重用任何先前构建的任务输出，从而使构建速度更快。 |
| **org.gradle.caching.debug=(true,false)**                  | 设置为 true 时，每个任务的单个输入属性哈希值和构建缓存键都会记录在控制台上。 |
| **org.gradle.configureondemand=(true,false)**              | 按需启用配置，其中 Gradle 将尝试仅配置必要的项目。           |
| **org.gradle.console=(auto,plain,rich,verbose)**           | 自定义控制台输出颜色或详细程度。 默认值取决于 Gradle 的调用方式。 |
| **org.gradle.daemon=(true,false)**                         | 当设置为 true 时，Gradle 守护程序用于运行构建。 默认为true。 |
| **org.gradle.daemon.idletimeout=(# of idle millis)**       | Gradle 守护进程将在指定的空闲毫秒数后自行终止。 默认值为 10800000（3 小时）。 |
| **org.gradle.debug=(true,false)**                          | 当设置为 true 时，Gradle 将在启用远程调试的情况下运行构建，侦听端口 5005。请注意，这相当于将 -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 添加到 JVM 命令行并将挂起虚拟机，直到连接调试器。 默认为false。 |
| **org.gradle.java.home=(path to JDK home)**                | 为 Gradle 构建过程指定 Java 主目录。 该值可以设置为 jdk 或 jre 位置，但是，根据您的构建功能，使用 JDK 更安全。 如果未指定设置，则合理的默认值来自您的环境（JAVA_HOME 或 java 的路径）。 这不会影响用于启动 Gradle 客户端 VM 的 Java 版本。 |
| **org.gradle.jvmargs=(JVM arguments)**                     | 指定用于 Gradle 守护程序的 JVM 参数。 该设置对于为构建性能配置 JVM 内存设置特别有用。 这不会影响 Gradle 客户端 VM 的 JVM 设置。 |
| org.gradle.logging.level=(quiet,warn,lifecycle,info,debug) | 当设置为 quiet、warn、lifecycle、info 或 debug 时，Gradle 将使用此日志级别。 这些值不区分大小写。 lifecycle级别是默认值。 请参阅选择日志级别。 |
| org.gradle.parallel=(true,false)                           | 配置后，Gradle 将fork 出 `org.gradle.workers.max` 个 JVM 以并行执行项目。 |
| org.gradle.priority=(low,normal)                           | 指定 Gradle 守护进程及其启动的所有进程的调度优先级。 默认为normal。 |
| org.gradle.vfs.verbose=(true,false)                        | 在查看文件系统时配置详细日志记录。 默认false。               |
| org.gradle.vfs.watch=(true,false)                          | 切换监视文件系统。 启用 Gradle 后，它会重新使用它在构建之间收集的有关文件系统的信息。 在 Gradle 支持此功能的操作系统上默认启用。 |
| org.gradle.warning.mode=(all,fail,summary,none)            | 当设置为 all、summary 或 none 时，Gradle 将使用不同的警告类型显示。 有关详细信息，请参阅命令行日志记录选项。 |
| org.gradle.workers.max=(max # of worker processes)         | 配置后，Gradle 将使用最多给定数量的worker。 默认为 CPU 处理器数。 |

## 系统属性

使用 -D 命令行选项，您可以将系统属性传递给运行 Gradle 的 JVM。 gradle 命令的 -D 选项与 java 命令的 -D 选项具有相同的效果。

还可以在gradle.properties 文件中设置系统属性，属性需要增加systemProp前缀。

> 必须是root项目下gradle.properties文件中的系统属性才起作用。

## 环境变量

以下环境变量可用于 gradle 命令。 请注意，命令行选项和系统属性优先于环境变量。

* **GRADLE_OPTS**：指定启动 Gradle 客户端 VM 时要使用的 JVM 参数。 客户端 VM 仅处理命令行输入/输出，因此很少需要更改其 VM 选项。 实际构建由 Gradle 守护程序运行，不受此环境变量的影响。
* **GRADLE_USER_HOME**：
* **JAVA_HOME**：

## 项目属性

您可以通过 -P 命令行选项将属性直接添加到您的项目对象。

Gradle 还可以在看到特殊命名的系统属性或环境变量时设置项目属性。 如果环境变量名称看起来像 ORG_GRADLE_PROJECT_prop=somevalue，那么 Gradle 将在您的项目对象上设置一个 prop 属性，其值为 somevalue。 Gradle 也支持系统属性，但使用不同的命名模式，类似于 org.gradle.project.prop。 以下两项都会将您的 Project 对象上的 foo 属性设置为“bar”。

## 配置http代理

配置 HTTP 或 HTTPS 代理（例如用于下载依赖项）是通过标准 JVM 系统属性完成的。 这些属性可以直接在构建脚本中设置； 例如，设置 HTTP 代理主机将使用 `System.setProperty('http.proxyHost', 'www.somehost.org')` 完成。 或者，可以在 gradle.properties 中指定属性。

**gradle.properties**

```properties
systemProp.http.proxyHost=www.somehost.org
systemProp.http.proxyPort=8080
systemProp.http.proxyUser=userid
systemProp.http.proxyPassword=password
systemProp.http.nonProxyHosts=*.nonproxyrepos.com|localhost

systemProp.https.proxyHost=www.somehost.org
systemProp.https.proxyPort=8080
systemProp.https.proxyUser=userid
systemProp.https.proxyPassword=password
systemProp.http.nonProxyHosts=*.nonproxyrepos.com|localhost
```

# 初始化脚本

初始化脚本类似于 Gradle 中的其他脚本。 但是，这些脚本在构建开始之前运行。 可以做些前置工作，例如，配置在哪里查找自定义插件。

**init脚本的位置**

* 在命令行上指定一个文件。 命令行选项是 -I 或 --init-script ，后面跟脚本的路径。 命令行选项可以出现多次，是追加而不是覆盖。 如果命令行上指定的任何文件不存在，构建将失败。
* 将名为 init.gradle的文件放在 USER_HOME/.gradle/ 目录中。
* 将一个以 .gradle结尾的文件放在 USER_HOME/.gradle/init.d/ 目录中。
* GRADLE_HOME/init.d/ 目录中放置一个以 .gradle结尾的文件。 

如果找到多个 init 脚本，它们将按照上面指定的顺序全部执行。 

示例： 配置默认存储库

**init.gradle**

```groovy
allprojects {
    repositories {
      maven {
      	url 'https://maven.aliyun.com/repository/public'
      }
    }
}
```

以后idea创建的项目都会使用这个仓库了，再也不用创建个项目就修改成阿里的仓库了。

# 构建脚本

项目： project

任务： task

每个 Gradle 构建都由一个或多个项目组成。 一个项目代表什么取决于你用 Gradle 做什么。 例如，一个项目可能代表一个库 JAR 或一个 Web 应用程序。

Gradle 可以在项目上执行的工作由一项或多项任务定义。 任务代表构建执行的一些原子工作。 这可能是编译一些类、创建 JAR、生成 Javadoc。

通常，任务是通过应用插件提供的，因此您不必自己定义它们。 可以理解插件定义了多个任务，以及任务的依赖关系。

## ext属性



除了groovy方式定义变量之外，还可以使用ext属性定义，这些属性可以被子项目读取到。

```
ext {
    springVersion = "3.1.0.RELEASE"
    emailNotification = "build@master.org"
}
```

## 插件

* 二进制插件：实现Plugin接口或以DSL 语言声明方式编写。二进制插件可以驻留在构建脚本中、项目层次结构中或外部插件 jar 中。 

* 脚本插件：脚本插件是额外的构建脚本，可进一步配置构建并通常实施声明性方法来操作构建。 它们通常在构建中使用，尽管它们可以被外部化并从远程位置访问。

插件通常从脚本插件开始（因为它们易于编写），然后随着代码变得更有价值，它会迁移到二进制插件，可以在多个项目或组织之间轻松测试和共享。



要使用封装在插件中的构建逻辑，Gradle 需要执行两个步骤。 首先，它需要解析插件，然后它需要将插件应用到目标，通常是一个项目。

解析插件意味着找到包含给定插件的 jar 的正确版本并将其添加到脚本类路径中。 一旦插件被解析，它的 API 就可以在构建脚本中使用。 脚本插件是自解析的，因为它们是从应用它们时提供的特定文件路径或 URL 解析的。 作为 Gradle 发行版的一部分提供的核心二进制插件会自动解析。

### 二进制插件使用

#### DSL方式

```groovy
plugins {
    id 'com.jfrog.bintray' version '1.8.5'
}
```

这种方式会从[插件仓库](https://plugins.gradle.org/)或私有仓库下载。gradle自带插件可以使用简短方式：

```groovy
plugins {
    id 'java'
}
```



**示例：多项目配置插件**

settings.gradle

```groovy
include 'hello-a'
include 'hello-b'
include 'goodbye-c'
```

build.gradle

```groovy
plugins {
    id 'com.example.hello' version '1.0.0' apply false
    id 'com.example.goodbye' version '1.0.0' apply false
}
```

hello-a/build.gradle

```groovy
plugins {
    id 'com.example.hello'
}
```

hello-b/build.gradle

```groovy
plugins {
    id 'com.example.hello'
}
```

goodbye-c/build.gradle

```groovy
plugins {
    id 'com.example.goodbye'
}
```



**示例：*buildSrc* 方式**

buildSrc/build.gradle

```groovy
plugins {
    id 'java-gradle-plugin'
}

gradlePlugin {
    plugins {
        myPlugins {
            id = 'my-plugin'
            implementationClass = 'my.MyPlugin'
        }
    }
}
```

build.gradle

```groovy
plugins {
    id 'my-plugin'
}
```

示例：**指定私人插件仓库方式一**

settings.gradle

```groovy
pluginManagement {
    repositories {
        maven {
            url './maven-repo'
        }
        gradlePluginPortal()
        ivy {
            url './ivy-repo'
        }
    }
}
```

示例：**指定私人插件仓库方式二**

init.gradle

```groovy
settingsEvaluated { settings ->
    settings.pluginManagement {
        plugins {
        }
        resolutionStrategy {
        }
        repositories {
        }
    }
}
```

#### buildscript 方式

通过将插件添加到构建脚本类路径然后应用插件，可以将已发布为外部 jar 文件的二进制插件添加到项目中。 

```groovy
buildscript {
    repositories {
        gradlePluginPortal()
    }
    dependencies {
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.8.5'
    }
}

apply plugin: 'com.jfrog.bintray'
```

### 脚本插件使用

```groovy
apply from: 'other.gradle'
```

脚本插件是自动解析的，可以从本地文件系统或远程位置加载。 文件系统位置相对于项目目录，而远程脚本位置使用 HTTP URL 指定。