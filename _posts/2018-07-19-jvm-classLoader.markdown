---
layout:     post
title:      "jvm类加载机制"
subtitle:   " jvm classLoader "
date:       2018-07-19 13:15:00
author:     "Aliyang"
header-img: "img/post-bg-jvm-classloader.jpg"
tags:
    - JVM
---
## 类加载的时机
![生命周期](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/load_class.png?raw=true)
类初始化情况：
1. 遇到**new ,getstatic,putstatic或invokestatic**四条字节码指令时，若类没有初始化，那么进行初始化。
2. 类反射调用。
3. 初始化子类发现父类未初始化，那么初始化父类。
4. 虚拟机启动时初始化包含main()的主类。
5. 如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

## 类加载过程
#### 加载
需要完成3件事情：
1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。
数组类和非数组类加载：
1.非数组类加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成，开发人员可以通过定义自己的类加载器去控制字节流的获取方式。
2.数组类本身不通过类加载器创建，它是由Java虚拟机直接创建的，但是数组类的元素类型(去掉所有维度后的类型)最终靠类加载器创建。
	* 若最终类型是引用类，使用本节定义的加载过程加载该组件，数组C将在加载该组件类型的类加载器的类名称空间上被标识。
	* 若不是引用类型，Java虚拟机将会把数组C标记为与引导类加载器关联。

#### 验证
目的：确保Class文件字节流中包含的信息符合当前虚拟机的要求。
1. 文件格式验证：验证字节流是否符合Class文件格式的规范，并且能被当前版本虚拟机处理。
2. 元数据验证：对字节码描述的信息进行语义分析。
3. 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑。
4. 符号引用验证：对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验，如全限定名是否能找到对应的类等；发生在虚拟机将符号引用转化为直接引用的时候，在解析阶段中发生。

#### 准备
目的：为**类变量**在方法区中分配内存并设置变量初始值。
初始值指的是默认的值，即0，false等，但是如果类变量有ConstantValue属性(final)修饰，那么就初始化为对应的值。

#### 解析
目的：虚拟机将常量池内的符号引用替换为直接引用的过程。
> 符号引用：以一组符号来描述所引用的目标，符号引用可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。与虚拟机的内存布局无关。

1. 类或接口的解析
若当前所处类D，把未解析过的符号引用N解析为一个类或接口C的直接引用步骤：
	* 若C非数组类型，会把N的全限定名传递给D的类加载器加载类C，会引发一些列的新类的递归加载。
	* 若C是数组类型，数组元素类型为对象，按上边方式加载数组元素类型，接着虚拟机生成一个代表数组维度和元素的数组对象。
	* 上述步骤完成后无异常，那么C已经是一个有效的类或接口，但是还要进行符号引用验证，确定D具备对C的访问权限，不具备将抛出IllegalAccessError异常。

2. 字段解析
首先对字段表内class_index项中索引的CONSTANT_Class_info符号引用解析，也就是字段所属的类或接口的符号引用。然后解析符号引用，若出现异常则失败；否则，用C表示该字段所属的类或接口，接下来，解析C中该字段的符号引用。
	* 如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
	* 否则，如果在C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
	* 否则，如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
	* 否则，查找失败，抛出java.lang.NoSuchFieldError异常。
	* 最后需要对字段权限验证，如果不具备访问权限，抛出java.lang.IllegalAccessError异常。

3. 类方法解析
	* 第一步类似字段解析第一步。
	* 类方法和接口方法符号引用的常量类型定义是分开的，如果在类方法表中发现class_index中索引的C是个接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。
	* 如果通过了上边这步，在类C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
	* 否则，在类C的父类中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
	* 否则，在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类，这时查找结束，抛出java.lang.AbstractMethodError异常。
	* 否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError。
	* 如果查找过程成功返回了直接引用，将会对这个方法进行权限验证，如果发现不具备对此方法的访问权限，将抛出java.lang.IllegalAccessError异常。

4. 接口方法解析
	* 第一步也是差class_index项中索引，不多说。
	* 如果在接口方法表中发现class_index中的索引C是个类而不是接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。
	* 否则，在接口C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
	* 否则，在接口C的父接口中递归查找，直到java.lang.Object类（查找范围会包括Object类）为止，看是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
	* 否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError异常。
> 因为接口方法的所有方法默认都是public，所以不会抛出java.lang.IllegalAccessError异常。

#### 初始化
初始化阶段是执行类构造器**`<clinit>()`**过程。

1. `<clinit>()`由编译器自动收集类中的所有类变量的赋值动作和静态语句块（`static{}`块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，**定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问**。
2. `<clinit>()`方法与类的构造函数（或者说实例构造器`<init>()`方法）不同，它不需要显式地调用父类构造器，虚拟机会保证在子类的`<clinit>()`方法执行之前，父类的`<clinit>()`方法已经执行完毕。因此在虚拟机中第一个被执行的`<clinit>()`方法的类肯定是java.lang.Object。
3. `<clinit>()`对于类或接口来说并不是必须的，若没有静态语句块和对变量的赋值操作，那么不用生成。
4. 接口中生成的`<clinit>()`只有在父类的定义的变量被使用的时候才会去初始化父类接口。

## 类加载器
把类加载阶段中的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“类加载器”。
#### 类与类加载器
比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义。

#### 双亲委派模型
1. 类加载器分类，从开发人员角度：
	* 启动类加载器(Bootstrap ClassLoader)：加载`<JAVA_HOME>\lib`下包，无法被程序员直接引用。
	* 扩展类加载器(Extension ClassLoader)：加载`<JAVA_HOME>\lib\ext`中的包，或被java.ext.dirs系统变量所指定的路径中的所有类库。
	* 应用程序类加载器(Application ClassLoader)：这个类加载器是ClassLoader中的getSystemClassLoader（）方法的返回值，所以一般也称它为系统类加载器。负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器。

2. 双亲委派模型
指类加载器之间的层次关系，双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器，使用组合关系来复用父加载器的代码。
工作过程：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。
![类加载器](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/ClassLoader.png?raw=true)

## Tomcat类加载器
#### 主要的类加载器
借用引文博客的图片一用。
![tomcat_classloader](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/tomcat_classloader.jpg?raw=true)
tomcat的类加载器和Java虚拟机的类加载器有些不同，tomcat的类加载器主要有如下几个：

* Bootstrap引导类加载器：加载JVM运行的类，即jre/lib/root.rar以及标准扩展类jre/lib/ext中的类包。
* System系统类加载器：加载tomcat启动的类，如CATALINA_HOME/bin下的bootstrap.jar。
* Common通用类加载器：加载tomcat使用以及应用通用的一些类，在CATALINA_HOME/lib下，如servlet-api.jar
* webapp应用类加载器：每个应用部署后会有一个唯一的类加载器，加载位于WEB-INF/lib下的以及WEB-INF/classes下的class文件

#### 类加载顺序
* bootstrap类加载器加载
* system类加载器加载
* 用webapp应用类加载器先加载WEB-INF/classes中类，然后是WEB-INF/lib中的类
* 使用common类加载CATALINA_HOME/lib中的类

## OSGI类加载器
(Open Service Gateway Initiative)OSGI技术是Java动态化模块化系统的一系列规范。OSGI真正意义上实现了解耦，以往的类似SSH的框架，虽然说逻辑上达到了解耦，但是实际物理上并没有解耦，当一个完整的项目上线后，我们去掉其中的一个模块，那么项目是没法运行了。而OSGI即使去掉了一个模块，也能继续运行。
#### OSGI类加载和Java类加载的区别
Java类加载器是分层次的结构，而OSGI是一种网状的结构。OSGI将所有类分成很多个Bundle，每个Bundle有一个类加载器，假设A，B两个Bundle里的类调用Bundle C里边的类，那么Bundle A和B会将类加载委托给Bundle C
的类加载器来加载，以此类推。
#### 类加载步骤
* 先检查要加载的类是否以java.*开头，或者是否在org.osgi.framework.bootdelegation中定义，是的话，那么委托给父类加载器，一般是委托给Application类加载器，因为这些类需要Bootstrap类加载器来加载。
* 检查要加载的类是否在**Import-Package**中声明，是的话，找到对应的包的Bundle的类加载器来加载。
* 检查要加载的类是否在**Require-Bundle**中声明，是的话，将类加载请求委托给对应的Bundle的类加载器加载。
* 检查要加载的类是否是当前类所在jar包的内部的类，是的话调用类加载器
* 检查要加载的类是否在附加在当前bundle上的fragment中的内部类，那么调用对应的fragment的类加载器加载。
* 检查要加载的类是否在Dynamic Import列表的Bundle中，是的话委托给对应的Bundle类加载器加载
* 如果上面的情况都不满足，那么类加载失败


## Reference
* [https://www.cnblogs.com/xing901022/p/4574961.html](https://www.cnblogs.com/xing901022/p/4574961.html)
* [https://blog.csdn.net/vking_wang/article/details/12875619](https://blog.csdn.net/vking_wang/article/details/12875619)