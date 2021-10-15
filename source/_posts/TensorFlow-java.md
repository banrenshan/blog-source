有多种方法可以将 TensorFlow Java 添加到您的项目中。 最简单的方法是添加 `tensorflow-core-platform` 依赖：

```java
<dependency>
  <groupId>org.tensorflow</groupId>
  <artifactId>tensorflow-core-platform</artifactId>
  <version>0.3.3</version>
</dependency>
```

需要注意的是，该依赖不仅包括 TensorFlow Java Core API， 还会导入所有的平台的本地库，这会显着增加项目的大小。

如果您希望针对单个平台，则可以使用 Maven 依赖项排除功能。

以下是支持的本地库：

* tensorflow-core-platform-mkl：支持英特尔® MKL-DNN
* tensorflow-core-platform-gpu：支持 Linux 和 Windows 平台上的 CUDA®
* tensorflow-core-platform-mkl-gpu：支持 Linux 平台上的英特尔® MKL-DNN 和 CUDA®。

此外，可以添加对 tensorflow-framework 库的单独依赖，可以在 JVM 上使用基于 TensorFlow 机器学习的丰富实用程序。

使用 Gradle 从 TensorFlow Java 中排除原生工件并不像使用 Maven 那样容易。 我们建议您使用 Gradle [JavaCPP](https://github.com/bytedeco/gradle-javacpp) 插件来减少依赖项的数量。

下面是样例代码：

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.myorg</groupId>
    <artifactId>hellotensorflow</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <exec.mainClass>HelloTensorFlow</exec.mainClass>
        <!-- Minimal version for compiling TensorFlow Java is JDK 8 -->
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Include TensorFlow (pure CPU only) for all supported platforms -->
        <dependency>
            <groupId>org.tensorflow</groupId>
            <artifactId>tensorflow-core-platform</artifactId>
            <version>0.3.3</version>
        </dependency>
    </dependencies>
</project>
```

```java
import org.tensorflow.ConcreteFunction;
import org.tensorflow.Signature;
import org.tensorflow.Tensor;
import org.tensorflow.TensorFlow;
import org.tensorflow.op.Ops;
import org.tensorflow.op.core.Placeholder;
import org.tensorflow.op.math.Add;
import org.tensorflow.types.TInt32;

public class HelloTensorFlow {

  public static void main(String[] args) throws Exception {
    System.out.println("Hello TensorFlow " + TensorFlow.version());

    try (ConcreteFunction dbl = ConcreteFunction.create(HelloTensorFlow::dbl);
        TInt32 x = TInt32.scalarOf(10);
        Tensor dblX = dbl.call(x)) {
      System.out.println(x.getInt() + " doubled is " + ((TInt32)dblX).getInt());
    }
  }

  private static Signature dbl(Ops tf) {
    Placeholder<TInt32> x = tf.placeholder(TInt32.class);
    Add<TInt32> dblX = tf.math.add(x, x);
    return Signature.builder().input("x", x).output("dbl", dblX).build();
  }
}
```

