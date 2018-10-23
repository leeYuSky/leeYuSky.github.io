---
title: [ClassLoader] 深入理解
data: 2018-10-23
categories:
- 类加载
tags:
- 类加载
- ClassLoader
---

## [ClassLoader] 深入理解

### 类加载过程和类加载器

#### 类加载过程

每个编写的`.java`拓展名类文件都存储着需要执行的程序逻辑，这些`.java`文件经过`Java`编译器编译成拓展名为`.class`的文件，`.class`文件中保存着`Java`代码经转换后的虚拟机指令，当需要使用某个类时，虚拟机将会加载它的`.class`文件，并创建对应的`class`对象，将`class`文件加载到虚拟机的内存，这个过程称为类加载。类加载的过程如下：

![](https://leeyusky.github.io/assets/images/类加载过程.png)

* **加载**：类加载过程的一个阶段：通过一个类的完全限定查找此类字节码文件，并利用字节码文件创建一个 Class 对象。
* **验证**：目的在于确保 Class 文件的字节流中包含信息符合当前虚拟机要求，不会危害虚拟机自身安全。主要包括四种验证，文件格式验证，元数据验证，字节码验证，符号引用验证。
* **准备**：为类变量(即static修饰的字段变量)分配内存并且设置该类变量的初始值即0(如`static int i=5;`这里只将 i 初始化为0，至于5的值将在初始化时赋值)，这里不包含用`final`修饰的`static`，因为`final`在编译的时候就会分配了，注意这里不会为实例变量分配初始化，类变量会分配在方法区中，而实例变量是会随着对象一起分配到 Java 堆中。
* **解析**：主要将常量池中的符号引用替换为直接引用的过程。符号引用就是一组符号来描述目标，可以是任何字面量，而直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。有类或接口的解析，字段解析，类方法解析，接口方法解析(这里涉及到字节码变量的引用，如需更详细了解，可参考《深入Java虚拟机》)。
* **初始化**：类加载最后阶段，若该类具有超类，则对其进行初始化，执行静态初始化器和静态初始化成员变量(如前面只初始化了默认值的`static`变量将会在这个阶段赋值，成员变量也将被初始化)。

这便是类加载的5个过程，而类加载器的任务是根据一个类的全限定名来读取此类的二进制字节流到JVM中，然后转换为一个与目标类对应的`java.lang.Class`对象实例，在虚拟机提供了3种类加载器，引导（Bootstrap）类加载器、扩展（Extension）类加载器、系统（System）类加载器（也称应用类加载器）。

#### 引导（Bootstrap）类加载器

启动类加载器最顶层的加载类，主要加载的是JVM自身需要的类，这个类加载使用 C++ 语言实现的，是虚拟机自身的一部分，它负责将 `<JAVA_HOME>/lib`路径下的核心类库或`-Xbootclasspath`参数指定的路径下的 jar 包加载到内存中，注意必由于虚拟机是按照文件名识别加载 jar 包的，如`rt.jar`，如果文件名不被虚拟机识别，即使把 jar 包丢到 lib 目录下也是没有作用的(出于安全考虑，Bootstrap启动类加载器只加载包名为 java、javax、sun 等开头的类)。


#### 扩展（Extension）类加载器

扩展类加载器是指 Sun 公司(已被 Oracle 收购)实现的`sun.misc.Launcher$ExtClassLoader`类，由 Java 语言实现的，是`Launcher`的静态内部类，它负责加载`<JAVA_HOME>/lib/ext`目录下或者由系统变量`-Djava.ext.dir`指定位路径中的类库，开发者可以直接使用标准扩展类加载器。


#### 系统（System）类加载器

也称应用程序加载器是指 Sun 公司实现的`sun.misc.Launcher$AppClassLoader`。它负责加载系统类路径`java -classpath`或`-D java.class.path`指定路径下的类库，也就是我们经常用到的`classpath`路径，开发者可以直接使用系统类加载器，一般情况下该类加载是程序中默认的类加载器，通过`ClassLoader#getSystemClassLoader()`方法可以获取到该类加载器。 

#### Launcher 类

`sun.misc.Launcher`,它是一个 java 虚拟机的入口应用，该类里声明了`sun.misc.Launcher$ExtClassLoader`和`sun.misc.Launcher$AppClassLoader`。

```java
public class Launcher {
    private static URLStreamHandlerFactory factory = new Launcher.Factory();
    private static Launcher launcher = new Launcher();
    private static String bootClassPath = System.getProperty("sun.boot.class.path");
    private ClassLoader loader;
    private static URLStreamHandler fileHandler;

    public static Launcher getLauncher() {
        return launcher;
    }

    public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }

        Thread.currentThread().setContextClassLoader(this.loader);

        ...省略其他
}
```

> 从`Launcher`类的构造函数中可以看到，其首先初始化了扩展类加载器，接着初始化了系统类加载器（将扩展类加载器作为了其 parent），之后将系统加载器设置为线程上下文类类加载器。

### 双亲委托机制

#### 双亲委托机制概念

双亲委派模式是在 Java 1.2 后引入的，其工作原理的是，如果一个类加载器收到了类加载请求，它并不会自己先去加载，而是把这个请求委托给父类的加载器去执行，如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归，请求最终将到达顶层的启动类加载器，如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，这就是双亲委派模式。

![](https://leeyusky.github.io/assets/images/双亲委托机制.png)

采用双亲委派模式的是好处是 Java 类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子 ClassLoader 再加载一次。其次是考虑到安全因素，java 核心 api 中定义类型不会被随意替换，假设通过网络传递一个名为`java.lang.Integer`的类，通过双亲委托模式传递到启动类加载器，而启动类加载器在核心`Java API`发现这个名字的类，发现该类已被加载，并不会重新加载网络传递的过来的`java.lang.Integer`，而直接返回已加载过的`Integer.class`，这样便可以防止核心 API 库被随意篡改。可能你会想，如果我们在`classpath`路径下自定义一个名为`java.lang.SingleInterge`类(该类是胡编的)呢？该类并不存在`java.lang`中，经过双亲委托模式，传递到启动类加载器中，由于父类加载器路径下并没有该类，所以不会加载，将反向委托给子类加载器加载，最终会通过系统类加载器加载该类。但是这样做是不允许，因为`java.lang`是核心 API 包，需要访问权限，强制加载将会报出如下异常：

```java
Exception in thread "main" java.lang.SecurityException: Prohibited package name: java.lang
```

#### ClassLoader 源码

`ClassLoader`是一个抽象类，其后所有的类加载器都继承自ClassLoader（不包括引导类加载器）。

![](https://leeyusky.github.io/assets/images/classloader依赖关系.png)

##### loadClass(String)

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

> 该方法描述的步骤为：
> 
> 1. 执行`findLoadedClass(String)`去检测这个 class 是不是已经加载过了。 
> 2. 执行父加载器的`loadClass(String)`方法。如果父加载器为 null，则 jvm 内置的加载器去替代，也就是 Bootstrap ClassLoader。
> 3. 如果向上委托父加载器没有加载成功，则通过`findClass(String)`查找。
> 4. 如果 class 在上面的步骤中找到了，参数 resolve 又是 true 的话，那么`loadClass()`又会调用`resolveClass(Class)`这个方法来生成最终的 Class 对象 (该方法的作用是：链接指定的类。这个方法给 Classloader 用来链接一个类，如果这个类已经被链接过了，那么这个方法只做一个简单的返回。否则，这个类将被按照  Java™ 规范中的 Execution 描述进行链接)。

##### findClass(String)

在JDK1.2之前，在自定义类加载时，总会去继承 ClassLoader 类并重写`loadClass(String)`方法，从而实现自定义的类加载类，但是在JDK1.2之后已不再建议用户去覆盖`loadClass(String)`方法，而是建议把自定义的类加载逻辑写在`findClass(String)`方法中，从前面的分析可知，`findClass(String)`方法是在`loadClass(String)`方法中被调用的，当`loadClass(String)`方法中父加载器加载失败后，则会调用自己的`findClass(String)`方法来完成类加载，这样就可以保证自定义的类加载器也符合双亲委托模式。需要注意的是 ClassLoader 类中并没有实现 `findClass(String)` 方法的具体代码逻辑，取而代之的是抛出 ClassNotFoundException 异常，同时应该知道的是 `findClass(String)` 方法通常是和 `defineClass()` 方法一起使用的(稍后会分析)，ClassLoader 类中 `findClass(String)` 方法源码如下：

```
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```

> **注**：如果要编写一个classLoader的子类，也就是自定义一个classloader，建议覆盖 findClass() 方法，而不要直接改写 loadClass() 方法。

##### defineClass(byte[] b, int off, int len) 

`defineClass()`方法是用来将 byte 字节流解析成 JVM 能够识别的 Class 对象 (ClassLoader 中已实现该方法逻辑)，通过这个方法不仅能够通过 class 文件实例化 class 对象，也可以通过其他方式实例化 class 对象，如通过网络接收一个类的字节码，然后转换为 byte 字节流创建对应的 Class 对象，`defineClass()`方法通常与`findClass()`方法一起使用，一般情况下，在自定义类加载器时，会直接覆盖 ClassLoader 的`findClass()`方法并编写加载规则，取得要加载类的字节码后转换成流，然后调用`defineClass()`方法生成类的 Class 对象。

##### resolveClass(Class≺?≻ c)

使用该方法可以使用类的 Class 对象创建完成也同时被解析。前面我们说链接阶段主要是对字节码进行验证，为类变量分配内存并设置初始值同时将字节码文件中的符号引用转换为直接引用。

### 线程上下文类加载器(TCCL)

在 Java 应用中存在着很多服务提供者接口（Service Provider Interface，SPI），这些接口允许第三方为它们提供实现，如常见的 SPI 有 JDBC、JNDI 等，这些 SPI 的接口属于 Java 核心库，一般存在 rt.jar 包中，由 Bootstrap 类加载器加载，而 SPI 的第三方实现代码则是作为Java应用所依赖的 jar 包被存放在 classpath 路径下，由于 SPI 接口中的代码经常需要加载具体的第三方实现类并调用其相关方法，但 SPI 的核心接口类是由引导类加载器来加载的，而 Bootstrap 类加载器无法直接加载 SPI 的实现类，同时由于双亲委派模式的存在，Bootstrap 类加载器也无法反向委托 AppClassLoader 加载器 SPI 的实现类。在这种情况下，我们就需要一种特殊的类加载器来加载第三方的类库，而线程上下文类加载器就是很好的选择。 

线程上下文类加载器（contextClassLoader）是从 JDK 1.2 开始引入的，我们可以通过`java.lang.Thread`类中的g`etContextClassLoader()`和`setContextClassLoader(ClassLoader cl)`方法来获取和设置线程的上下文类加载器。如果没有手动设置上下文类加载器，线程将继承其父线程的上下文类加载器，初始线程的上下文类加载器是系统类加载器（AppClassLoader）,在线程中运行的代码可以通过此类加载器来加载类和资源

关于 TCCL 的分析可以参考一下文章：

[真正理解线程上下文类加载器（多案例分析)](https://blog.csdn.net/yangcheng33/article/details/52631940)

> 对于TCCL的解释多以 SPI 中 JDBC 的实现类加载为案例。但是由于我目的的实践经验不足，依然不能很好的理解。

### 自定义类加载器

> 自己编写的类加载器一般遵循以下步骤：
> 
> * 重写`findClass()`方法（最好不要重写`loadClass()`方法，因为有可能会破坏双亲委托机制）
> * 获得 class 文件的字节流
> * 使用`defineClass()`方法获得 class 对象

> 可参考 [深入理解Java类加载器(ClassLoader)](https://blog.csdn.net/javazejian/article/details/73413292#类与类加载器-1) 中 “编写自己的类加载器” 一节。


### 常用方法

#### 显示类加载方法

##### Class::forName()

> `Class.forname()`:是一个静态方法,最常用的是`Class.forname(String className);`根据传入的类的全限定名返回一个 Class 对象.该方法在将 Class 文件加载到内存的同时,会执行类的初始化.

> 如: `Class.forName("com.wang.HelloWorld");`

##### ClassLoader#loadClass()

> `ClassLoader.loadClass()`:这是一个实例方法,需要一个 ClassLoader 对象来调用该方法,该方法将 Class 文件加载到内存时,并不会执行类的初始化,直到这个类第一次使用时才进行初始化.该方法因为需要得到一个 ClassLoader 对象,所以可以根据需要指定使用哪个类加载器.

> 如:`ClassLoader cl=.......;cl.loadClass("com.wang.HelloWorld");`

#### 资源加载方法

##### ClassLoader::getSystemResource()

```java
public static URL getSystemResource(String name) {
    ClassLoader system = getSystemClassLoader();
    if (system == null) {
        return getBootstrapResource(name);
    }
    return system.getResource(name);
}
```

> * 该方法首先获得系统类加载器，若系统类加载器不存在，则用引导类加载器加载资源。
> * 调用`ClassLoader#getResource()`方法，具体过程参考下面的讲解。

##### Class<T>#getResource()

```java
public java.net.URL getResource(String name) {
    name = resolveName(name);
    ClassLoader cl = getClassLoader0();
    if (cl==null) {
        // A system class.
        return ClassLoader.getSystemResource(name);
    }
    return cl.getResource(name);
}

private String resolveName(String name) {
    if (name == null) {
        return name;
    }
    if (!name.startsWith("/")) {
        Class<?> c = this;
        while (c.isArray()) {
            c = c.getComponentType();
        }
        String baseName = c.getName();
        int index = baseName.lastIndexOf('.');
        if (index != -1) {
            name = baseName.substring(0, index).replace('.', '/')
                +"/"+name;
        }
    } else {
        name = name.substring(1);
    }
    return name;
}
```


> * 首先通过`resolveName()`方法将输入的资源名称进行解析
> 	* 如果资源名称是非绝对路径，则将该资源名称前面加上该类所在的包名，如：类为`edu.tju.scs.HelloWorld`，资源为`log4j.properties`，则得到资源路径为`edu/tju/scs/log4j.properties`
> 	* 如果资源名称是绝对路径，则将前缀 / 去掉
> * 之后获得该类的类加载器。若不存在，则调`ClassLoader::getSystemResource()`
> * 调用`ClassLoader#getResource()`方法，具体过程参考下面的讲解。

> **注**：因此如果资源文件和该类处于同一目录下，可以使用该方法


##### ClassLoader#getResource()

```java
public URL getResource(String name) {
    URL url;
    if (parent != null) {
        url = parent.getResource(name);
    } else {
        url = getBootstrapResource(name);
    }
    if (url == null) {
        url = findResource(name);
    }
    return url;
}

protected URL findResource(String name) {
    return null;
}
```

> 可以看到，在 ClassLoader 类中，`getResource()`方法步骤如下：
> 
> * 首先调用父加载器的`getResource()`方法，若不存在父加载器，则用引导类加载器加载资源
> * 若上述步骤均失败，则调用`findResource()`方法寻找资源

> 在 ClassLoader 中，`findResource()`方法直接返回 null。但是在其子类`URLClassLoader`中重写了`findResource()`方法。

> **注**：因此该方法将沿着系统类加载器向上搜索所有的路径（需要自己的资源位于项目根目录）

##### 总结

> * `ClassLoader::getSystemResource()` uses the system classloader. This uses the classpath that was used to start the program. If you are in a web container such as tomcat, this will NOT pick up resources from your WAR file.
> * `Class<T>#getResource()` prepends the package name of the class to the resource name, and then delegates to its classloader. If your resources are stored in a package hierarchy that mirrors your classes, use this method.
> * `ClassLoader#getResource()` delegates to its parent classloader. This will eventually search for the resource all the way upto the system classloader.
> 
> 参考： [Class.getResource and ClassLoader.getSystemResource: is there a reason to prefer one to another?](https://stackoverflow.com/questions/6298400/class-getresource-and-classloader-getsystemresource-is-there-a-reason-to-prefer)



### 参考文章

* [深入理解Java类加载器(ClassLoader)](https://blog.csdn.net/javazejian/article/details/73413292#类与类加载器-1)
* [一看你就懂，超详细java中的ClassLoader详解](https://blog.csdn.net/briblue/article/details/54973413)
* [深度分析Java的ClassLoader机制（源码级别](http://www.hollischuang.com/archives/199)
* [真正理解线程上下文类加载器（多案例分析)](https://blog.csdn.net/yangcheng33/article/details/52631940)




